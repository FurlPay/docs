# FurlPay — Production Readiness

**Updated:** 6 September 2026
**Verdict:** **READY FOR TESTNET.** Not ready for real-money private beta. Not ready for mainnet.

Statuses: **READY** · **IN PROGRESS** · **BLOCKED** · **REQUIRES EXTERNAL DEPENDENCY** · **NOT STARTED**

Nothing is READY because code exists. READY means implemented, tested, and evidence someone can point at.

---

## Financial correctness

| Item | Status | Evidence |
|---|---|---|
| Exact money, no floats | READY | `lib/money/money.ts`; 20 tests incl. exactness past `Number.MAX_SAFE_INTEGER` |
| Asset carries settlement domain | READY | `USDC.arbitrum` ≠ `USDC.base`; addition across them throws |
| Fee/FX as rationals with explicit rounding | READY | `Money.scale(n, d, mode)` — no default mode; rounding is a business decision |
| Split preserves the whole | READY | `allocate()` distributes remainder; sum invariant tested across shapes |
| Money settled only on verifiable evidence | READY | `REQUIRED_EVIDENCE` + `assertTransition`; 20 lifecycle tests |
| Transaction envelope (11 attributes) | READY (schema) / BLOCKED (unapplied) | migration `0014` |
| Peg never assumed at par | READY | `pegHealth()` returns `unknown` for absent/stale; settlement refused |

## Ledger

| Item | Status | Evidence |
|---|---|---|
| Double-entry, balance checked inside the write | READY | `ledger_post()` in `0009` |
| Append-only history | READY | Triggers on `ledger_postings` and `financial_transaction_events` |
| Corrections are compensating entries | READY | Double-reversal and reversal-of-reversal refused, in TS and SQL |
| Customer liability modelled as a liability | READY | Was an asset; inverted. 20 chart-of-accounts tests |
| Customer / corporate separation | READY | `CUSTOMER_FUND_ASSETS`; `holdsFor()` in the treasury register |
| Durable-or-refuse in production | READY | `LedgerDurabilityError` |
| **Migrations applied** | **BLOCKED** | Only `0006` has rollout evidence. Requires your credentials |
| Negative-balance prohibition at DB level | NOT STARTED | Enforced per-route only |

## Reconciliation

| Item | Status | Evidence |
|---|---|---|
| Internal consistency checks | READY | 7 checks in `lib/reconciliation.ts` |
| Alerting on breaks | READY | `/api/cron/reconcile`, fingerprinted per check |
| Explicit break states | READY | `reconciliation_status` on every transaction; 4 SQL views |
| Never silently repairs | READY | Detection only; `docs/RECONCILIATION.md` states why |
| Bank ⇄ ledger | REQUIRES EXTERNAL DEPENDENCY | `getStatement` port exists; no partner |
| Provider ⇄ ledger | REQUIRES EXTERNAL DEPENDENCY | `getStatus` port exists; no provider |
| Chain ⇄ ledger | NOT STARTED | Needs an indexer |
| Treasury coverage / solvency | IN PROGRESS | `coverage()` implemented; register empty so it always reports a shortfall |

## Banking

| Item | Status | Evidence |
|---|---|---|
| Rail interfaces (UPI/IMPS/NEFT/RTGS) | READY | `lib/rails/india/types.ts` |
| Initiation ≠ acknowledgement ≠ confirmation ≠ settlement | READY | `RailStage`; `isSpendable()` is `settled`-only |
| Fail-closed endpoints | READY | `/api/banking` POST returns 503 in every environment |
| Rail adapter | REQUIRES EXTERNAL DEPENDENCY | `indiaRailAdapter()` returns `null` |
| PA contract / payout partner / escrow / sponsor bank | REQUIRES EXTERNAL DEPENDENCY | — |
| `BANKING_LIVE` cannot be enabled | READY | `unimplemented` reason outranks any env var |

## Cards

