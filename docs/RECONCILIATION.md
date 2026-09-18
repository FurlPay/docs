# Reconciliation and event ingestion

How order state stays true, and what happens when it does not.

## Why this exists

Orders fill after the request that created them has returned. Streams drop
events. A lambda dies between the provider accepting an order and our ledger
writing it down.

None of those produce an error anyone sees. They produce a quiet divergence
between what we believe and what is true, and the only way to find one is to
compare both sides on a schedule.

## The correction that shaped the design

**Alpaca does not offer an HMAC-signed webhook for trade events.** Its trade
updates are a Server-Sent Events stream at `/v2/events/trades`, authenticated
with the same API-key headers as every other call, delivered over a long-lived
connection.

Building `/api/webhooks/alpaca` with a signature check would be inventing a
contract the provider does not have — the endpoint would sit there verifying a
header nobody sends, and it would look like working infrastructure.

So the architecture splits into a transport half and a logic half.

## Three feeds, one pipeline

```
  ┌─ polling (sync.ts) ────────┐
  │                            │
  ├─ SSE worker ───────────────┼──→ ingestTradeEvent()
  │                            │      │
  └─ signed webhook (future) ──┘      ├─ claim (atomic)
                                      ├─ persist raw
                                      ├─ ordering guard
                                      └─ apply via state machine
```

### Polling is the production path

Vercel functions have an execution ceiling measured in seconds. Nothing in this
deployment can hold an SSE socket open for hours, so an SSE consumer would have
to live on infrastructure that does not currently exist.

Polling has no connection to drop, no reconnect cursor to get wrong, and is
self-healing — a missed update is picked up on the next pass. The latency cost
is the poll interval, which for *order state* (as opposed to price ticks) is a
few seconds nobody notices.

When an SSE worker does exist, polling keeps running underneath. They are not
alternatives: **a stream is fast and can drop events; a poll is slow and
cannot.** Every serious order system runs both.

`GET /api/investing/orders` also syncs before it answers — the screen someone is
looking at is the screen that gets refreshed, without waiting for a cron tick.

## Deduplication

Alpaca's event stream is **replayable by design**: you reconnect with `since`
and receive events you have already seen. Applying a `fill` twice books the same
shares twice.

The dedupe key is the provider's own `execution_id`:

```ts
if (payload.execution_id) return `exec:${payload.execution_id}`;
```

This is not a preference. Two partial fills of one order share an order id and
differ **only** in `execution_id` — keyed on the order id, the second partial
would look like a replay of the first and real shares would be silently
dropped.

The claim uses `kvSetNx`, which is atomic, so two workers racing on the same
replayed event have exactly one winner. A get-then-set would let both through.

**A KV failure returns "duplicate", not "proceed."** Skipping an event costs a
delay the reconciler closes; applying one twice costs money.

## Ordering

Two guards, because they catch different things.

**The state machine** (`orderStore.applyProviderOrder`) refuses transitions out
of a terminal state. A late `accepted` arriving after a `fill` cannot walk the
order backwards and re-open a closed position.

**The timestamp guard** catches the subtler case: an event that is simply older
than what we already applied. But it is deliberately narrow —

```ts
function isStrictlyStale(event, record): boolean {
  if (event.filledQty == null) return true;
  if (record.filledQty == null) return false;
  return event.filledQty <= record.filledQty;
}
```

— because a late event reporting **more** filled quantity carries information we
are missing. Dropping it on a timestamp comparison alone would lose a real fill.

Fill quantities only ever move forward.

## Events that are not status changes

`trade_bust` and `trade_correct` **reverse or amend a completed execution**.
They are not order-state transitions, and mapping them to one would quietly
advance an order on the strength of a reversal.

Both map to `unknown` and are listed in `AMENDING_EVENTS`. Correct handling is a
compensating ledger entry plus human review — **not implemented yet**, and
tracked below.

## Unknown orders are the finding, not an error

An event for an order we have never seen means the provider has an order this
deployment did not create. That is the single most important thing
reconciliation can surface.

It is dead-lettered loudly. It is **not** used to manufacture a local order
record — we could not say which user it belonged to, so doing that would erase
the discrepancy *and* invent an attribution.

## Severity is not uniform

| Finding | Severity | Why |
|---|---|---|
| Provider filled, we booked nothing | critical | An unbooked position |
| We think it's finished, provider doesn't | critical | It can still fill, and we stopped watching |
| We have an order the provider doesn't | critical | Either it never landed, or it's on another account |
| We're merely behind the provider | warning | Ordinary lag |
| Fill price differs | warning | Cost basis wrong; ownership right |
| **Any** position quantity difference | critical | No size of ownership discrepancy is informational |

### Tolerances

`QTY_EPSILON = 1e-7` — fractional quantities cross the wire as decimal strings
and round-trip through IEEE-754 doubles. A tenth of a millionth of a share is
far below any economically meaningful amount and far above float noise.

`PRICE_EPSILON = 0.005` — half a cent, below the minimum tick, so a real price
difference cannot hide underneath it.

## Reconciliation fixes nothing

`reconcile.ts` detects and classifies. It does not mutate.

Auto-correcting a financial discrepancy is how a *detection* bug becomes a
*money* bug — a comparator with an off-by-one would happily "repair" a correct
ledger. Remediation is a human decision with an audit trail behind it.

## An incomplete run is never a pass

```ts
export function needsAttention(report: ReconcileReport): boolean {
  return report.summary.critical > 0 || report.incomplete !== null;
}
```

Zero findings from an unreachable provider looks exactly like "everything
matches". That is the most dangerous output this system can produce, so
incompleteness is itself an alarm.

## What is NOT built

| Gap | Consequence |
|---|---|
| **No scheduler** | Nothing runs `syncOpenOrders` on a timer. Sync happens only when a user opens the orders screen. |
| **No alerting sink** | `needsAttention` returns true into a log line. Nothing pages anyone. |
| **No bust/correct handling** | `trade_bust` and `trade_correct` are detected and parked. The compensating ledger entry is manual. |
| **No cash reconciliation** | Positions and orders are compared; cash balances are not. |
| **No dividend/corporate-action reconciliation** | The engine has no comparator for either. |
| **No SSE worker** | Requires infrastructure outside the serverless runtime. |
| **KV, not Postgres** | Order records and events are cross-instance and durable, but not relational, not joinable, and not in the DB backup policy. |

## Operating it

- `GET /api/investing/orders` — a user's orders, synced on read
- `GET /api/ops/readiness` (admin) — can this deployment move real money, and if not, why
- `FURLPAY_MONEY_KILL_SWITCH=1` — disables every money capability at once
