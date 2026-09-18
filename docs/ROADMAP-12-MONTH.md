# FurlPay — 12-Month Execution Roadmap

**Author:** CTO / CPO / technical GC synthesis
**Date:** 2026-07-23
**Horizon:** 2026-08 → 2027-07
**Constraint:** one engineer. Everything below is scoped to that.

---

## 0. The constraint nobody wrote down

The repo has **210 API routes** and **24 packages**. One person maintains them.
That is the binding constraint on this roadmap — not ambition, not funding, not
regulation.

Every route is a security boundary that must be audited, rate-limited, and kept
working. Every package is an npm surface with a CVE inbox and a support
obligation. Twelve packages are AI-framework adapters that exist to be
integrated *with*, not sold.

**A roadmap that only adds will not survive the year.** Phase 1 therefore has a
deletion track that is not optional and not deferrable. The goal for month 12 is
not "more" — it is *one payment path that provably works in production, and a
surface small enough that one person can defend it*.

Second observation: the agentic stack is the most-built and least-productised
thing here. `/api/agents/keys`, `/api/agents/ap2`, and eight `/api/x402/*`
routes already exist. Track C is not greenfield. It is 70% built and 0% sold.

---

## Part 1 — Vision, rewritten for what is true today

### 30-word positioning

> FurlPay is settlement infrastructure for programmable money. We move
> stablecoins for merchants and give autonomous agents spending authority under
> enforceable policy. We orchestrate regulated partners; we never become one.

### One-sentence pitch

FurlPay turns any balance into a settled merchant payment — with a policy check
before it moves and an auditable receipt after.

### Investor pitch

Stablecoin payment volume is commoditising: Stripe bought Bridge, Circle owns
issuance, and merchant acceptance margin is compressing toward zero. The
un-commoditised layer is **authority** — deciding what an autonomous agent is
allowed to spend, on whose behalf, under what limits, with what audit trail. We
have a running x402 facilitator, an agent-spend control plane in production, and
published adversarial research on the five failure classes of agentic payment
protocols. We monetise through interchange and enterprise API, not float or AUM,
so we stay asset-light and out of the licensing perimeter.

### Enterprise pitch

Give your AI agents a spending limit that is actually enforced. FurlPay issues
scoped agent credentials, evaluates every payment against your policy before it
executes, escalates to a human above your threshold, and writes an immutable
audit record. Your treasury keeps custody. We never touch your securities.

### Developer pitch

One API for stablecoin payments across EVM and Solana. Server-issued single-use
quotes, idempotency keys on every mutation, signed webhooks, an MIT-licensed
x402 facilitator you can self-host, and adapters for the agent framework you
already use.

### What we explicitly do NOT build

Custody of securities · broker-dealer execution · a lending book · a stablecoin ·
a consumer super-app · an exchange · a bank · an RIA/advisory product.

---

## Part 2 — Three tracks

### Track A — Payments *(pays rent · ship first)*

