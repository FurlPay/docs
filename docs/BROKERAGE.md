# Brokerage architecture

How an equity order gets from a tap to a market, what is enforced along the
way, and what is still missing.

## What this replaced

`POST /api/investing/order` used to be entirely internal. It minted an order id
with `crypto.randomBytes`, decided the outcome on the spot —

```ts
const status = type === "market" ? "filled" : "pending";
const filledQty = notional / holding.price;
```

— and moved the tax-lot ledger. No brokerage was ever contacted. **Every
"filled" that route returned described an order that had never reached a
market**, at a price taken from the last quote we happened to have cached.

The same shape existed in `GET /api/investing/rwa`, which returned a constant
`1.5 bNVDA / 3.0 bAAPL` to every caller, marked to real live prices and summed
into a real "total equity value".

Both now go through real integrations, and both fail closed when those
integrations are unavailable.

## The layers

```
POST /api/investing/order
        │
        ├─ 1. flags.ts ─────────── is BROKERAGE_LIVE on, with credentials?
        │                           off → 503, never a simulated fill
        │
        ├─ 2. orderStore.ts ────── write `created` BEFORE submitting
        │                           unwritable → 503, refuse to trade
        │
        ├─ 3. brokerage/index.ts ─ select the adapter
        │        ├── AlpacaBrokerageProvider   (live or paper host)
        │        └── SimulatedBrokerageProvider (never in production)
        │
        ├─ 4. placeOrder(clientOrderId) ── provider-side idempotency
        │
        └─ 5. ledger ──────────── ONLY on a provider-reported fill,
                                   at the provider's price
```

### 1. Flags (`lib/flags.ts`)

Four gates, all of which must pass: the flag is on, the environment permits it,
**every credential it names is present**, and the kill switch is not engaged.

Gate three is the one that matters. "Enabled but unconfigured" is the state that
produces a fabricated success — the flag says go, the client has no key, and a
fallback invents a result. Requiring the credentials before the flag can report
`on` makes that state unreachable.

`FURLPAY_ENV` is separate from `NODE_ENV` on purpose: Next sets
`NODE_ENV=production` for preview branches too, so inferring from it would treat
a preview as a place where real money may move.

### 2. The order record (`lib/brokerage/orderStore.ts`)

Written **before** submission. If the record is written only on success, a
request that times out leaves no trace of an order that may be live at the
venue — real to the market, invisible to us.

The state machine is enforced in `applyProviderOrder`, not documented and hoped
for. Webhook delivery is unordered and at-least-once, so a `fill` and an
`accepted` for the same order routinely arrive in the wrong order; without the
transition table the late `accepted` walks a filled order backwards and re-opens
a closed position.

Fill quantities only move forward. A late event reporting a smaller filled
quantity describes an earlier moment.

### 3. Provider selection (`lib/brokerage/index.ts`)

| Condition | Provider |
|---|---|
| `BROKERAGE_LIVE` on + credentials | Alpaca, live host |
| Non-production, paper credentials present | Alpaca, paper host |
| Non-production, no credentials | Simulator |
| **Production, flag off** | **Throws. There is no provider.** |

The last row is the point. In production with the capability off, asking for a
brokerage is a programming error, not a condition to degrade around. Returning a
simulator there means a real user receiving a fabricated fill.

`SimulatedBrokerageProvider` enforces this in its own constructor — it throws if
constructed in the production environment. A guard the caller owns is a guard
somebody eventually forgets to write.

The simulator also refuses to invent prices: a market order fills at the price
from the real quote pipeline, and **rejects** when no quote is available.

### 4. Idempotency

`client_order_id` is Alpaca's idempotency mechanism. A second order with an id
already in use is refused with 422 — and that refusal is the **success** case
for a retry, because it proves the first attempt landed. `placeOrder` catches it
and reads the existing order back. Surfacing it as an error would make a
protected retry look like a failure and invite another attempt.

The client order id is derived from the caller's `Idempotency-Key`, hashed with
the user id, so the same retried request reaches the same provider-side order
even after our own idempotency cache has expired.

### 5. Indeterminate ≠ failed

A timeout means the order **may exist**. `BrokerageIndeterminateError` exists so
callers cannot treat that as "nothing happened":

```
202 Accepted
{ "status": "unknown", "orderId": "fp_…",
  "detail": "…may have been accepted. Check its status before placing another." }
```

Retrying on the assumption that nothing happened is how one tap becomes two
orders.

### 6. The ledger moves only on a real fill

```ts
if ((status === "filled" || status === "partially_filled")
    && filledQty != null && filledQty > 0
    && fillPrice != null && fillPrice > 0) { … }
```

Both numbers must come from the provider. A `filled` status with no quantity is
not enough to move a tax lot. The position is marked at the price actually paid,
not at the quote shown before the trade.

