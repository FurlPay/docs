# Implementation backlog, October 2026

> Engineering work that follows from the [enterprise architecture](ENTERPRISE-ARCHITECTURE-2026-2027.md), kept separate from the documentation. An item is done when its acceptance check passes in the stated environment, not when code exists, and never because a document was updated. Status as of 22 October 2026. Item IDs are stable; new items are appended.

**Priority:** P0 blocks the AWS cutover or live settlement · P1 required before go-live · P2 follow-up.
**Status:** Open · In progress (code on a branch, not merged) · Implemented and tested (branch, not merged) · Done (merged and verified where stated).

Paths are in the private application repository; `lib/` means `apps/web/src/lib/`.

## A. Move off Vercel and Supabase

| ID | P | Item | Depends on | Affected | Acceptance | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| A1 | P0 | Native PostgreSQL data layer on `pg`, preserving the data-function interface | — | `lib/supabase.ts` → `lib/db.ts`, `lib/db/query.ts`; 73 calling modules | Unit tests against real PostgreSQL; control-plane and API integration suites pass unchanged; full test suite green | In progress |
| A2 | P0 | Supabase Storage → S3 (onboarding documents, research PDFs) | A1 | `lib/settlement/onboardingStore.ts`, `lib/research/storage.ts`, `lib/aws/*` | Upload, read, delete and presign tests; SigV4 matches AWS's published vectors; S3 conditional write refuses an existing key | In progress |
| A3 | P0 | Least-privilege runtime role (`furlpay_app`) and owner role (`furlpay_owner`), with per-table access policies | A4 | New migration; Terraform `DATABASE_URL` | Runtime role cannot run DDL, cannot bypass RLS, cannot update or delete ledger postings | Open |
| A4 | P0 | Bootstrap for a fresh cluster: `NOLOGIN` compatibility roles; `storage.buckets` stub for migration 0038 | — | New `db/aws/0000_bootstrap.sql` | All migrations apply cleanly, twice, on an empty Aurora PostgreSQL 16 cluster | Open |
| A5 | P1 | `TimeZone=UTC` as a startup parameter, not `options` or `SET` | A1 | `lib/db.ts` | Timestamps serialise as `+00:00` directly and through RDS Proxy | Open |
| A6 | P0 | Terraform: enable the AWS data plane per environment; secret list without Supabase; add `DATABASE_CA_CERT` | A1 | `infra/aws/modules/env/{variables,compute}.tf`, `infra/aws/env-contract.json` | `check-env-contract.mjs` passes; staging plan shows cluster, secrets and egress rules (reconciliation T1, T2) | Open |
| A7 | P1 | Confirm the AWS region (Terraform: `eu-west-1`) | — | `infra/aws/envs/*/main.tf` | Decision recorded with latency and data-residency reasoning (T7) | Open decision |
| A8 | P1 | One ElastiCache design: Valkey vs Redis, one cluster vs two | D2 | `infra/aws/modules/env/data.tf`, web runbook | Terraform and runbook agree (T5) | Open |
| A9 | P0 | Schedule all ten jobs; the three GET-only ones accept a signed POST | — | `app/api/cron/{settlement-safety,onchain-reconcile,payroll-schedules}`, `infra/aws/modules/env/scheduler.tf` | Every job returns 200 to a signed POST in staging; dead-letter queue empty for 24 h (T6) | Open |
| A10 | P0 | Staging on AWS and a data-migration rehearsal | A1–A6, A9 | Terraform `envs/staging` | Row counts, primary-key checksums and ledger totals match; `ledger_imbalance` is 0; object checksums match | Open |
| A11 | P0 | Production cutover with Vercel kept warm; passkeys verified on the new origin first | A10, D2 | DNS, CloudFront | Smoke tests pass; rollback rehearsed in staging | Open |
| A12 | P1 | Legal subprocessor list updated when production data moves | A11 | `app/legal/subprocessors` | Published on cutover day | Open |
| A13 | P0 | Database identity: RDS-managed master password; runtime role in `DATABASE_URL`; no password in Terraform state | A3 | `infra/aws/modules/env/data.tf`, `compute.tf` | `terraform show` contains no database password; the app connects as `furlpay_app` (T3) | Open |
| A14 | P1 | Verified database TLS | A1, A6 | `lib/db.ts`, `DATABASE_URL` format | A connection with a wrong CA fails; with the RDS bundle it succeeds; `sslmode` in the URL cannot weaken it (T4) | Open |
| A15 | P2 | RDS Proxy, only if short-lived workers are added or connections near the limit | A5 | Terraform | Load test shows no pinning-induced exhaustion | Open |
| A16 | P1 | S3 buckets in Terraform: documents, research PDFs, evidence (Object Lock) | — | `infra/aws/modules/env` | Block Public Access, SSE-KMS, task-role-only access verified (T10) | Open |

## B. Financial correctness (see the [ledger document](LEDGER-SETTLEMENT-RECONCILIATION.md))

