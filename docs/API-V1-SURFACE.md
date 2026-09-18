# FurlPay v1 API Surface

**Status:** proposed · supersedes ad-hoc route growth
**Companion to:** `ROADMAP-12-MONTH.md`, `ROADMAP-ISSUES.md`

---

## Why this is smaller than the 175-endpoint proposal

The current repo has **210 routes**. A ~175-endpoint target sounds like a
reduction. It is not — nearly every domain in that proposal (organizations,
RBAC, API keys, invoices, receipts, bridge, RWA orders, treasury, policies,
analytics, admin, merchant) is **net new**. Kept alongside what exists, the real
total lands near **330**. That is 60% growth described as focus.

Endpoint count is not the metric anyway. **Every route that moves money is a
security boundary that must be authenticated, rate-limited, idempotent,
audited, and tested.** One person can defend perhaps 100 of those well. The
target below is ~105, staged across four phases, and **Phase 1 ships 34**.

---

## Endpoints that must never exist

These appeared in the proposal. Each one converts FurlPay from an orchestrator
into a regulated entity, or creates an unrecoverable security hole.

| Endpoint | Why never |
|---|---|
| `POST /v1/wallets/sign` | Server-side signing **is custody**. Verified: the server currently holds no private keys — `wallet/link/*` is challenge/verify proof-of-ownership. This single route triggers custody licensing in every jurisdiction and makes the company a theft target |
| `POST /v1/wallets/export` | Exports private keys over HTTP. There is no design that makes this safe |
| `POST /v1/wallets/import` | Same, inbound. Accepting a user's key makes you the custodian of it |
| `POST /v1/rwa/orders` | Placing securities orders **is broker-dealer activity** — directly contradicts the stated position |
| `POST /v1/treasury/deposit` + `GET /v1/treasury/yield` | GENIUS Act yield anti-evasion exposure (OCC NPRM, Feb 2026). Needs counsel before any yield-facing surface |
| `GET /metrics` (unauthenticated) | Operational metrics are reconnaissance. Admin-gate it |
| `POST /v1/swaps/:id/cancel` | An on-chain swap cannot be cancelled once submitted. Shipping this endpoint promises something physics does not allow |
| `POST /v1/payments/:id/refund` | On irreversible rails a refund is a **new payment in the opposite direction**, not a reversal. Model it as such or users will assume chargeback semantics that do not exist |

---

## Phase 1 — Production Payments (34 endpoints)

Everything needed for one real payment path. Nothing else.

### Auth (9)
```
POST   /v1/auth/register
POST   /v1/auth/login
POST   /v1/auth/logout
POST   /v1/auth/refresh
POST   /v1/auth/passkeys/register
POST   /v1/auth/passkeys/login
GET    /v1/auth/me
PATCH  /v1/auth/profile
DELETE /v1/auth/account
```
MFA enable/verify already exist under `/api/security/mfa` — migrate, don't
duplicate. Email verification folds into register.

### Payments (6)
```
POST /v1/payments                 # Idempotency-Key required
GET  /v1/payments
GET  /v1/payments/:id
POST /v1/payments/:id/execute     # Idempotency-Key required
POST /v1/payments/:id/cancel      # pre-submission only
GET  /v1/payments/:id/status
```
No `/approve` in Phase 1 — approval is agent-policy surface (Phase 4).
No `/refund` — see above.

### Checkout (5)
```
POST   /v1/checkout
GET    /v1/checkout/:id
PATCH  /v1/checkout/:id
DELETE /v1/checkout/:id
POST   /v1/checkout/:id/pay
```
Expiry is a background job, not an endpoint.

### Receipts (3)
```
GET /v1/receipts
GET /v1/receipts/:id
GET /v1/receipts/:id.pdf
```
One canonical download; `/pdf` and `/download` as separate routes is duplication.

### Balances & Transactions (5)
```
GET /v1/balances
GET /v1/balances/history
GET /v1/transactions
GET /v1/transactions/:id
GET /v1/transactions/export
```
`/net-worth` is a field on `/v1/balances`, not a route. `/transactions/history`
is `/transactions` with a date filter.

### Swaps (3)
```
POST /v1/swaps/quote              # issues single-use quoteId
POST /v1/swaps                    # executes a quoteId; Idempotency-Key required
GET  /v1/swaps/:id
```
Quote is POST, not GET — it has side effects (it issues and stores a quoteId).
This is already how `/api/swaps` behaves; keep it.

### Health (3)
```
GET /livez      # unauthenticated, leaks nothing, "is the process up"
GET /readyz     # unauthenticated, dependency-aware, gates traffic
GET /v1/version # unauthenticated, build sha only
```
`/api/ops/health` already does real probes but is admin-gated — an uptime
monitor cannot use it. These three are the public-safe complement, not a
replacement.

---

## Phase 2 — Enterprise Platform (+28 → 62)

### Organizations & RBAC (9)
```
GET    /v1/orgs
POST   /v1/orgs
GET    /v1/orgs/:id
PATCH  /v1/orgs/:id
GET    /v1/orgs/:id/members
POST   /v1/orgs/:id/members
PATCH  /v1/orgs/:id/members/:memberId
DELETE /v1/orgs/:id/members/:memberId
POST   /v1/orgs/invitations/accept
```
**Genuinely net-new** — `authz.ts` currently pins admin to a single
`FOUNDER_ADMIN_EMAIL` constant. There is no org model at all. Org deletion is a
support operation with a retention period, not a DELETE route.

