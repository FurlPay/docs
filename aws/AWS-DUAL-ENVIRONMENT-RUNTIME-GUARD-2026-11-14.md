# AWS dual-environment runtime guard & infrastructure scaffold (14 November 2026)

> **Status: implemented in the application repository, not verified against live AWS.** Nothing is deployed to any AWS account; production is unchanged. Status labels follow the [enterprise architecture](../docs/ENTERPRISE-ARCHITECTURE-2026-2027.md).

## What shipped on 14 November (in `Ashutosh0x/furlpay`, branch `feat/agent-guardrails-clean`)

### 1. The executable environment guard (`apps/web/src/lib/aws/environment.ts`)

`deploy/ENVIRONMENTS.md` made `FURLPAY_ENV` mandatory on AWS as a documented rule. It is now an **executable boot-time boundary** (`assertAwsConfiguration()` / `assertAwsIdentity()`):

| Rule | Enforcement |
|---|---|
| Explicit environment | On any AWS deployment signal (task role, assumed role, expected account) a missing or unknown `FURLPAY_ENV` refuses to boot — the documented silent-`development` outage mode cannot recur. `AWS_DEPLOYMENT_ENV` is an accepted alias; disagreement between the two refuses to boot. |
| Account identity | `assertAwsIdentity()` resolves the real credentials chain, calls **STS GetCallerIdentity**, and fails closed when the account is not `FURLPAY_EXPECTED_AWS_ACCOUNT_ID`. A development workload holding a production role dies at boot. |
| Role allowlists | `AWS_ROLE_ARN` must appear in `FURLPAY_ALLOWED_ROLE_ARNS` for the environment, and the role's account must match the expected account. |
| Namespace-bound resources | Secrets resolve to `furlpay/<environment>/…` and storage to `furlpay-<environment>-…` — a cross-environment read targets a different, nonexistent path. |
| Money movement | `movesRealMoney()` is true **only** for production. Development and staging are simulated and fail-closed by contract; the payment layer consults the guard instead of re-deriving it. |

Every AWS-specific decision in the module carries a `//` comment per the dual-environment integration rules; comments document decisions, the code enforces them.

**Verification:** 26 unit tests pin the rules (selection, alias disagreement, allowlists, account mismatch, namespace traversal rejection, money switch). A live STS assertion suite exists and skips visibly until dedicated `TEST_AWS_*` credentials exist.

### 2. Terraform scaffold (`infrastructure/aws/` — PREPARED FOR APPROVAL, not applied)

Six modules and three environment roots with **separate state per environment**:

- **iam**: task / payment-signer / deployment roles. The task role has no `kms:Sign` and no deployment permissions; the deployment role (CI OIDC federation, no static keys) administers infrastructure but never signs and never reads secret values.
- **kms**: non-exportable Ed25519 `SIGN_VERIFY` key; `kms:Sign` granted **only** to the payment-signer role; the deployment role administers but never signs. Algorithm note: secp256k1 is not a KMS signature algorithm — where it is required, the design is the Nitro Enclave signer described in [AWS-SECURITY-SIGNING-CONTROLS](AWS-SECURITY-SIGNING-CONTROLS.md).
- **secrets**: `furlpay/<environment>/…` namespace scaffold; values are created by the operator, never in Terraform state.
- **monitoring**: validated multi-region CloudTrail, GuardDuty, and alarms mapped to the [CERT-In CIAD-2026-0058](https://www.cert-in.org.in) attack chain — KMS policy changes, IAM changes, new access keys, role assumptions, secrets-retrieval anomalies, S3 public-access changes.
- **networking**: private execution subnets; signing tasks have no internet egress and reach KMS through a VPC endpoint.
- **storage**: per-environment private buckets with public access blocked.

Each environment root refuses to apply into the wrong account (`terraform_data` precondition on `aws_caller_identity` vs `expected_account_id`). Staging is its own account per the [security baseline](../deploy/AWS-SECURITY-BASELINE.md).

### 3. The 13 November Upstash capacity incident — resolved

The production Redis dataset reached its 256 MiB quota (261 MiB used) because 165 `mkt:sec:facts:*` keys cached **raw SEC EDGAR company-facts dumps of 4–6 MiB each** (~251 MiB total). Every kv write failed; quotes, orders, sessions and rate limits all fail closed, which surfaced as "The request could not be completed" on `/orders/new`.

- Root cause found by adding stack logging to the orderflow error envelope (a swallowed `KvError` at `kvSet`).
- The 163 regenerable caches were deleted; writes verified restored.
- Structural fix: the markets cache refuses any value over 256 KiB (`cache.ts` discipline), and the SEC provider caches the **derived** fundamentals (~2 KB) instead of the raw dump.
- Follow-up recorded: rotate Upstash REST tokens (the April 2026 Vercel incident advice).

### 4. Tokenized card payment methods (migrations 0090 + 0091)

Cards are tokenized by Stripe hosted fields — the PAN never reaches FurlPay; storage holds only brand/last4/expiry/billing (migration 0090, service-role-only RLS). `set_default_payment_method` (migration **0091**) performs the default switch atomically with a per-user advisory lock, `FOR UPDATE` ownership enforcement, and execution revoked from PUBLIC/anon. A real-RPC integration suite (no database mocks) runs against a configured test database and skips visibly otherwise.

**Blocked-external:** migrations 0090 + 0091 are **not yet applied to the production database** (verified: the table and function do not exist there). Production card endpoints fail with the real database error until the operator applies them.

## Status matrix

| Control | Status |
|---|---|
| Runtime environment guard (selection, no silent default) | **Implemented, unit-tested** |
| STS account identity assertion | **Implemented; live verification gated on TEST_AWS_* credentials** |
| Role allowlists (dev cannot assume prod) | **Implemented, unit-tested** |
| Namespace-bound secrets/storage | **Implemented, unit-tested** |
| Terraform scaffold (6 modules, 3 roots, account guards) | **Prepared for approval — not applied** |
| Real-AWS integration tests | **Written; skipped visibly until a dedicated TEST account exists** |
| CloudTrail/GuardDuty/alarms | **Prepared for approval — not applied** |
| Upstash capacity incident | **Resolved 13 Nov 2026** (root cause fixed, cache discipline added) |
| Production card migrations (0090/0091) | **Blocked-external — operator must apply via the Supabase SQL editor** |