If the ledger write fails **after** a real fill, the response is a 500 carrying
`warning: "The order executed at the broker but could not be recorded locally"` —
not a 400 telling the user their valid order was invalid.

## Reconciliation (`lib/brokerage/reconcile.ts`)

Orders fill after the request that created them returned; webhooks are dropped;
a lambda dies between acceptance and the ledger write. None of those produce an
error anyone sees. They produce a quiet divergence, and the only way to find one
is to compare both sides on a schedule.

**The provider is authoritative for execution.** A mismatch is always reported as
"our record is wrong".

**This module fixes nothing.** Auto-correcting a financial discrepancy is how a
detection bug becomes a money bug — a comparator with an off-by-one would happily
"repair" a correct ledger. Output is a finding, not a mutation.

Severity is not uniform:

| Finding | Severity | Why |
|---|---|---|
| Provider filled, we booked nothing | critical | An unbooked position |
| We think it's finished, provider doesn't | critical | It can still fill, and we stopped watching |
| We're merely behind the provider | warning | Ordinary lag |
| Fill price differs | warning | Cost basis wrong; ownership right |
| Any position quantity difference | critical | No size of ownership discrepancy is informational |

A run that could not complete sets `incomplete` and **never** returns a clean
report. Zero findings from an unreachable provider looks exactly like "everything
matches" — the most dangerous output this system can produce.

## Tokenized equities

`lib/onchain/stockTokenRegistry.ts` ships with **no contract addresses**, and
that is deliberate. A recalled mint address looks exactly like the right answer
and is wrong in a way nobody can see; the consequence is USDC sent to a contract
that is not the asset the user asked for.

An entry becomes tradable only when a person has:

1. opened the issuer's published token list at `sourceUrl`
2. copied the address in
3. read `decimals` off-chain (**not** assumed to be 18 — several issuers use 6 or 8)
4. recorded the issuer's eligibility rules
5. filled in `verifiedBy` / `verifiedAt` and set `status: "active"`

`tradableTokens()` filters on all of it. Eligibility is **default deny**: an
entry with no recorded eligibility list denies everyone.

## On-chain balances (`lib/onchain/balances.ts`)

EVM reads go through Multicall3 at
`0xcA11bde05977b3631167028862bE2a173976CA11` — one `eth_call` for every balance
and every `decimals()`.

Solana queries **both** token programs. Asking only the legacy SPL Token program
returns zero for every Token-2022 mint, which is what xStocks and most
2025-onward tokenized equities are issued as — a portfolio confidently showing no
position in assets the user holds.

Raw amounts are carried as decimal **strings**. A token with 18 decimals
overflows `Number.MAX_SAFE_INTEGER` at about 9 units.

A zero balance and an unreadable chain are different answers and are never
collapsed. An RPC failure yields `{ ok: false, error, detail }`, and the caller
renders "unavailable" — telling someone their position is empty because a node
timed out is a false statement about their money.

## Operating it

`GET /api/ops/readiness` (admin) answers "can this deployment move real money,
and if not, why". Blocking checks vs advisory checks, every flag with its reason,
and the registry audit.

`FURLPAY_MONEY_KILL_SWITCH=1` disables every money capability at once, checked
before any per-flag setting.

## What is NOT built

Stated plainly, because a flag that is off is not the same as a feature that
exists.

| Gap | Status |
|---|---|
| **Brokerage webhooks** | No `/api/webhooks/alpaca` handler. Order state advances only when something reads it back. Alpaca's `/v2/events/trades` SSE stream is the documented path. |
| **Scheduled reconciliation** | The engine exists and is unit-tested; nothing runs it on a timer, and there is no alerting sink. |
| **DeFi collateral** | Flag permanently blocked — the risk engine (LTV, liquidation threshold, health factor) is not built. |
| **Cross-chain stock transfers** | Flag permanently blocked — per-issuer transfer-restriction checks are not implemented. |
| **KMS / HSM signing** | `FURLPAY_KMS_PROVIDER` is read and reported; no signing backend is implemented behind it. |
| **Tokenized stock routing** | No Jupiter or LI.FI integration. The registry and balance reads are the prerequisites, and both are in place. |
| **Order records in Postgres** | Records live in durable KV. Adequate and cross-instance, but not relational, not joinable, and not covered by the DB backup policy. |

### External blockers

- **Alpaca production approval** — live trading requires an approved account.
- **Independent smart-contract audit** — not started; no contract may be called
  a production custody path until it exists.

### Legal / regulatory blockers

- **Broker-dealer registration or a licensed partner** for server-side order
  execution.
- **Money transmitter licensing** per jurisdiction.
- **Securities disclosures and issuer eligibility terms** before any tokenized
  equity is offered.

None of these is solved by code, and nothing in this codebase should be read as
solving them.
