# MAINNET READINESS

**Assessed:** 6 September 2026 · **Commit:** working tree on `extension-redesign` @ `63f76e5` + this change set
**Verdict:** **READY FOR TESTNET.** Not ready for private beta, controlled mainnet or public mainnet.

Status values: **READY** (implemented, tested, evidence recorded) · **PARTIAL** (implemented in part, or implemented and unverified) · **BLOCKED** (waiting on something outside the codebase) · **NOT STARTED**.

A line is READY only if someone can point at the evidence. "The code exists" is not evidence that money can move.

---

## 1. Architecture

| Item | Status | Evidence |
|---|---|---|
| Corrected architecture documented | READY | `docs/ARCHITECTURE-CORRECTED.md` — supersedes the root PNG, which draws an Indian UPI rail through a provider that appears nowhere in the codebase |
| Provider abstraction | READY | `lib/providers/types.ts` — canonical on-ramp / off-ramp / banking ports. No `if (provider === …)` anywhere in application code |
| Provider registry, health, selection, failover | READY | `lib/providers/router.ts`; 20 tests in `providers/__tests__/router.spec.ts` |
| Capability matrix | READY (empty, honestly) | `lib/providers/capability.ts` — zero rows, because zero corridors are contracted |
| Environment separation | PARTIAL | `financialEnvironment()` reads `FURLPAY_ENV`, never `NODE_ENV` (`lib/flags.ts:44`). Separate staging/production Supabase projects exist. **Separate RPC, KMS keys, treasury addresses and webhook secrets are not yet provisioned** |

## 2. Financial model

| Item | Status | Evidence |
|---|---|---|
| Money type — integer minor units, no floats | READY | `lib/money/money.ts`; 20 tests incl. exactness beyond `Number.MAX_SAFE_INTEGER` and refusal to add `USDC.arbitrum` to `USDC.base` |
| Chart of accounts | READY | `lib/ledger.ts` `ACCOUNTS` — customer fiat/stablecoin liability, escrow, treasury, provider receivable, in-transit fiat/chain, gas float, operational cash, InCA/OCA, merchant payable, refund liability, chargeback liability, TDS, GST, fee revenue, gas expense, operational expense |
| Customer funds separated from operational | READY | `CUSTOMER_FUND_ASSETS` excludes gas float and operational cash |
| Transaction envelope (11 required attributes) | READY | migration `0014_financial_transactions.sql` — immutable id, idempotency key, ledger id, state, external/provider/chain refs, timestamps, actor, compliance decision, reconciliation status |
| Lifecycle state machine | READY | `lib/money/lifecycle.ts`; 20 tests |
| Evidence required for terminal success | READY | `REQUIRED_EVIDENCE` + `assertTransition`. A provider's HTTP 200 cannot become `COMPLETED` |
| Corrections are compensating entries | READY | `packages/ledger` refuses double-reversal and reversal-of-reversal; SQL enforces it again with a unique index on `reverses` |
| No historical mutation | READY | `ledger_postings` and `financial_transaction_events` are append-only by trigger |

## 3. Banking (India)

| Item | Status | Evidence |
|---|---|---|
| Rail interfaces (UPI / IMPS / NEFT / RTGS) | READY | `lib/rails/india/types.ts` |
| Initiation ≠ acknowledgement ≠ confirmation ≠ settlement | READY | `RailStage`; `isSpendable()` returns true for `settled` only |
| Rail adapter | **BLOCKED** | `indiaRailAdapter()` returns `null`. No partner |
| Payment-aggregator contract | **BLOCKED** | External. RBI (Regulation of Payment Aggregators) Directions, 2025 — customer funds in escrow at a scheduled commercial bank, T+1 merchant settlement, InCA/OCA segregation for cross-border |
| Bank-sponsored payout partner | **BLOCKED** | External |
| Sponsor bank accepting VDA flows, in writing | **BLOCKED** | External |
| Escrow account | **BLOCKED** | External |
| Beneficiary verification (penny drop / name match) | PARTIAL | Interface exists and `unverified` is a refusal condition; no implementation without a partner |
| `BANKING_LIVE` cannot be switched on | READY | `flags.ts` — carries an `unimplemented` reason, checked before `enabled` |

## 4. On-ramp

