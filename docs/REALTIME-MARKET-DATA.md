# Real-time market data

The market-data engine behind Top Gainers, Top Losers and Most Active.

## The measurement that shaped everything

Probed with this deployment's own Alpaca keys on **2026-08-22**:

```
GET /v2/stocks/snapshots?symbols=AAPL&feed=iex  → 200
GET /v2/stocks/snapshots?symbols=AAPL&feed=sip  → 403
        "subscription does not permit querying recent SIP data"
```

We are on Alpaca's **Basic entitlement: the IEX feed**. IEX is one venue, not
the consolidated tape. Measured the same day against consolidated volume:

| Symbol | IEX volume | Consolidated | IEX share |
|---|---:|---:|---:|
| AAPL | 1,017,701 | 42,216,056 | **2.41%** |
| NVDA | 3,206,855 | 91,591,112 | **3.50%** |
| TSLA | 1,136,445 | 57,808,372 | **1.97%** |
| MSFT | 437,111 | 21,856,586 | **2.00%** |
| SPY | 1,272,336 | 38,583,716 | **3.30%** |

Two consequences, and the second decides a product question:

**Prices are real.** An IEX print is a real trade at a real price. Using it for
last price and percent change is sound. Verified: our computed change for AAPL
was −0.65% against consolidated −0.63%; NVDA −0.99% against −0.98%.

**Volume is not market volume.** And the share is **not constant** — 1.97% to
3.50% across five symbols on one day — so it cannot be scaled by a factor
either. Ranking "Most Active" by it answers *"which stock traded most on IEX"*,
which is a different question wearing the same label.

So `rank()` **refuses** a volume board unless the volume it holds came from a
consolidated source:

```
GET /api/markets/movers?board=most_active_volume
→ { "rows": [], "unavailable": { "reason": "volume_not_consolidated", … } }
```

`most_active_dollar` is offered instead, labelled as single-venue.

## Architecture

```
provider (Alpaca IEX)
      ↓  normalizeStreamMessage / fetchSnapshots
canonical MarketEvent
      ↓  MarketDataEngine.apply
per-symbol state  ← dedupe, ordering, sequence, conditions
      ↓  rank()
eligibility gate  ← freshness, halts, liquidity floors
      ↓  topK + total order
leaderboard
      ↓  diffBoards
delta to client
```

## The four guards

A market feed duplicates on reconnect, reorders across venues, drops messages,
and reports trades late. **None of that raises an error** — it silently
corrupts state, and the corruption looks exactly like market movement.

### Deduplication
Keyed on the venue's trade id, namespaced by symbol (Alpaca's id is unique per
symbol per day, not globally). Two prints of the same size at the same price in
the same millisecond are two real trades; only the id distinguishes them.

### Ordering — the asymmetry
A late print describes an earlier moment, so it must not move the last price
backwards. **But it must still count toward volume**, because the shares really
changed hands. Those two facts pull in opposite directions, and a single
`if (isNewer)` guard gets one of them wrong.

### Sequence gaps
A gap means messages were lost. Volume from that point is permanently short and
**cannot be recovered from the stream**. The state carries `sequenceGap: true`
until an authoritative snapshot replaces the running total.

### Sale conditions
Not every print sets the last price. An average-price trade (`W`, `B`) or a late
report (`L`, `Z`) is a real trade at a price nobody could deal at now. Excluding
on an *unrecognised* condition is deliberately **not** done — dropping real
prints on a code we don't understand is the worse error.

## Ranking

**Determinism.** Every comparator ends in `symbol` ascending, which cannot tie.
Without a total order, two symbols at exactly +4.82% swap places between renders
and a reader watches a leaderboard shuffle with no market behind it.

Tie-break chain: `changePercent` → `dollarVolume` → `symbol`.

**Eligibility.** Nothing ranks that can't be justified, and every rejection
carries a reason (`excluded` in the response). Gates: has a price, has a
previous close, both positive, not halted, not stale, freshness known, above
the price floor.

The `minPrice: 1` default exists because a $0.02 stock ticking to $0.03 is a
50% gainer that would own every leaderboard. In production this excludes 61 of
407 symbols.

**Freshness** is judged on the *exchange* timestamp, never receive time — our
network speed is not data freshness. A missing timestamp is `unknown`, not
fresh; a future timestamp is `unknown`, not maximally fresh (otherwise a clock
problem pins a symbol at the top forever).

Thresholds are generous (15s live / 120s degraded) because on a single venue a
liquid symbol can go seconds without an IEX print while trading briskly
elsewhere. A SIP entitlement would justify tightening them.

**topK** is a bounded insertion, not a sort. At K=20 over 10,000 symbols most
candidates lose to the current worst entry on their first comparison. A full
sort would be O(N log N) with a universe-sized allocation per recomputation.

## What is NOT built

Stated plainly.

| Gap | Reality |
|---|---|
| **No streaming in production** | Alpaca's IEX WebSocket works on this tier, but a socket held for hours has nowhere to live on Vercel. `/api/markets/movers` serves snapshots on the quote cache's 60s TTL and says `"transport": "snapshot"`. |
| **30-symbol stream cap** | The binding constraint on Basic is the channel limit, not latency. We cannot stream 409 symbols. |
| **No measured latency** | Nothing here reports p50/p95/p99, because nothing is streaming to measure. Any latency figure would be invented. |
| **No consolidated volume** | Requires Algo Trader Plus or above. |
| **No LULD bands** | Halts map to a status; the price bands are not surfaced. |
| **No session separation** | Pre-market / regular / after-hours are modelled in the type but not yet computed against session-specific reference prices. |
| **`previousClose` is reconstructed** | The movers route derives it from price ÷ (1 + change%), because the shared quote cache does not carry it. Correct to the precision of the change figure; the snapshot path (`fetchSnapshots`) reads the real `prevDailyBar.c`. |

## This is not HFT

The engineering discipline here — event-driven, dedupe, sequence handling,
incremental top-K, deterministic ordering — is the discipline a low-latency
market-data system needs. **The infrastructure is not.** We are on a
single-venue feed, served from serverless functions, on snapshots.

Nothing in this codebase should be described as HFT, and no latency claim should
appear anywhere until there is a stream to measure.

## Upgrade path

1. **Algo Trader Plus** → SIP entitlement. `detectEntitlement()` picks it up
   automatically; volume boards start working with no code change.
2. **A long-lived worker** outside Vercel → feed `engine.apply()` from the
   WebSocket. The engine is already written for it.
3. **Then** measure latency, and only then report numbers.
