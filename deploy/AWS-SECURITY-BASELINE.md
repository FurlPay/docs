# AWS security baseline

> Scope: the controls that must exist **before** FurlPay moves money on AWS.
> The longer architectural treatment is the
> [Enterprise Security & Compliance Blueprint](../aws/FURLPAY_AWS_ENTERPRISE_SECURITY_AND_COMPLIANCE_BLUEPRINT.md);
> this page is the shorter list of things that block a go-live.
>
> This repository is **public**. Nothing here names an account id, ARN, role
> name, VPC id, bucket, endpoint or key alias. Those belong in the private
> infrastructure repository. A security baseline is safe to publish; an
> inventory of your control plane is not.

---

## 1. Account topology

Three accounts minimum, under one Organization:

```
Organization root
├── Security       CloudTrail org trail, Config, GuardDuty, Security Hub
├── Staging        pre-production, no real money, no production data
└── Production     the only account with live funds
```

The boundary between staging and production is an **account boundary enforced
by SCPs**, not a tag or a naming convention. That is what makes "a staging
credential cannot touch production" a property of the platform rather than a
promise in a runbook.

**Service Control Policies to attach at the OU level.** SCPs set the maximum
permission boundary: even an account-local administrator with
`AdministratorAccess` cannot exceed them.

- Deny disabling or deleting CloudTrail, Config or GuardDuty
- Deny `iam:CreateUser` and `iam:CreateAccessKey` — humans get Identity Center, workloads get roles
- Deny operating outside the approved regions
- Deny deleting KMS keys or shortening their deletion window
- Deny disabling S3 Block Public Access

---

## 2. Identity

**No IAM users.** People authenticate through **IAM Identity Center** (SSO),
federated to your identity provider. Workloads assume roles. Access keys are
the credential most likely to end up in a repository, a CI log or a laptop
backup, and the only reliable way to stop that is to have none to leak.

**The root user** of every account: hardware MFA (FIDO2/WebAuthn — a security
key, not a TOTP app, because TOTP is phishable), no access keys, and a
break-glass procedure that is audited when used.

**Least privilege is measured, not asserted.** IAM Access Analyzer reports
unused access — roles and permissions that have not been exercised — and
generates policies from observed CloudTrail activity. Run it on a schedule and
act on the findings; a policy nobody has ever needed is a policy nobody should
have.

**Policy checks belong in CI.** Access Analyzer's policy validation runs
against Terraform plans before apply, so an over-broad grant is rejected at
review time rather than discovered in an audit.

---

## 3. Secrets

Every secret in **AWS Secrets Manager**, injected at task start. Not in task
definition environment variables (they are visible to anyone with
`ecs:DescribeTaskDefinition`), not in the image, not in SSM plaintext
parameters.

Rotate on a schedule. The ones that matter most here are the session signing
secret, the database credentials, and the third-party API keys that can move
money — Stripe Issuing, Circle, the relayer.

> A note from operational history: the Discord bot token lives in three places
> that must move together, and a rotation that updates two of them fails in a
> way that looks like an unrelated outage. When you migrate a secret, migrate
> every copy, and write down where the copies are.

---

## 4. Key management

Customer-managed KMS keys, one per data class, with key policies that name the
roles allowed to use them. Not the AWS-managed defaults — those cannot express
"only the settlement task role may decrypt settlement data".

**Relayer signing keys do not belong in Secrets Manager.** A key that signs
on-chain transactions is not a credential you can rotate after a compromise —
the funds are already gone. These belong in **Nitro Enclaves**, with KMS key
policies gated on PCR attestation, so the key is usable only by an
attested enclave image and never appears in the parent instance's memory.
This is already the design in the
[payment flow architecture](../aws/FURLPAY_USDC_PAYMENT_FLOW_AND_AWS_ARCHITECTURE.md).

Encrypt at rest everywhere (Aurora, S3, EBS, ElastiCache) and in transit
everywhere (TLS 1.2 minimum, 1.3 preferred).

