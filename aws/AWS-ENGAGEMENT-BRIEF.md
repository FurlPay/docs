# AWS Engagement Brief: Current State, Pain Points and Asks

> **Status, 15 October 2026.** FurlPay production runs on Vercel, Supabase
> Postgres and Upstash Redis. The AWS account is set up, but no production
> workloads run on it yet, so there is no meaningful AWS bill. Migrating to AWS
> is the plan, and this brief lists what that migration has to solve.

**Audience:** the AWS startup and solutions-architect team working with FurlPay.
**Companion documents:** [Environments](../deploy/ENVIRONMENTS.md) ·
[Web: Vercel → AWS](../deploy/WEB-AWS.md) ·
[AWS Security Baseline](../deploy/AWS-SECURITY-BASELINE.md) ·
[1-Year Migration Blueprint](FURLPAY_1_YEAR_AWS_INFRASTRUCTURE_REPORT.md) ·
[TOKEN2049 Engagement Log](TOKEN2049-AWS-ENGAGEMENT-LOG.md)

---

## 1. What FurlPay is

FurlPay is a stablecoin payments platform, built around USDC, with a travel
focus. It ships five surfaces from one codebase: a web app (which is also the
API), Android, iOS, Wear OS and watchOS. The web app is Next.js 15 with
**446 route handlers and 269 pages** (15 October 2026).

What is live and what is not is tracked in the
[verified-claims register](../docs/VERIFIED-CLAIMS.md). In short:

- **Card issuing is designed but switched off.** No card issuer is contracted.
- Banking, on-ramp and off-ramp are switched off. No provider is contracted.
- The iOS app is blocked on an Apple Developer account.
- FurlPay has had internal security reviews, but no third-party audit yet.

---

## 2. Where it runs today

| Concern | Today | Notes |
| :--- | :--- | :--- |
| Web app and API | Vercel | Edge middleware handles auth, CSRF and rate limiting |
| Relational data | Supabase Postgres | Reached through PostgREST over HTTP, not a Postgres driver |
| Key-value data | Upstash Redis (REST) | Also called from edge middleware |
| Scheduled jobs | Five Vercel cron jobs | Listed in [Environments](../deploy/ENVIRONMENTS.md) |
| AI features | Claude Haiku 4.5 via the Anthropic API | Copilot and customer support |
| Solana access | Third-party RPC providers | No dedicated nodes |
| CI | GitHub Actions | Currently blocked by an account billing issue |

---

## 3. The pain points

### 3.1 Moving the web app and API off Vercel

The target is ECS Fargate behind CloudFront and an ALB
([runbook](../deploy/WEB-AWS.md)). The hard parts:

- **Edge middleware** carries authentication, CSRF checks and rate limiting.
  It needs an equivalent that runs before the app on AWS.
- **Scheduled jobs.** Our cron routes accept GET. EventBridge API destinations
  send POST with a five-second timeout, so a slow job is retried and can run
  twice. The routes need POST support and idempotency before cutover.
- **Passkeys.** WebAuthn credentials are bound to the relying-party ID.
  The cutover must keep `furlpay.com` as the RP ID, or every user's passkey
  stops working.
- A zero-downtime cutover with a tested rollback.

**We want:** a reference architecture review and a cutover plan.

### 3.2 The database is reached through PostgREST

The app has no Postgres driver. Every query goes through PostgREST over HTTP.
The schema uses `citext`, `postgis` and `pg_trgm`, and relies on Supabase's
`service_role`, `anon` and `authenticated` roles. Aurora PostgreSQL is
therefore not a drop-in replacement. It needs either a self-hosted PostgREST
layer or a rewrite of the data-access layer.

**We want:** a recommendation between keeping Supabase for phase 1, running
PostgREST on AWS in front of Aurora or RDS, or rewriting the data layer. We
also want a migration path for the data itself.

### 3.3 Key-value data is a system of record

Upstash Redis holds passkeys, API keys, OAuth clients and agent-payment
settlement records. It is not only a cache, and losing it would log users out.
It is reached over REST, including from edge middleware. In October 2026 a
provider quota limit caused an outage. ElastiCache speaks the Redis TCP
protocol, not REST, so it is not a drop-in replacement either.

**We want:** a durable design (DynamoDB, ElastiCache with persistence, or a
split between the two) and a migration that keeps sessions and passkeys valid.

### 3.4 Keys and secrets

Signing keys and webhook secrets live in platform environment variables today.
The blueprint targets KMS envelope encryption and Nitro Enclaves for signing.

**We want:** a review of the key-custody design, covering KMS, CloudHSM and
Nitro Enclaves, and of how the application receives secrets.

### 3.5 Solana RPC

FurlPay depends on third-party Solana RPC providers. The blueprint proposes
dedicated Agave nodes on storage-optimised EC2 instances.

**We want:** a sizing and cost check of dedicated nodes against managed
providers, before we commit to running them.

### 3.6 AI workloads

Copilot and support run on Claude Haiku 4.5 through the Anthropic API.

**We want:** to move them to Amazon Bedrock. That would give us one bill,
the option of using credits, and Bedrock guardrails.

### 3.7 Security and compliance

The [security baseline](../deploy/AWS-SECURITY-BASELINE.md) plans a
multi-account organisation, IAM Identity Center with no IAM users, GuardDuty,
Security Hub and WAF. Card issuing, once an issuer is contracted, brings
PCI DSS scope. FurlPay holds no PCI DSS or SOC 2 certification today.

**We want:** a review of the account structure and of how to keep PCI DSS
scope as small as possible from the start.

### 3.8 CI/CD

GitHub Actions is currently blocked by an account billing issue, so
deployments run without CI.

**We want:** guidance on CodeBuild or CodePipeline with OIDC federation, or on
keeping GitHub Actions with OIDC into AWS.

### 3.9 Cost

There is no AWS spend yet. The [cost model](FURLPAY_AWS_COST_OPTIMIZATION_REPORT.md)
estimates the planned architecture at roughly $21,500 to $44,000 a year,
depending on choices. That is a model, not a bill.

**We want:** a realistic cost review and support to cover the migration period.

---

## 4. What we are asking AWS for

1. A solutions-architect review of this plan, in Well-Architected format.
2. Guidance on AWS Activate credits. FurlPay is not yet affiliated with an
   Activate Provider. Is there another route?
3. Whether migration funding applies to a move from Vercel to AWS.
4. Access to Claude on Amazon Bedrock, and confirmation that credits apply.
5. A named contact for follow-up architecture questions during the migration.

---

## 5. Active AWS relationship

On 7 October 2026 FurlPay attended **The Agentic Mixer: Build, Fund, Scale**,
a TOKEN2049 Singapore side event co-hosted by AWS Startups and Animoca Brands.
A working relationship was established with Cyrus Fang (`fangzy@amazon.com`),
AWS Startups BD in Singapore. Discussions about Activate credits, Bedrock
access and go-to-market support are active.

Full timeline and call-preparation notes: [TOKEN2049 Engagement Log](TOKEN2049-AWS-ENGAGEMENT-LOG.md).

---

*Prepared 15 October 2026. Facts in this brief are checked against the
FurlPay codebase and the [verified-claims register](../docs/VERIFIED-CLAIMS.md).*
