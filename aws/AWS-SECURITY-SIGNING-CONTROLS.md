# AWS Security, Cryptographic Signing and Enterprise Controls

> **Status, 22 October 2026.** Application-level controls are **IMPLEMENTED** and run on furlpay.com today. Every AWS control below is **PLANNED**: it is defined in Terraform in the application repository, which has **not been applied**. **FurlPay holds no PCI DSS or SOC 2 certification and has had no third-party security audit.** Both are goals. Status labels are defined in the [enterprise architecture](../docs/ENTERPRISE-ARCHITECTURE-2026-2027.md). The account-level baseline and go-live checklist live in [AWS Security Baseline](../deploy/AWS-SECURITY-BASELINE.md); this page covers signing and payment controls.

---

## 1. Controls by layer

| Layer | Control | Status |
| :--- | :--- | :--- |
| Organization | Organization-wide CloudTrail to an S3 bucket with **Object Lock**; AWS Config recorder; GuardDuty and Security Hub with a delegated administrator; service control policies | **PLANNED** (Terraform `infra/aws/org`) |
| Network | VPC across three Availability Zones; app and database in private subnets; VPC endpoints; VPC flow logs | **PLANNED** |
| Edge | CloudFront → AWS WAF → ALB. The ALB accepts only CloudFront's origin-facing prefix list **and** a secret origin-verify header; anything else gets a fixed 404 | **PLANNED** |
| Identity | IAM roles per ECS task; GitHub Actions deploys through OIDC (no long-lived keys); IAM Identity Center for people, no IAM users | **PLANNED** |
| Secrets | AWS Secrets Manager, injected into tasks at start; nothing in images or task definitions | **PLANNED** |
| Encryption | A customer-managed KMS key for the database, queues, logs and buckets; S3 buckets private with Block Public Access | **PLANNED** |
| Application | Schema validation and a 100 KB cap on every mutating request; per-route rate limits on public routes; HMAC verification over the raw body with timestamp checks on webhooks; secrets never returned in responses | **IMPLEMENTED** |
| Authentication | Passkeys (WebAuthn, relying-party ID from configuration, never from the Host header); step-up verification for sensitive actions | **IMPLEMENTED** |

---

## 2. Signing keys

| Key | Today | Target | Status |
| :--- | :--- | :--- | :--- |
| End-user wallet keys | On the user's device (Android Keystore, Apple Secure Enclave) | Unchanged: FurlPay never holds them | **IMPLEMENTED** |
| Server-side MPC wallets | A signing route co-signs with Turnkey (MPC). Policy is enforced at the point of signing: agent spend policy, plus a single-use authorization bound to the device and challenge | Unchanged | **IMPLEMENTED**; production usage not verified |
| Platform keys: x402 facilitator signer (EVM); Solana fee payer | The EVM facilitator key is an environment secret (`FACILITATOR_EVM_PRIVATE_KEY`). No Solana fee-payer key is configured in production. No key is held in KMS today | **AWS KMS asymmetric keys**: `ECC_NIST_EDWARDS25519` (Ed25519, supported since 7 November 2025) for Solana; `ECC_SECG_P256K1` for EVM. The private key never leaves KMS; each signature is an auditable `Sign` call allowed only to the task role | **PLANNED** |
| Policy-bound signing in an isolated environment | — | Nitro Enclaves with attestation-conditioned KMS key policies | **RESEARCH** |
| Dedicated HSM | — | AWS CloudHSM. Not required while KMS meets the need; reconsidered only for a regulatory requirement for single-tenant HSMs | **RESEARCH** |
| Smart-contract accounts (e.g. Safe) | — | Under evaluation for business treasuries with multi-party approval | **RESEARCH** |
| Other MPC custody providers | Interface only | Evaluated per partner requirement | **RESEARCH** |