---

## 5. Network

- Application tasks in **private subnets**. No public IPs.
- Egress through NAT with an allow-list where practical — a payments backend
  has a short list of hosts it legitimately calls.
- **VPC endpoints** for S3, Secrets Manager, KMS and ECR so that traffic never
  leaves the AWS network.
- ALB terminates TLS; **AWS WAF** in front with the managed core rule set plus
  rate-based rules.
- Security groups reference other security groups, not CIDR blocks.

---

## 6. Detection

| Control | Purpose |
|---|---|
| CloudTrail (org trail) | every API call, to the Security account, immutable |
| GuardDuty | threat detection incl. malicious IP / crypto-mining signatures |
| Security Hub | aggregates findings against CIS + AWS Foundational |
| AWS Config | drift and conformance packs |
| CloudWatch alarms | see below |

Logs go to the **Security account**, which the production account cannot
write over. An attacker with production access must not be able to erase the
record of what they did.

### Alarms that matter for this product

Generic infra alarms are table stakes. These are the FurlPay-specific ones,
each of which corresponds to something that has actually gone wrong:

- **Auth KV capacity** — at 70% of the limit. A full cache took down passkey
  sign-in once already; see [Environments §5](ENVIRONMENTS.md).
- **Reconciliation job outcome** — not just "did it run" but "did it balance".
- **Cron 5xx rate** — the scheduled jobs failed silently in production for
  want of a secret.
- **Passkey assertion failure rate** — a spike means an RP ID or origin
  misconfiguration, which is invisible to uptime checks because the page loads
  fine.
- **Webhook signature rejections** — a spike is either a rotated secret or an
  attack.

---

## 7. Supply chain

The repository currently reports **90 Dependabot advisories on the default
branch (2 critical, 49 high)**. That is a pre-existing condition, not a
migration artefact, but a payments backend should not carry two criticals into
a production launch.

- Resolve the criticals and highs before go-live; record accepted risks with
  reasons in `.github/audit-allowlist.json`, which this repo already uses.
- Pin base images **by digest**, not by tag. `:3.12-slim` moves.
- Scan images in ECR on push; fail the pipeline on critical findings.
- Sign build artefacts and verify signatures at deploy.
- Never `npm audit fix --force` — it has broken this codebase's lockfile before.

---

## 8. Data

- Aurora with **deletion protection**, automated backups, and PITR.
- Backups encrypted with a customer-managed key, copied cross-region.
- **Restore is tested**, not assumed. An untested backup is a hypothesis.
- PII classified and access-logged; KYC artefacts (Persona) treated as the
  most sensitive class in the system.
- Retention policies written down, because "keep everything forever" is itself
  a compliance risk.

---

## 9. Go-live gate

Do not move real money until every line is true:

- [ ] Three accounts, SCPs attached and tested (try to violate one)
- [ ] Zero IAM users; Identity Center federated; root keys absent, hardware MFA on
- [ ] All secrets in Secrets Manager; none in task definitions or images
- [ ] Relayer keys in Nitro Enclaves with attestation-gated KMS policies
- [ ] Private subnets, WAF, VPC endpoints for S3/KMS/Secrets Manager/ECR
- [ ] CloudTrail org trail → Security account; GuardDuty + Security Hub on
- [ ] Product-specific alarms from §6 firing into a paged channel
- [ ] Critical and high advisories resolved or explicitly accepted
- [ ] Backup **restore** exercised end-to-end
- [ ] Access Analyzer clean, or findings triaged with owners

---

## Sources

- [Security best practices in IAM — AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Organizing Your AWS Environment Using Multiple Accounts — AWS whitepaper](https://docs.aws.amazon.com/pdfs/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.pdf)
- [Well-Architected best practices for software supply chain security — AWS Security Blog](https://aws.amazon.com/blogs/security/well-architected-best-practices-for-software-supply-chain-security/)
- [AWS multi-account best practices](https://towardsthecloud.com/blog/aws-multi-account-best-practices)
