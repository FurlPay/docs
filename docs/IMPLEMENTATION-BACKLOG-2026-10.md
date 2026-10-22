# Implementation backlog, October 2026

> Engineering work that follows from the [enterprise architecture](ENTERPRISE-ARCHITECTURE-2026-2027.md), kept separate from the documentation. Each item is done when its acceptance check passes, not when code exists. Status as of 22 October 2026.

## A. Move off Vercel and Supabase

| # | Item | Acceptance check | Status |
| :--- | :--- | :--- | :--- |
| A1 | PostgreSQL data layer on the `pg` driver, preserving the existing data-function interface | All data call sites and integration suites pass against real PostgreSQL | **In progress** (feature branch) |
| A2 | Supabase Storage → S3 for onboarding documents and research PDFs | Upload, read, delete and presigned-URL tests; SigV4 matches AWS's published vectors | **In progress** (feature branch) |
| A3 | Run as a least-privilege `furlpay_app` role; owner role used only by migrations; per-table access policies for that role | The app role cannot run DDL, cannot read with RLS bypassed, cannot update or delete ledger postings | Planned |
| A4 | Bootstrap script for a fresh Aurora cluster (`NOLOGIN` compatibility roles; `storage.buckets` stub for migration 0038) | All 58 migrations apply cleanly, twice, on an empty cluster | Planned |
| A5 | Pass `TimeZone=UTC` as a startup parameter rather than through `options` (RDS Proxy compatibility) | Timestamps serialise as `+00:00` through RDS Proxy | Planned |
| A6 | Production Terraform values: `multi_az = true`, `deletion_protection = true`, backup retention set, region chosen | `terraform plan` for prod shows them; reviewed | Planned |
| A7 | Choose the AWS region | Decision recorded with latency and data-residency reasoning (Terraform default is `eu-west-1`; Vercel functions run in `iad1`) | Open decision |
| A8 | Reconcile the ElastiCache engine (Terraform: Redis 7.1) with the runbook (Valkey, two clusters) | One documented choice, reflected in Terraform | Planned |
| A9 | Scheduled jobs accept the POST requests EventBridge API destinations send | Every scheduled route answers 200 to a signed POST in staging | Planned |
| A10 | Staging on AWS; data-migration rehearsal | Row counts, checksums and ledger totals match; `ledger_imbalance` reports 0 | Planned |
| A11 | Cutover with Vercel kept warm for rollback; passkeys verified on the new origin first | Production smoke tests pass; rollback rehearsed | Planned |
| A12 | Legal subprocessor list updated when production data moves to AWS | Published on the day of cutover | Planned |

## B. Financial correctness (see the [ledger document](LEDGER-SETTLEMENT-RECONCILIATION.md))

| # | Item | Acceptance check | Status |
| :--- | :--- | :--- | :--- |
| B1 | Payment creation and status changes write the payment and its audit event in one transaction (G1) | A fault injected between the two writes leaves neither | Planned |
| B2 | Production fails closed when a payments table or function is missing (G2) | A missing migration yields 503, never in-memory state | Planned |
| B3 | Account persistence: targeted updates or a version column (G3) | Two concurrent writers to different fields of one account both persist | Planned |
| B4 | Money never read through JavaScript `Number` (G4, G5) | Values beyond 2^53 minor units and 18-decimal balances round-trip exactly | Planned |
| B5 | Outbox for payment paths other than merchant collection (G6) | Killing the process after commit never loses the event | Planned |
| B6 | Replace the legacy wallet-risk service, which returns deterministic demonstration scores, with the address-screening module used by money routes (OFAC SDN, plus Circle Compliance where configured), or remove its callers | No code path returns a risk score that was not produced by a screening source | Planned |
| B7 | Evidence bundles written to an S3 bucket with Object Lock | Evidence for each settled payment can be retrieved and cannot be deleted within retention | Planned |

## C. Solana after Alpenglow (see the [Alpenglow document](SOLANA-ALPENGLOW-FINALITY.md))

| # | Item | Acceptance check | Status |
| :--- | :--- | :--- | :--- |
| C1 | Consensus read from mainnet (feature account and genesis certificate) | Unit tests with the real mainnet account data | **Implemented, tested** (feature branch, not merged) |
| C2 | Solana in chain-health monitoring and alerts | A stale Solana head raises an alert | **Implemented, tested** (feature branch, not merged) |
| C3 | Credit latency p50/p95/p99 per chain in the operations console | Values computed from receipts; skewed clocks excluded, not clamped | **Implemented, tested** (feature branch, not merged) |
| C4 | Move balance reads from `confirmed` to `finalized` before `confirmed` is deprecated | No `confirmed` reads remain on money paths | Planned |
| C5 | Submission-to-finality timings from FurlPay-submitted transactions | p50/p95/p99 published from production samples | Planned |
| C6 | Any future Geyser/Yellowstone consumer buffers per `(slot, bank_id)` and reconciles by blockhash | Design review before the first line of stream code | Research |

## D. Keys and security

| # | Item | Acceptance check | Status |
| :--- | :--- | :--- | :--- |
| D1 | Platform signing keys (Solana fee payer, x402 facilitator) in AWS KMS: Ed25519 and secp256k1 | No platform private key exists outside KMS; every signature appears in CloudTrail | Planned |
| D2 | Passkeys, API keys, OAuth clients and x402 settlement records moved from Redis into PostgreSQL | A Redis flush loses no credential | Planned |
| D3 | Apply the organization Terraform (CloudTrail with Object Lock, Config, GuardDuty, Security Hub, SCPs) | Security Hub reports the standards enabled | Planned |
| D4 | Third-party security assessment before live settlement | Report received; findings tracked | Planned |