| Item | Status | Evidence |
|---|---|---|
| Issuing disabled, no fake cards or PANs | READY | `CARD_ISSUING_LIVE`; 8 routes gated |
| Lifecycle model | IN PROGRESS | Service exists; not yet a provider port |
| Issuer / BIN sponsor / PCI scope | REQUIRES EXTERNAL DEPENDENCY | — |
| Wallet provisioning | NOT STARTED | UI toggle with an explicit `TODO(backend)` |

## Blockchain

| Item | Status | Evidence |
|---|---|---|
| Mainnet USDC addresses verified | READY | All 5 against the issuer's own published list |
| Confirmation depth scaled by amount | READY | Enforced by the facilitator |
| Replaced / dropped transactions | READY | viem `onReplaced`; terminal `abandoned` |
| Testnet addresses | NOT STARTED | Base Sepolia, Ethereum Sepolia, Solana devnet absent |
| Chain adapter abstraction | NOT STARTED | **Top engineering blocker** |
| Reorg-aware deposit crediting | NOT STARTED | — |
| Nonce / gas management | NOT STARTED | — |
| RPC failover | IN PROGRESS | Supported and readiness-checked; URLs not provisioned |
| Solana | NOT STARTED | Safe is EVM-only |

## Smart contracts

| Item | Status | Evidence |
|---|---|---|
| Inventory | READY | 6 contracts |
| Deployed | NOT STARTED | All `pending_deployment` |
| External audit | REQUIRES EXTERNAL DEPENDENCY | **No mainnet deploy before this** |
| Upgrade / admin authority documented | NOT STARTED | — |

## Security

| Item | Status | Evidence |
|---|---|---|
| Route auth manifest enforced in CI | READY | 302 routes; drift fails the build |
| Webhook signature / replay / ordering | READY | Per-provider schemes; Redis dedup; high-water mark |
| Idempotency scoped per principal | READY | Route-level + DB unique index |
| KMS boundary, no key in application code | READY | Production refuses a raw env key |
| Failover cannot double-pay | READY | 20 router tests |
| Transaction simulation before signing | NOT STARTED | — |
| Penetration test | REQUIRES EXTERNAL DEPENDENCY | — |

## Compliance

| Item | Status | Evidence |
|---|---|---|
| KYC tiers + caps | READY | Server-side |
| KYC cannot be demoted by a webhook | READY | 5 tests |
| Both parties screened | READY | Payer + destination |
| Sanctions coverage | IN PROGRESS | OFAC crypto-addresses only, community mirror |
| Wallet risk / analytics | REQUIRES EXTERNAL DEPENDENCY | Deliberately not faked |
| Travel Rule | NOT STARTED | Pure function; transmits nothing |
| India CDD (PAN/OVD/CKYC) | NOT STARTED | Required by the FIU-IND 2026 guidelines |
| EDD triggers (unhosted wallets, mixers) | NOT STARTED | Named higher-risk by those guidelines |
| Account freeze / hold / LEA | NOT STARTED | — |
| FIU-IND registration | REQUIRES EXTERNAL DEPENDENCY | — |
| Restricted capability cannot be enabled by accident | READY | `unimplemented` outranks env; 12 money flags |

## Fraud / identity / risk