| Item | Status | Evidence |
|---|---|---|
| Canonical port | READY | `OnrampProviderAdapter` — quote, createOrder, getStatus, cancel, refund, verifyWebhook |
| Quote separate from order | READY | `Quote` carries `expiresAt` and creates no obligation |
| Status is pullable, not webhook-only | READY | `getStatus` is mandatory on every adapter |
| Adapter implementations | **BLOCKED** | None. No contracted provider |
| Existing `/api/onramp/*` routes | PARTIAL | `lib/services/onramp.ts` is a demo quote engine with in-memory sessions and **no INR**. Real hosted leg is a Coinbase CDP session token when credentialed |
| `ONRAMP_LIVE` cannot be switched on | READY | `flags.ts` |

## 5. Off-ramp

| Item | Status | Evidence |
|---|---|---|
| Canonical port | READY | `OfframpProviderAdapter` incl. `notifyDeposit` |
| Chain confirmation does not complete a fiat withdrawal | READY | `lifecycle.ts` OFFRAMP table: `CRYPTO_CONFIRMED` cannot reach `COMPLETED`; it must pass `PROVIDER_PENDING` → `BANK_PENDING`. Tested |
| Provider deposit address screened before sending | PARTIAL | Documented as required on `OfframpOrder.depositAddress`; screening call not yet wired (no adapter to wire it to) |
| Adapter implementations | **BLOCKED** | None |
| Existing `/api/offramp` route | PARTIAL | Quotes from live ECB rates; `execute` returns 503 in production |
| `OFFRAMP_LIVE` cannot be switched on | READY | `flags.ts` |

## 6. Ledger and reconciliation

| Item | Status | Evidence |
|---|---|---|
| Durable double-entry ledger | READY (code) / **BLOCKED** (deployment) | `0009` + `0014`. **Neither migration is confirmed applied to any environment** — `rollout-evidence/` covers `0006` only |
| Production refuses an in-memory ledger | READY | `assertLedgerDurable()` throws `LedgerDurabilityError`; 4 tests |
| Internal reconciliation | READY | `lib/reconciliation.ts` — 7 checks: durability, ledger balance, settled-without-ledger (×2), completed-without-evidence, stale-in-flight, awaiting-review |
| Alerting on discrepancies | READY | `/api/cron/reconcile` raises `raiseAlert` per failing check, fingerprinted per check |
| Bank ⇄ ledger reconciliation | **BLOCKED** | Needs a partner statement API (`getStatement` port exists) |
| Chain ⇄ ledger reconciliation | NOT STARTED | Needs a chain indexer. `lib/reconciliation.ts` states this gap explicitly |
| Negative-balance prohibition | PARTIAL | Enforced per-route at the funds check; not a database constraint |

## 7. Compliance

| Item | Status | Evidence |
|---|---|---|
| KYC tiers and per-tier caps | READY | `lib/kycTiers.ts`, enforced server-side |
| KYC provider webhooks verify the real scheme | READY | Persona (`persona-signature`), Sumsub (`x-payload-digest` + alg), constant-time |
| KYC cannot be demoted by a webhook | READY | `kycTransitionAllowed`; 5 tests |
| India CDD (PAN / OVD / CKYC) | NOT STARTED | Required by FIU-IND AML/CFT Guidelines for VDA reporting entities (08 Jan 2026) |
| Sanctions screening | PARTIAL | OFAC SDN **crypto addresses only**, from a community mirror. No UNSC list, no MHA/UAPA §51A, no PEP, no adverse media |
| Both parties screened | READY | `/api/wallets/transfer` screens payer and destination; `services/payments.ts` already did |
| Wallet risk / blockchain analytics | NOT STARTED | Requires a paid provider. `aml.ts` explicitly declines to fake a score |
| Travel Rule | NOT STARTED | `packages/compliance/src/travelRule.ts` is a pure function that transmits nothing |
| Transaction monitoring | PARTIAL | Velocity, device intelligence, MCC policy. No typology library, no alert queue, no case management |
| EDD triggers (unhosted wallets, mixers, privacy assets) | NOT STARTED | Named as higher-risk by the 2026 guidelines |
| 5-year record retention | PARTIAL | `audit_log` table exists; no retention policy or archival is configured |
| STR filing | NOT STARTED | — |
| FIU-IND registration | **BLOCKED** | External. Activity-based, not location-based |
| Freeze / hold / law-enforcement response | NOT STARTED | No `frozen` state on an account |

## 8. Blockchain