| | |
|---|---|
| **Mission** | One stablecoin payment path that works in production, end to end, with a receipt |
| **Customer** | Crypto-native SMB merchants, API-first platforms, marketplaces |
| **Revenue** | Merchant fee (compressing) · FX spread (durable) · swap routing (live: LI.FI integrator fee) |
| **Core APIs** | `/api/checkout` · `/api/payments` · `/api/swaps` · `/api/receipts` · `/api/webhooks` |
| **Expansion** | Card interchange (Q3) · invoice financing (never — that's lending) |

Track A is not the differentiator. It is the reason anyone takes a meeting.

### Track B — Treasury *(defer most of it)*

| | |
|---|---|
| **Mission** | Read-only visibility across tokenized + traditional balances; funding-source selection |
| **Customer** | Crypto-native companies holding idle USDC |
| **Revenue** | Enterprise API subscription. **Not** AUM fees — that needs licences |
| **Core APIs** | `/api/overview` · `/api/investing/portfolio` · `/api/investing/rwa` (auth-fixed) |
| **Expansion** | Tokenized T-bill as a *funding source*, via partner, Q4+ |

**The GENIUS constraint.** The OCC's Feb-2026 NPRM creates a rebuttable
presumption that an issuer paying an affiliate who routes yield to holders
violates the interest prohibition. `lib/services/earn.ts` and any "idle cash
earns yield" copy need counsel review **before** Phase 2, not after. The
surviving carve-out is *merchant discounts for stablecoin payment* — build that
instead. It is compliant, it is a real acquisition lever, and it does not
require a licence.

### Track C — Agentic Commerce *(the actual moat)*

| | |
|---|---|
| **Mission** | The authority layer for autonomous spend |
| **Customer** | AI platform teams, agent framework vendors, enterprise automation |
| **Revenue** | Per-seat/per-agent subscription + volume fee. Highest margin, hardest to copy |
| **Core APIs** | `/api/agents/keys` · `/api/agents/ap2` · `/api/x402/*` · `/api/intents` |
| **Expansion** | Enterprise governance console · SOC 2 · policy marketplace |

Everything here exists in some form. The work is productisation, not invention:
docs, SDK ergonomics, a governance UI, and one referenceable design partner.

---

## Part 3 — RWA strategy without becoming a securities platform

### Inside FurlPay

- **Read-only aggregation.** Display balances held elsewhere. Displaying is not
  custody, brokerage, or advice.
- **Valuation display** with mandatory staleness + market-state annotation.
- **Funding-source *suggestion*** — ranked options, user selects, consent recorded.
- **Post-trade reconciliation** against partner statements.
- **Receipts and audit export.**

### Always delegated

Issuance · custody · order execution · KYC/AML of securities accounts ·
corporate actions of record · lending origination · suitability assessment.

### Legal boundaries (write these into code, not just policy)

1. **No security ever settles into a FurlPay-controlled address.** Enforce in
   `SettlementProvider`; test it.
2. **No automated disposal.** The engine proposes; the user confirms. Anything
   else is investment discretion.
3. **Jurisdiction gate runs before quoting**, not before settlement. A blocked
   user must never see a price.
4. **Provider registry is per-jurisdiction data**, not branching logic. Backed
   is unreachable for US persons; Dinari serves US accredited via BD custody.
   That is a table, and it is testable.

### Never build

Custody wallet for securities · matching engine · margin/liquidation engine ·
robo-advisor · own tokenized asset · securities-backed credit on own balance sheet.

---

## Part 4 — Four phases

### Phase 1 — Production Payments *(Aug–Oct 2026)*

**Objective:** one stablecoin path works in prod, provably, with a receipt. And
the surface shrinks.

**Do not** harden everything. Pick USDC on Arbitrum, merchant checkout, one
flow. Make it real. Everything else stays mock and *says so*.

- **Architecture:** unblock `INTEGRATION_MODE=live` for the x402 settlement path
  only. Keep `FURLPAY_MAX_*_USD` at $500 until 30 days of clean reconciliation.
- **DB:** apply `0009_payments_and_ledger.sql` to staging → 14-day soak → prod.
  Then `0010_idempotency_keys`, `0011_receipts`.
- **APIs:** `/api/checkout`, `/api/payments`, `/api/receipts`. Freeze the rest.
- **Infra:** error tracking (Sentry), uptime monitoring, a reconciliation cron
  that alerts on ledger-vs-chain drift. **You currently have none of this — it is
  the single largest production risk, larger than any missing feature.**
- **Testing:** end-to-end test that moves real USDC on testnet through
  checkout → settle → receipt → ledger, run in CI nightly.
- **Deletion track:** archive ≥8 AI-framework packages to a separate repo;
  deprecate or delete ≥40 demo/marketing API routes.
- **Success:** 100 real payments settled · zero ledger drift over 30 days ·
  route count < 170 · packages < 16 · fresh clone builds.

### Phase 2 — Enterprise Platform *(Nov 2026 – Jan 2027)*

**Objective:** a second company can integrate without talking to you.

- Idempotency on every mutating route (contract-tested, not documented).
- Signed webhooks + replay endpoint + delivery log.
- API versioning (`/v1`), deprecation policy, changelog.
- One SDK excellent (TypeScript) rather than nine adequate.
- `0012_api_keys_scopes`, `0013_webhook_deliveries`.
- **Success:** one external integration in prod without founder involvement ·
  p99 < 500ms · 99.9% uptime measured, not claimed.

### Phase 3 — Treasury & RWA read-only *(Feb–Apr 2027)*

- Fix `/api/investing/rwa` auth (**Phase 1 hotfix**, listed here for completeness).
- `ValuationProvider` with `marketState: open|closed|halted` + staleness.
- `PortfolioProvider` aggregation across partner accounts.
- Funding-source advisor: proposes, never executes.
- `0014_positions_snapshots`.
- **Success:** 10 companies aggregating · zero disposals initiated by FurlPay.

### Phase 4 — Autonomous Finance *(May–Jul 2027)*

- Enterprise governance console for agent policy.
- Approval workflows + HITL above `FURLPAY_HITL_THRESHOLD_USD`.
- Recurring autonomous payments with budget windows.
- Full audit export.
- **Success:** 3 enterprise design partners · 1000 agent-initiated payments ·
  SOC 2 Type I started.

---

## Part 5 — Final architecture

Boundaries matter more than boxes. The rule: **regulated activity never crosses
into a FurlPay-controlled process.**

| Engine | Owns | Must never |
|---|---|---|
| **Payment** | Intent lifecycle, funding selection, idempotency | Hold funds |
| **Settlement** | On-chain execution, confirmation, retry | Settle a security to a FurlPay address |
| **Swap** | Sync atomic quotes (LI.FI, Jupiter), TTL, single-use quoteId | Model an order |
| **Order** | Async fills, venue hours, partial fills, state machine | Use a 30s TTL |
| **Portfolio** | Aggregation, positions, cost basis display | Be valuation of record |
| **Compliance** | Jurisdiction, sanctions, tier limits — **pre-quote** | Run after execution |
| **Policy** | Agent spend rules, budgets, HITL escalation | Be bypassable by any caller |
| **Treasury** | Funding-source ranking, proposals | Execute a disposal |
| **Notification** | Webhooks, push, delivery guarantees | Be in the money path |
| **Audit** | Append-only event log, export | Be mutable. Ever |
| **API Gateway** | AuthN/Z, rate limits, versioning, idempotency | Contain business logic |

The Swap/Order split is the one architectural decision that must not be
compromised. A swap is atomic and priced for 30 seconds. An order has venue
hours, partial fills, and a settlement cycle. Modelling the second as the first
produces fills that silently never happen.

---

## Part 6 — Partner ranking

Effort/value on 1–5. Ranked by *what unblocks revenue soonest*.

| Rank | Partner | Category | Why | Effort | Distribution | Regulatory | Long-term |
|---|---|---|---|---|---|---|---|
| 1 | Card issuer (Marqeta/Stripe Issuing class) | Cards | Interchange = largest durable revenue | 4 | 5 | 4 | 5 |
| 2 | Circle | Stablecoins | USDC + CPN; already core | 2 | 4 | 3 | 5 |
| 3 | Sumsub | Identity | Already integrated — deepen | 1 | 2 | 5 | 4 |
| 4 | LI.FI | Cross-chain | Live, fee-earning | 1 | 3 | 1 | 3 |
| 5 | Jupiter | Solana | Unblocks every Solana pair | 3 | 3 | 1 | 3 |
| 6 | Fireblocks / Anchorage | Custody | Enterprise credibility | 4 | 4 | 5 | 5 |
| 7 | Dinari / tZERO | Brokerage | Only compliant US tokenized-equity path | 3 | 3 | 5 | 4 |
| 8 | Ondo | RWA / Treasury | Best tokenized T-bill funding source | 3 | 3 | 4 | 4 |
| 9 | ComplyAdvantage | Compliance | Webhook already exists | 2 | 1 | 5 | 4 |
| 10 | Backed | RWA issuer | Non-US only — jurisdiction-gated | 3 | 2 | 4 | 3 |
| 11 | Morpho / Figure | Lending | Referral only. **Never originate** | 3 | 2 | 5 | 2 |
| 12 | MAS Project Guardian | Regulatory | Institutional credibility, SG track | 5 | 2 | 5 | 4 |

---

## Part 7 — API platform

**Non-negotiables, applied uniformly:**

- **Auth:** API key (server) + session JWT (app) + DPoP-bound tokens (MCP).
  Scopes per key.
- **Versioning:** `/v1` prefix, additive-only within a major, 6-month deprecation.
- **Idempotency:** `Idempotency-Key` required on every POST/PATCH that moves
  money. Stored, replayed, contract-tested.
- **Rate limits:** documented per route; the middleware baseline (150/min) is a
  backstop, not a policy.
- **Errors:** stable machine-readable `error` codes + human `detail`. Never leak
  internals.
- **Webhooks:** HMAC over raw body, timestamp tolerance, delivery log, manual replay.

Surface for v1 — **freeze here**: payments · checkout · invoices · cards ·
swaps · balances · portfolios · agents · x402 · webhooks. Everything else is
internal or deleted.

---

## Part 8 — Agentic commerce (the differentiator)

The pieces exist. What is missing is the product around them.

- **Agent wallets** — scoped credentials, not private keys. Revocable, per-agent.
- **Policy** — declarative: per-tx cap, daily/monthly budget, merchant allow/deny,
  category, time window, jurisdiction. Evaluated server-side, pre-execution,
  fail-closed.
- **Merchant authorization** — agent proves authority; merchant verifies without
  learning the principal's identity.
- **Budget enforcement** — atomic decrement (Redis primitives already shipped).
  Concurrency-safe or it is decorative.
- **HITL** — above `FURLPAY_HITL_THRESHOLD_USD`, escalate: push → approve/deny →
  audit. Timeout = deny.
- **x402** — the facilitator is the wedge. Keep it MIT and self-hostable; that is
  why anyone trusts it.
- **Audit** — every decision, including denials, with the policy version that
  produced it. Denials are the compliance artifact that sells this.

Publishing the x402-guard adversarial work is a *distribution* strategy, not a
vanity exercise. It is how a solo founder earns enterprise credibility without a
sales team.

---

## Part 9 — KPIs

| Area | Metric | 12-mo target |
|---|---|---|
| Engineering | Route count · package count | <170 · <16 |
| Reliability | Uptime · p99 · ledger drift | 99.9% · <500ms · 0 |
| Payments | Settled volume · failure rate | — · <0.5% |
| Product | Time-to-first-payment (new dev) | <30 min |
| Revenue | MRR · interchange | — |
| Enterprise | Design partners · integrations w/o founder | 3 · 1 |
| Developer | SDK weekly installs · docs → key conversion | — |
| Security | Open Dependabot alerts · unauthenticated money routes | 0 · 0 |
| Compliance | Jurisdiction-gate coverage · audit completeness | 100% · 100% |
| Treasury | Companies aggregating · disposals initiated by FurlPay | 10 · **0** |

---

## Part 10 — Deliverables

### Quarterly milestones

| Quarter | Milestone | Gate |
|---|---|---|
| Q3 2026 | Prod payments live, capped $500 | 30 days zero drift |
| Q4 2026 | Enterprise API v1, one external integration | No founder involvement |
| Q1 2027 | RWA read-only aggregation | Zero FurlPay-initiated disposals |
| Q2 2027 | Agentic governance + 3 design partners | SOC 2 Type I started |

### Migration plan

| # | Name | When | Risk |
|---|---|---|---|
| 0009 | payments_and_ledger | **Now** — staging, 14-day soak, then prod | Medium — has the settle-before-status-flip ordering fix |
| 0010 | idempotency_keys | Phase 1 | Low |
| 0011 | receipts | Phase 1 | Low |
| 0012 | api_keys_scopes | Phase 2 | Low |
| 0013 | webhook_deliveries | Phase 2 | Low |
| 0014 | positions_snapshots | Phase 3 | Low |
| 0015 | agent_policy_versions | Phase 4 | Medium — audit immutability |

Rule: every migration is forward-only, applied to staging first, and paired with
a rollback note. No exceptions when tired.

### Top 10 things to never build

1. Securities custody
2. Broker-dealer execution
3. Own lending book / balance-sheet credit
4. Own stablecoin
5. Matching engine or exchange
6. Robo-advisor / discretionary allocation
7. Margin + liquidation engine
8. Own tokenized asset issuance
9. Consumer super-app
10. Anything requiring a banking charter

---

## Compliance roadmap

| When | Action |
|---|---|
| **Immediately** | Counsel review: GENIUS yield anti-evasion vs `earn.ts` and all yield copy |
| **Immediately** | Counsel review: is swap-to-pay broker-dealer activity even when routed to a licensed venue? |
| Phase 1 | Jurisdiction gate pre-quote; per-jurisdiction provider registry |
| Phase 2 | Data retention + GDPR review; audit export |
| Phase 3 | Partner agreements: Dinari/Ondo. Verify SEC innovation exemption **actually landed** — as of this writing there is reporting, not a final rule |
| Phase 4 | SOC 2 Type I; Singapore MAS track evaluation |

Preserve the existing honesty on `/ae/trust` ("not currently VARA-licensed").
It is a genuine trust asset and cheaper to keep than to rebuild.