| ID | P | Item | Depends on | Affected | Acceptance | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| B1 | P0 | Payment and audit event in one transaction (G1) | A1 | `lib/paymentsDb.ts` `insertPayment`, `transitionPayment` | Fault injected between the writes leaves neither row | Open |
| B2 | P0 | Fail closed on a missing payments table or function in production (G2) | — | `lib/paymentsDb.ts` `optional()`; boot check | Missing table under `FURLPAY_ENV=production` → 503, no in-memory record | Open |
| B3 | P1 | Lost-update prevention for account persistence (G3) | A1 | `lib/storeDb.ts` `saveAccount`, `lib/store.ts` | Concurrent writers to one account never silently overwrite | Open |
| B4 | P0 | No money through JavaScript `Number` (G4, G5) | A1 | `lib/storeDb.ts` hydration, `lib/paymentsDb.ts` `ledgerBalance` | 18-decimal balances and values above 2^53 minor units round-trip exactly | Open |
| B5 | P1 | Outbox for payment paths other than merchant collection (G6) | A1, F1 | Payment services; new outbox table | Kill after commit, before dispatch: event delivered once after restart | Open |
| B6 | P0 | Replace the legacy wallet-risk service, which returns deterministic demonstration scores, with the address-screening module used by money routes | — | `lib/services/compliance.ts`; callers `app/api/compliance/route.ts`, `lib/services/payments.ts` | No code path returns a risk score not produced by a screening source; tests assert the screening module is called | Open |
| B7 | P1 | Evidence bundles in S3 with Object Lock | A16 | Settlement path | Evidence retrievable per settled payment; deletion refused within retention | Open |
| B8 | P1 | Settlement pauses automatically while `ledger_imbalance` is non-zero | — | Settlement entry points; reconciliation job | With an injected imbalance, settlement refuses and an alert fires | Open |

## C. Solana after Alpenglow (see the [Alpenglow document](SOLANA-ALPENGLOW-FINALITY.md))

| ID | P | Item | Depends on | Affected | Acceptance | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| C1 | P1 | Consensus read from mainnet (feature account and genesis certificate) | — | `lib/chains/solanaConsensus.ts` | Unit tests with the real mainnet account data | Implemented and tested |
| C2 | P1 | Solana in chain-health monitoring and alerts | C1 | `lib/payments/merchant/chainHealth.ts` | A stale Solana head raises an alert; a node without `getAgGenesisCert` is not put in cooldown | Implemented and tested |
| C3 | P1 | Credit latency p50/p95/p99 per chain | — | `lib/payments/merchant/latency.ts`, operations console | Computed from receipts; skewed clocks excluded, not clamped | Implemented and tested |
| C4 | P2 | Balance reads from `confirmed` to `finalized` before `confirmed` is deprecated | — | `lib/onchain/*`, `lib/token/*` | No `confirmed` reads on money paths | Open |
| C5 | P1 | Submission-to-finality timings for FurlPay-submitted transactions | — | Settlement lifecycle | p50/p95/p99 published from production samples | Open |
| C6 | P2 | Any Geyser/Yellowstone consumer buffers per `(slot, bank_id)` and reconciles by blockhash | — | — | Design review before the first line of stream code | Research |
| C7 | P1 | Multi-provider RPC failover configured with at least two production Solana endpoints | — | `SOLANA_RPC_URL` (comma-separated) | Disabling the first endpoint does not stop crediting; each endpoint proves mainnet before use | Open (code IMPLEMENTED; production endpoints not verified) |

## D. Keys, credentials and security

| ID | P | Item | Depends on | Affected | Acceptance | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| D1 | P0 before live settlement | Platform signing keys in KMS (Ed25519, secp256k1) | A11 | x402 facilitator signer; any fee payer | No platform private key outside KMS; every signature in CloudTrail; key policy limited to the task role (T9) | Open |
| D2 | P0 | Passkeys, API keys, OAuth clients and x402 settlement records moved from Redis into PostgreSQL | A1 | `lib/kv.ts` callers | A full Redis flush loses no credential (T11) | Open |
| D3 | P1 | Apply the organization Terraform (CloudTrail with Object Lock, Config, GuardDuty, Security Hub, SCPs) | — | `infra/aws/org` | Security Hub reports the standards enabled | Open |
| D4 | P0 before live settlement | Third-party security assessment | A11 | — | Report received; findings tracked to closure | Open |
| D5 | P1 | Penetration test of the AWS edge (CloudFront, WAF, origin-verify) in staging | A10 | — | Direct-to-ALB requests refused; WAF rules exercised | Open |
| D6 | P1 | Restore drill: point-in-time recovery into a new cluster | A10 | Runbook | Measured recovery time and recovery point recorded in the resilience page | Open |

## E. Agentic payments

| ID | P | Item | Depends on | Affected | Acceptance | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| E1 | P1 | Control-plane tenant-isolation suites run on Aurora in staging, not only PGlite | A10 | `lib/controlPlane/__tests__` | Suites green against the staging cluster | Open |
| E2 | P1 | Alarm on unusual signing volume per agent and per platform key | D1 | Monitoring | A synthetic burst raises an alert | Open |

## F. Asynchronous processing

| ID | P | Item | Depends on | Affected | Acceptance | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| F1 | P2 | SQS-backed outbox dispatch worker on ECS, with a dead-letter queue and alarm | A10 | New worker service; Terraform | Messages survive a worker restart; dead-lettered messages page an operator | Open |

## L. Legal and compliance readiness

| ID | P | Item | Depends on | Affected | Acceptance | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| L1 | P1 | Data-processing terms and subprocessor notice for AWS | A7 | Legal pages | Counsel sign-off recorded | Open |
| L2 | P1 | Evidence retention periods per jurisdiction | — | Evidence bucket policy (B7) | Counsel-provided periods configured | Open |