| Item | Status | Evidence |
|---|---|---|
| Reason-code registry | READY | `lib/trust/reasons.ts`; 31 codes; standalone set capped and tested |
| Trust score 0–1000 | READY | `lib/trust/score.ts`; noisy-OR, decay, confidence; 25 tests |
| Confidence guard on automated decline | READY | High score + thin evidence → `manual_review` |
| Entity graph + ring detection | READY (analysis) | `lib/trust/graph.ts`; 19 tests incl. 4 that prove it does NOT fire |
| Feedback-loop protection | READY | `auto_declined` does not propagate; only human-confirmed does |
| Adaptive verification | READY | `lib/trust/verification.ts`; 16 tests |
| **Graph is populated with data** | NOT STARTED | Analysis exists; nothing feeds it |
| **Assessments persisted / continuous trust** | NOT STARTED | Every call recomputes from nothing |
| **Wired into money routes** | NOT STARTED | `/api/wallets/transfer` still calls the old 4-signal engine |
| Device SDK (iOS / Android) | NOT STARTED | Browser collector only |
| Behavioural baselines | NOT STARTED | Collector exists; no baseline store |
| Synthetic-identity / residential-proxy detection | NOT STARTED | Reason codes defined; nothing populates them |
| ML models | NOT STARTED | Deliberate — rules and graph first, models when labels exist |
| Fraud operations console | NOT STARTED | — |
| Thresholds calibrated against real outcomes | NOT STARTED | **All thresholds are policy, not measurement.** Shadow mode required before enforcement |

## Infrastructure

| Item | Status | Evidence |
|---|---|---|
| CI: lint, typecheck, tests, build, invariants | READY | `ci.yml` |
| Deploy gated on green CI for the exact SHA | READY | `check-deploy-gate.mjs` |
| Environment separation | IN PROGRESS | `FURLPAY_ENV` never `NODE_ENV`; separate keys/RPC/treasury not provisioned |
| Queues / DLQ | NOT STARTED | Provider calls are inline |
| Blue/green or canary | NOT STARTED | — |
| Readiness endpoint | READY | `/api/ops/readiness` |

## Observability

| Item | Status | Evidence |
|---|---|---|
| Structured logs + correlation id | READY | `lib/log.ts`, middleware |
| Financial traceability on one row | READY | `0014` carries every reference |
| Secrets never logged | READY | Alert context coerced to scalars |
| Alerting | READY | `raiseAlert`, fingerprinted |
| Metrics / dashboards | IN PROGRESS | `/api/ops/*`; no APM, no tracing export |

## Disaster recovery

| Item | Status |
|---|---|
| Backups / PITR | NOT STARTED |
| RTO / RPO | NOT STARTED |
| Restore drill | NOT STARTED |
| Incident runbook / on-call | NOT STARTED |
| Emergency shutdown | READY — `FURLPAY_MONEY_KILL_SWITCH`, checked before any per-flag setting |

## Testing

| Item | Status | Evidence |
|---|---|---|
| Unit + invariant | READY | 3,289 passing |
| Lifecycle / state machine | READY | 20 tests incl. table integrity |
| Provider contract + failover | READY | 20 tests |
| Ledger invariants | READY | 20 tests incl. randomised trial-balance |
| Asset / peg / concentration | READY | 25 tests |
| Treasury segregation | READY | 10 tests |
| Webhook ordering | READY | 17 tests |
| Concurrency against a real database | NOT STARTED | Needs a test Postgres |
| Failure injection (provider/RPC/KMS/DB down) | NOT STARTED | — |
| End-to-end testnet lifecycle | NOT STARTED | — |

---

## Scores

| Category | Score |
|---|---:|
| Financial correctness | 84 |
| Ledger | 74 |
| Reconciliation | 58 |
| Banking | 12 |
| Cards | 15 |
| Blockchain | 62 |
| Security | 74 |
| Compliance | 20 |
| Infrastructure | 46 |
| Observability | 60 |
| Disaster recovery | 12 |
| Testing | 66 |

**Overall: 49 / 100** (was 47).

The rise is small on purpose. This cycle added asset independence, treasury segregation and the two tracking documents — real correctness work that moves no rail. Banking stays at 12 and compliance at 20 because interfaces are not partners, and a score that rewarded interfaces would be the fake readiness this document exists to prevent.

**The next 10 points are engineering** (chain adapter, chain⇄ledger reconciliation, reorg-aware deposits, DR, failure-injection tests). **The 30 after that are not** — they are a legal opinion, a licence, and signed contracts.
