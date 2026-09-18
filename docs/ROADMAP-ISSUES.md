# FurlPay — Top 50 Issues & Top 20 PRs

Companion to `ROADMAP-12-MONTH.md`. Ordered so that **1–10 are done before
anything else is started.** Effort: S <1d · M 1–3d · L 1–2w · XL >2w.

---

## P0 — Blocking (do these first, in this order)

| # | Issue | Where | Effort |
|---|---|---|---|
| 1 | Fresh clone does not build: `page.tsx:24` imports untracked `VerifiedCapabilities.tsx` | `apps/web/src/app/page.tsx` | S |
| 2 | Push 18 unpushed commits; a laptop failure loses a month of work | repo | S |
| 3 | `/api/investing/rwa` has **no authentication** — `GET()` takes no request, reads global store | `api/investing/rwa/route.ts` | S |
| 4 | `/api/investing/schedule` has no authentication | `api/investing/schedule/route.ts` | S |
| 5 | Apply `0009_payments_and_ledger.sql` to staging; 14-day soak | `supabase/migrations/` | M |
| 6 | No error tracking in production (Sentry or equivalent) | infra | M |
| 7 | No uptime monitoring / alerting on the money path | infra | M |
| 8 | No reconciliation job: ledger-vs-chain drift is currently undetectable | `lib/services/` | L |
| 9 | Counsel review: GENIUS yield anti-evasion vs `earn.ts` + all yield copy | legal | M |
| 10 | Counsel review: is swap-to-pay broker-dealer activity? | legal | M |

## P1 — Production payments (Phase 1)

| # | Issue | Effort |
|---|---|---|
| 11 | Decide + document the ONE launch path (USDC/Arbitrum/checkout); everything else labelled mock in-product | S |
| 12 | Unblock `INTEGRATION_MODE=live` for x402 settlement only | L |
| 13 | Set `FACILITATOR_EVM_PRIVATE_KEY` + RPC keys in prod; verify fail-closed when unset | M |
| 14 | Keep `FURLPAY_MAX_*_USD` at $500 until 30 clean days; document the raise criteria | S |
| 15 | Nightly CI e2e: testnet USDC → checkout → settle → receipt → ledger | L |
| 16 | Apply 0009 to prod after soak | M |
| 17 | `0010_idempotency_keys` + enforce on every money-moving POST | L |
| 18 | `0011_receipts` + `/api/receipts` | M |
| 19 | Structured logging with correlation id end-to-end (client already sends `X-Correlation-Id`) | M |
| 20 | Runbook: what to do when settlement fails at 3am | M |
| 21 | Swap execution 503s in prod (`NODE_ENV` gate) — decide: enable, or say so in-product | M |
| 22 | Alert on `INTEGRATION_MODE` drift between envs | S |

## P1 — Deletion track (not optional)

| # | Issue | Effort |
|---|---|---|
| 23 | Audit all 210 routes; classify keep / internal / delete | L |
| 24 | Delete or archive ≥40 demo & marketing routes | L |
| 25 | Move ≥8 AI-framework packages to a separate repo | M |
| 26 | Unpublish or deprecate npm packages that will not be maintained | M |
| 27 | Delete `fintech_founder_opportunities_2026.csv`, `stablecoin_funding_investors_2026.csv`, `wikiexpo-lo.svg` from repo root | S |
| 28 | Enforce route-manifest gate in CI so new routes require classification | M |

## P2 — Enterprise platform (Phase 2)

| # | Issue | Effort |
|---|---|---|
| 29 | `/v1` versioning + deprecation policy | L |
| 30 | Signed webhooks: delivery log, retry, manual replay | L |
| 31 | `0012_api_keys_scopes` — scoped keys | L |
| 32 | `0013_webhook_deliveries` | M |
| 33 | Stable machine-readable error codes across all v1 routes | M |
| 34 | Per-route documented rate limits | M |
| 35 | Make the TypeScript SDK excellent; freeze the rest | L |
| 36 | Public API reference generated from source | L |
| 37 | Time-to-first-payment <30 min (measure it with a real stranger) | M |
| 38 | Status page | S |

## P2 — RWA read-only (Phase 3)

| # | Issue | Effort |
|---|---|---|
| 39 | `ValuationProvider` with `marketState: open\|closed\|halted` + staleness as required fields | L |
| 40 | Per-jurisdiction provider registry (table, not branching) | L |
| 41 | Compliance gate moved **pre-quote** | L |
| 42 | `PortfolioProvider` aggregation across partner accounts | XL |
| 43 | `OrderProvider` abstraction, distinct from `SwapProvider` | XL |
| 44 | Funding-source advisor: proposes only, records consent | L |
| 45 | Test asserting no security can settle to a FurlPay-controlled address | M |
| 46 | `0014_positions_snapshots` | M |

## P3 — Agentic (Phase 4)

| # | Issue | Effort |
|---|---|---|
| 47 | Enterprise governance console for agent policy | XL |
| 48 | HITL approval workflow; timeout = deny | L |
| 49 | Audit export incl. denials + policy version | L |
| 50 | `0015_agent_policy_versions` with immutability trigger | L |

---

## Top 20 pull requests

Small, reviewable, independently shippable. Roughly in order.

1. `fix(build): commit VerifiedCapabilities.tsx` — unbreaks fresh clone
2. `fix(security): require auth on /api/investing/rwa`
3. `fix(security): require auth on /api/investing/schedule`
4. `chore: remove stray CSVs and SVG from repo root`
5. `feat(obs): Sentry + structured logging with correlation id`
6. `feat(obs): uptime + money-path alerting`
7. `db: apply 0009 to staging with rollback note`
8. `feat(recon): ledger-vs-chain reconciliation job + alert`
9. `test(e2e): nightly testnet checkout→settle→receipt→ledger`
10. `feat(settlement): enable INTEGRATION_MODE=live for x402 path only`
11. `db: 0010 idempotency_keys`
12. `feat(api): enforce Idempotency-Key on money-moving POSTs`
13. `db: 0011 receipts` + `feat(api): /api/receipts`
14. `chore(surface): archive AI-framework packages to separate repo`
15. `chore(surface): delete 40 demo/marketing routes`
16. `ci(security): route-manifest gate requires classification`
17. `feat(api): /v1 versioning + deprecation policy`
18. `feat(webhooks): delivery log, retry, replay`
19. `feat(valuation): marketState + staleness as required fields`
20. `feat(compliance): jurisdiction gate runs pre-quote`

---

## Sequencing rule

PRs 1–4 are one afternoon. Do them today. They are the difference between a
repo that survives a laptop failure and one that does not.

Nothing in Phase 2 starts until the reconciliation job (PR 8) has run clean for
30 days. Without it you cannot tell whether the payment path works — you can
only tell whether it errored.