**What KMS guarantees, and what it does not.**
- KMS asymmetric private keys are generated inside KMS's HSMs and cannot be exported in plaintext. Every use is a `Sign` call authorized by the key policy and IAM, and logged in CloudTrail.
- KMS does **not** decide *whether* a payment should be signed. That is FurlPay's policy layer (§3). A compromised task role could still ask KMS to sign; the controls are a narrow key policy, the authorization checks before the call, spending limits, and alarms on unusual `Sign` volume.
- Moving a platform key into KMS changes how that key is protected. It does **not** change custody: funds a platform key controls are still controlled by FurlPay. End-user funds are non-custodial only where the user's device holds the key.
- **Nitro Enclaves (RESEARCH)** add a further control: a KMS key policy can require a signed enclave attestation (`kms:RecipientAttestation` condition keys), so only a measured enclave image can use the key.

**Separation and revocation.**
- One key per purpose (fee payer, facilitator), never a treasury key on a hot path. Each key holds only the float its function needs.
- Revoking a signer means disabling the KMS key or removing the task role from its policy, both effective immediately and both logged. Agent signers are revoked in the control plane: a revoked mandate is refused before limits are even evaluated.

**Rotation and recovery.**
- KMS asymmetric keys cannot be rotated automatically. Rotation means creating a new key, moving funds or authority to the new address, and disabling the old key after a waiting period.
- Every platform key address is listed in operator documentation with its purpose and limit.
- A compromised platform key is handled by moving authority to a new key, never by reusing the address.
- Platform hot keys hold only what their function needs (fees, not treasury).

---

## 3. Payment authorization controls

| Control | What it does | Status |
| :--- | :--- | :--- |
| Agent identity and delegation (KYA) | Each agent credential resolves to an organization and a delegation chain; requests outside the chain are refused | **IMPLEMENTED, TESTED** |
| Mandates and spending policy | Daily limits and each mandate's cumulative window, in exact integer amounts, evaluated in a fixed order (liveness, limits, risk, approval) before signing; an unreadable exposure figure is a refusal, never zero | **IMPLEMENTED, TESTED** |
| Recipient attestation | Recipients are attested before they can be paid; re-attestation cannot clear a revocation | **IMPLEMENTED, TESTED** |
| Human approval | Payments outside policy wait for an approver, with step-up; decisions are recorded | **IMPLEMENTED, TESTED** |
| Sanctions and AML screening | Money routes screen addresses against the OFAC SDN list (local, always), add Circle's Compliance Engine where configured and Arc's USDC blocklist on Arc; the stricter verdict wins and an unavailable screen is a refusal. Payments from screened senders are held for review. Case updates from a screening provider arrive through a signed webhook | **IMPLEMENTED** |
| Transaction simulation | Transactions are decoded and checked before signing, within the documented scope of the transaction-security module | **IMPLEMENTED**; see [Transaction Security](../docs/TRANSACTION_SECURITY.md) for exactly what it covers |
| Kill switch | Feature flags turn off each money rail independently; card issuing, banking and ramps are off today | **IMPLEMENTED** |

---

## 4. Monitoring and audit

- **Application:** reconciliation alerts for ledger defects, chain-head staleness and RPC failover state, held payments and refund state (**IMPLEMENTED**).
- **AWS:** CloudWatch alarms for the ALB, ECS and database to an SNS topic; GuardDuty; WAF metrics; VPC flow logs; CloudTrail with Object Lock for API audit (**PLANNED**).
- **Evidence retention:** payment evidence bundles in an S3 bucket with Object Lock (**PLANNED**). The retention period will be set from legal advice for each operating jurisdiction, not from a default.

---

## 5. Compliance position

| Framework | Position |
| :--- | :--- |
| PCI DSS | **Goal.** Card data is out of scope by design: the planned card program uses a partner issuer's hosted or tokenized components. No assessment has been performed. |
| SOC 2 Type II | **Goal.** Controls are being mapped; no audit has started. |
| Third-party security audit | **None yet.** Reports under `audit/` in this repository are internal reviews. |

---

## Sources

- [AWS KMS EdDSA support, 7 November 2025](https://aws.amazon.com/about-aws/whats-new/2025/11/aws-kms-edwards-curve-digital-signature-algorithm/)
- AWS documentation: KMS asymmetric keys; CloudFront origin-facing managed prefix list; S3 Object Lock.
