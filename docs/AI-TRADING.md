# AI trading policy

How an agent is prevented from doing more than a user agreed to.

## What this actually defends against

Not a rogue model. The realistic failure is mundane: a user asks an agent to
"rebalance towards tech", the model's arithmetic is off by a factor of ten, and
nothing between the intention and the venue notices.

Prompt injection through a fetched web page is the same shape with worse intent.
In both cases the model produces a **well-formed, plausible instruction that
nobody authorised** — which is precisely the kind of input that passes every
validation check written to catch malformed data.

## What the engine does and does not judge

It does **not** ask whether an action is *reasonable*. It cannot; that is a
question about the user's finances, goals and risk tolerance, and a model
answering it is the thing we are guarding against.

It answers a much narrower question that it *can* answer correctly:

> Is this action inside the envelope the user agreed to in advance?

## The three properties that make it worth having

### 1. Default deny

```ts
export const DEFAULT_POLICY: TradingPolicy = {
  tradingEnabled: false,
  maxTradeAmountUsd: 0,
  maxDailyAmountUsd: 0,
  allowedSymbols: [],
  // …every limit zero
};
```

A user who has never opened the settings screen has not consented to autonomous
trading. Inferring consent from silence is the thing this file exists to
prevent.

An empty `allowedSymbols` means **none**, never "all". A default-allow engine
protects nobody, because the accounts most needing protection are the ones
nobody configured.

### 2. The engine cannot be addressed by the model

Its inputs are a structured action and a stored policy. There is no field the
model writes that the engine reads as instruction — no `urgency`, no
`confidence`, no free-text `rationale` that could carry *"ignore previous
limits"*.

The model chooses *what* to propose. It has no channel for arguing about
*whether it is allowed*.

### 3. Approval is a separate event

An agent producing a decision and an agent obtaining consent must not be the
same step, or the consent is just another model output.

```
AI intent
  → structured action
  → policy engine        ← this file
  → risk checks
  → trade preview        ← computed, never model-written
  → user approval        ← a separate request, bound to a preview id
  → broker authorization
  → execution
```

The preview is computed from the structured action and real market data, so what
the user reads is what will be submitted. A model-written summary can describe a
different trade from the one in the payload, and the user would have no way to
tell.

Previews expire after 120 seconds — long enough to read, short enough that the
quote behind it is still meaningful. Approval carries a `previewId`; without it,
an approval is a blank cheque for any trade.

## Decision order

Checks run cheapest-and-most-absolute first, so a denied action is denied for
the most fundamental available reason:

1. `read` → always allowed (moves nothing)
2. capability off → `capability_off`
3. user has not enabled agent trading → `trading_disabled`
4. `cancel` → always allowed (reduces exposure)
5. blocked symbol → `symbol_blocked`
6. not on allow-list → `symbol_not_allowed`
7. over per-trade limit → `exceeds_trade_limit`
8. over daily order count → `exceeds_order_count`
9. over daily notional → `exceeds_daily_limit`
10. over biometric threshold → `require_approval` (biometric)
11. over approval threshold → `require_approval` (confirm)
12. otherwise → `allow`

A user told *"exceeds your daily limit"* when trading is disabled entirely would
go and raise their daily limit. Ordering matters.

## Decisions worth explaining

**Sells count against the daily budget.** It is tempting to exempt them because
they raise cash — but an agent that can liquidate a portfolio without limit is
exactly as dangerous as one that can spend without limit.

**Cancellation is always permitted.** A policy that blocks cancellation traps a
user in a position.

**Block-list beats allow-list.** A symbol on both is denied — the safer reading
of a contradictory configuration.

**Order count is bounded separately from size.** Ten $1 trades are individually
compliant and collectively ruinous in fees.

**Spending is measured on a ledger, not a counter.** `dailySpentUsd` is supplied
from actual executed orders. A counter the engine increments itself would reset
on redeploy, drift on a failed execution, and be invisible to reconciliation.

## Hostile stored policy

A policy is user-writable state, so `normalizePolicy` validates and clamps every
field rather than trusting it.

The one that matters most:

```ts
maxTradeAmountUsd: NaN   // `amount > NaN` is false
```

An unvalidated `NaN` limit would pass **every** size check — an infinite limit
written as a typo. Non-finite and negative values clamp to `0`; absurd values cap
at `1,000,000`; `tradingEnabled` must be boolean `true`, so a truthy string
cannot enable trading.

## What is NOT built

| Gap | Status |
|---|---|
| **MCP tool surface** | `buyStock`, `sellStock`, `createTradePreview` etc. are not wired into the MCP server. The policy engine has no caller yet. |
| **Preview storage + approval endpoint** | `TradePreview` and `previewIsValid` exist; the store and the `POST /approve` route do not. |
| **Policy persistence + settings UI** | `normalizePolicy` reads a stored shape; nothing writes one, and there is no screen for a user to set limits. |
| **Daily spend aggregation** | `dailySpentUsd` is a parameter. Nothing computes it from the order store yet. |
| **Risk engine** | Concentration, drawdown and correlation checks are not implemented. The policy engine is a *limits* check, not a risk model. |
| **Portfolio intelligence** | No concentration, diversification, tax-loss-harvesting or rebalancing analysis. |

`AI_TRADING_LIVE` is off by default and requires brokerage credentials before it
can report on — so none of the above is reachable in production today.

## The standing rule

An agent must never present a suggestion as a guaranteed return, and must never
execute above the user's thresholds without a separate human approval event.
Both are enforced in code, not in a prompt.