| Item | Status | Evidence |
|---|---|---|
| USDC contract addresses | READY | All 5 in `chainRegistry.ts` verified against Circle's official documentation |
| Base Sepolia / Ethereum Sepolia / Solana addresses | NOT STARTED | Absent from the registry; must be added from Circle, not from a blog |
| x402 / EIP-3009 settlement | READY (code) | `lib/x402Settler.ts`. Verified 2026-07-10 on a **local hardhat fork**, not on public testnet |
| Confirmation depth scaled by amount | READY | `requiredConfirmations()`, enforced by the facilitator |
| Replaced-transaction handling | READY | viem `onReplaced`; the landed hash is what is recorded |
| Dropped-transaction handling | READY | `PENDING_TIMEOUT_MS` → terminal `abandoned` (409), logged at error |
| Reorg handling for settlement | READY | Confirmation-depth gate |
| Reorg handling for deposits | NOT STARTED | — |
| Nonce management / gas top-up | NOT STARTED | — |
| Private RPC with failover | PARTIAL | `rpcTransport()` supports it; readiness check added; **URLs not provisioned** |
| Chain adapter abstraction | NOT STARTED | EVM logic is spread across `x402Settler`, `chainRegistry`, `onchain/balances` |
| Solana | NOT STARTED | Absent from `CHAIN_REGISTRY`. **Safe is EVM-only and cannot cover Solana custody** |

## 9. Smart contracts

| Item | Status | Evidence |
|---|---|---|
| Inventory | READY | 6 contracts: FurlPayRouter, FurlPayEscrow, X402Facilitator, BookingReceipt, USDCPaymaster, ComplianceRouter |
| Deployed anywhere | NOT STARTED | `contractDirectory()` reports all six `pending_deployment` |
| External security audit | **BLOCKED** | Not commissioned. **No mainnet deploy before this completes** |
| Upgrade / admin authority documented | NOT STARTED | Owner, pauser and upgrade authority are not recorded per contract |

## 10. Custody and treasury

| Item | Status | Evidence |
|---|---|---|
| KMS signing boundary | READY | `lib/kms.ts` + `lib/kmsAccount.ts`; Turnkey provider implemented |
| Production refuses a raw env key | READY | `x402Settler.resolveFeePayerAccount()`; 15 tests |
| Settler gated on a capability flag | READY | `ONCHAIN_SETTLEMENT_LIVE` + kill switch |
| Turnkey labelled CUSTODIAL | READY | README and `.env.example` corrected. Turnkey returns a complete signature; the device holds no share |
| Safe deployment | NOT STARTED | `safeAddress` is `sha256("furlpay:safe:"+key)` — no key, no contract. Removed from every receive surface |
| Treasury wallet register | NOT STARTED | No wallet-id → chain → address → purpose → environment → control-policy map |
| Hot / warm / cold separation | NOT STARTED | — |
| Key rotation / recovery | NOT STARTED | Interface only |

## 11. Cards

| Item | Status | Evidence |
|---|---|---|
| Issuing disabled | READY | `CARD_ISSUING_LIVE` with an `unimplemented` reason; 8 routes gated |
| No fake cards or PANs | READY | Issuance returns 503 in production |
| Card abstraction for a future issuer | PARTIAL | Lifecycle service exists; not yet expressed as a provider port |
| Issuer / BIN sponsor / network licence | **BLOCKED** | External |
| PCI DSS scoping | NOT STARTED | — |
| Wallet provisioning | NOT STARTED | UI toggle with an explicit `TODO(backend)`. An Android entitlement is not an implementation |

## 12. Security

| Item | Status | Evidence |
|---|---|---|
| Route auth manifest enforced in CI | READY | 302 routes classified; drift fails the build |
| Body validation + rate limits | READY | `check-route-security.mjs` in CI |
| Webhook signature verification | READY | Per-provider schemes; three impersonating endpoints deleted |
| Per-provider webhook secrets | READY | `FURLPAY_ENDPOINT_SECRET` is outbound-only, documented in `env.ts` |
| Webhook replay + ordering | READY | Redis `SET NX` dedup + per-resource high-water mark |
| Idempotency scoped per principal | READY | `withIdempotency(..., { principal })`; DB unique index in 0014 |
| Failover cannot double-pay | READY | `assertFailoverAllowed` refuses on indeterminate or committed state |
| Penetration test | **BLOCKED** | Not commissioned |
| Transaction simulation before signing | NOT STARTED | `/api/mpc/sign` accepts an opaque digest with advisory context |