### API Keys (5)
```
GET    /v1/api-keys
POST   /v1/api-keys
PATCH  /v1/api-keys/:id
DELETE /v1/api-keys/:id
POST   /v1/api-keys/:id/rotate
```

### Webhooks (6)
```
GET    /v1/webhooks
POST   /v1/webhooks
PATCH  /v1/webhooks/:id
DELETE /v1/webhooks/:id
GET    /v1/webhooks/:id/deliveries
POST   /v1/webhooks/deliveries/:deliveryId/replay
```
Replay targets a *delivery*, not an endpoint — replaying "the webhook" is
ambiguous about which event.

### Invoices (5)
```
POST   /v1/invoices
GET    /v1/invoices
GET    /v1/invoices/:id
PATCH  /v1/invoices/:id
POST   /v1/invoices/:id/pay
```

### Developer (3)
```
GET /v1/openapi.json
GET /v1/changelog
GET /v1/status
```
SDK and examples are docs-site content, not API endpoints.

---

## Phase 3 — Treasury & RWA read-only (+22 → 84)

### Portfolio (4)
```
GET /v1/portfolio
GET /v1/portfolio/history
GET /v1/portfolio/allocation
GET /v1/portfolio/performance
```

### Markets (4)
```
GET /v1/markets/quotes
GET /v1/assets/search
GET /v1/assets/:id
GET /v1/assets/:id/valuation    # MUST carry marketState + staleness
```
One asset namespace, filtered by class — not four parallel `/markets/*` routes
that will drift.

### RWA — read-only (4)
```
GET /v1/rwa/holdings
GET /v1/rwa/valuation
GET /v1/rwa/providers           # per-jurisdiction availability
GET /v1/rwa/orders              # READ partner order state; we never place
```
Note the asymmetry: `GET /orders` reads state from a partner. There is no POST.
That asymmetry **is** the legal boundary, expressed in the routing table.

### Funding advisor (2)
```
POST /v1/funding/options        # ranked proposals, no execution
POST /v1/funding/select         # records user consent
```

### Cards (8)
```
GET    /v1/cards
POST   /v1/cards
GET    /v1/cards/:id
PATCH  /v1/cards/:id
DELETE /v1/cards/:id
POST   /v1/cards/:id/freeze
POST   /v1/cards/:id/unfreeze
POST   /v1/cards/:id/pin
```

---

## Phase 4 — Agentic (+21 → 105)

### Agents (7)
```
POST   /v1/agents
GET    /v1/agents
GET    /v1/agents/:id
PATCH  /v1/agents/:id
DELETE /v1/agents/:id
POST   /v1/agents/:id/rotate-credential
GET    /v1/agents/:id/activity
```

### Policies (6)
```
GET    /v1/policies
POST   /v1/policies
GET    /v1/policies/:id
PATCH  /v1/policies/:id
POST   /v1/policies/:id/simulate
POST   /v1/policies/:id/publish
```
Published policy versions are **immutable**. `simulate` before `publish` is the
feature enterprises will actually pay for.

### Approvals / HITL (4)
```
GET  /v1/approvals
GET  /v1/approvals/:id
POST /v1/approvals/:id/approve
POST /v1/approvals/:id/deny
```
Timeout defaults to deny.

### Audit (2)
```
GET /v1/audit/events
GET /v1/audit/export
```
Includes denials with the policy version that produced them.

### Compliance (2)
```
GET  /v1/compliance/status
POST /v1/compliance/kyc
```
Jurisdiction and risk are fields on `/status`, not separate routes. AML runs
server-side on every payment — it is not a client-callable endpoint.

---

## Not in v1

**Merchant namespace** — `/v1/merchant/payments` and `/v1/payments` scoped to a
merchant org are the same data. Use org scoping, not a parallel tree. This alone
removes 7 endpoints and an entire class of drift bug.

**Analytics namespace** — 5 endpoints of aggregation nobody has asked for yet.
Ship one `/v1/analytics/summary` when a customer requests it.

**Admin namespace** — `/api/ops/*` already exists and works, with
`requireOpsAdmin`. Do not build a second admin surface.

**Notifications CRUD** — push preferences are 2 fields on `/v1/auth/profile`.

**Bridge** — a cross-chain swap is a swap with `fromChain != toChain`. It is
already modelled that way. A parallel `/bridge` tree duplicates quoting,
execution, idempotency and status for zero semantic gain.

---

## Cross-cutting rules

| Rule | Applies to |
|---|---|
| `Idempotency-Key` **required** | every POST that moves money |
| Zod validation | every POST/PATCH body (AGENTS.md rule 1) |
| Rate limit documented | every route (AGENTS.md rule 2) |
| HMAC + timestamp | every inbound webhook (AGENTS.md rule 3) |
| Stable `error` code + human `detail` | every error response |
| Additive-only within v1 | 6-month deprecation for removals |

---

## Migration path from 210 routes

1. **Freeze.** No new routes outside this document.
2. **Classify** all 210: keep-as-is / migrate-to-v1 / internal / delete.
3. **Alias, don't rewrite.** `/api/x` → `/v1/x` where the handler is already
   correct. Most Phase 1 endpoints already exist under `/api/`.
4. **Delete the demo tier** — marketing, hackathon, swag, ambassadors, events.
   These are content, not API.
5. **Enforce in CI** via the existing route-manifest validator.

Realistic outcome: **~105 v1 routes, ~40 internal/ops, everything else deleted.**
Net reduction from 210 to ~145 total — while *adding* orgs, RBAC, API keys and
agent governance.