## 13. Observability, DR, incident response

| Item | Status | Evidence |
|---|---|---|
| Structured logging | READY | `lib/log.ts`; correlation id in middleware |
| Financial traceability | READY | 0014 carries transaction id, ledger id, provider ref, bank ref, chain hash on one row |
| Secrets never logged | READY | Alert context coerced to scalars; KMS returns signatures, never keys |
| Metrics | PARTIAL | `/api/ops/*` and the ops event ledger. No APM, no tracing export |
| Alerting | READY | `raiseAlert` with fingerprinting; reconciliation wired |
| Backups / PITR | NOT STARTED | Undocumented |
| RTO / RPO | NOT STARTED | Never stated |
| Incident runbook / on-call | NOT STARTED | `SECURITY.md` covers vulnerability disclosure only |
| Emergency shutdown | READY | `FURLPAY_MONEY_KILL_SWITCH` + per-rail pause |

## 14. Deployment

| Item | Status | Evidence |
|---|---|---|
| CI: lint, typecheck, tests, build, security invariants | READY | `.github/workflows/ci.yml` |
| Deploy gated on a green CI run for the exact SHA | READY | `npm run deploy:prod` → `check-deploy-gate.mjs` |
| Rollback | PARTIAL | Vercel instant rollback; procedure undocumented |
| Blue/green or canary | NOT STARTED | — |
| Production readiness endpoint | READY | `/api/ops/readiness` — admin-only, blocking vs non-blocking, names the missing credential |

---

## Blockers, by who can clear them

**Engineering can clear these:** chain adapter abstraction; Solana custody design; treasury wallet register; chain⇄ledger reconciliation; reorg-aware deposit crediting; nonce/gas management; transaction simulation before signing; account freeze state; retention policy; DR runbook, RTO/RPO and a restore drill; testnet contract deployment; public-testnet x402 run.

**Only an external party can clear these:**

| Blocker | Who |
|---|---|
| Payment-aggregator contract with explicit VDA-merchant approval | RBI-authorised PA — **the hardest item, and not technical.** Most Indian PAs decline VDA merchants. Budget 6–12 months and expect refusals |
| Bank-sponsored payout partner | Payout partner + sponsor bank |
| Escrow account under the PA Directions, 2025 | Scheduled commercial bank |
| FIU-IND registration as a VDA reporting entity | FIU-IND |
| Legal opinion on regulatory classification | Counsel — **start this first; it determines everything above** |
| Blockchain analytics | Chainalysis / TRM / Elliptic class |
| Travel Rule transmission | Notabene / TRUST class |
| Card issuer, BIN sponsor, network licence | Issuer + sponsor |
| Smart-contract audit | External auditor |
| Penetration test | External firm |
| AML officer, compliance analysts, dispute and reconciliation ops | Hiring |
| Crime / cyber insurance | Insurer |

## Deployment procedure

```
1. npm run deploy:prod              # refuses unless CI is green for this SHA
2. GET /api/ops/readiness           # admin; must show no blocking failures
3. Confirm every money flag is OFF unless its dependency is genuinely ready
```

## Rollback procedure

```
1. FURLPAY_MONEY_KILL_SWITCH=1      # stops every money capability at once
2. Vercel instant rollback to the previous deployment
3. GET /api/ops/reconciliation      # confirm the books are unchanged
```

## Emergency shutdown

```
FURLPAY_MONEY_KILL_SWITCH=1
```
Checked before any per-flag setting, so nothing can defeat it. Per-rail pause via `lib/ops/controls.ts` for a narrower stop.

---

## Verdict

**READY FOR TESTNET.**

The financial core is correct: money is exact, the books are double-entry and durable-or-refuse, the lifecycle refuses to call an HTTP 200 a settlement, provider selection cannot pick an uncontracted corridor, and failover cannot double-pay. That is a genuine foundation.

It is not ready for private beta, because private beta means real customers and real money, and there is no rail to move it on: no banking partner, no on-ramp provider, no off-ramp provider, no deployed contract, no FIU-IND registration, no audit and no pentest. Those are not code.

**The one thing to do first is the legal opinion.** It determines whether FurlPay is a custodian, an exchange, an aggregator, or none of those — and every partner conversation, every licence application and half the remaining architecture depends on that answer.
