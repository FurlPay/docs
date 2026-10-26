# Review of the "AWS Architecture, Documentation & Cloud Infrastructure Audit: P1 to Pn", October 2026

> **What this is.** On 26 October 2026, an internally circulated audit (22 findings, FIX-AWS-001 to 022) and its companion "remaining percentages" roadmap proposed infrastructure changes for the AWS migration. Both were checked against the application repository, the never-applied Terraform and vendor documentation.
>
> **The reports themselves are not published here.** They were written against an older working copy of the code, so many of their findings describe problems already fixed. This page records what was kept, what was fixed as a result, and what was corrected or rejected.
>
> **Nothing on this page is deployed.** Production still runs on Vercel, Supabase and Upstash.

## Already in place

The reports described these as missing. They exist in the AWS branch of the application repository and are covered by plan-level tests:
- three Availability Zones, selected by zone ID;
- ElastiCache for Valkey 8.0;
- RDS Proxy with TLS required and IAM authentication;
- middleware on the Node.js runtime, so it can reach Valkey over TCP;
- hedged read-only RPC across providers, with broadcasts never hedged;
- a durable transactional outbox, drained through a scheduled SQS queue with a dead-letter queue.

## Fixed as a result of the review (in code, not applied)

| Finding | Change |
|---|---|
| One encryption key for every data domain | Three keys: data at rest; Secrets Manager; logs and alarm notifications. The review also found that the alarms topic and log group used a key whose policy admitted no AWS service, so alarms and security notifications **would not have been delivered**. The new logs key admits only the services that publish to it. |
| Disaster-recovery drill never run | A read-only drill script now runs the schema and ledger-balance checks against a scratch restore and refuses production. It records PASS only from observed results within the one-hour recovery target. **The drill itself has still not been run.** |
| No zonal failover on the load balancer | AWS zonal shift is enabled on the load balancer, so one command takes an impaired zone out of service. Zonal *autoshift* is documented as a manual step until the Terraform provider supports it. |

## Corrected or rejected

| Report said | Finding |
|---|---|
| The recovery runbook's Seoul (ap-northeast-2) region is a stale leftover; the database is in Ireland | **Wrong.** Production's Supabase database is in Seoul. Ireland (eu-west-1) is the AWS *target*, which is not live. The runbook now lists both. Acting on the report would have directed a restore to the wrong region. |
| Spread payment functions across three European Vercel regions | Automatic function failover is an Enterprise-plan feature. Multi-region placement alone does not fail over, and European functions in front of a Seoul database add an intercontinental round trip to every query. |
| Add a second SQS queue for the merchant outbox | The outbox is already durable in PostgreSQL and drained through SQS. A second queue would be a second source of truth. |
| Automatic DNS failover between Vercel and AWS | The two platforms write to different databases. Failing traffic between them would split the ledger, which the cutover design forbids. |
| Terraform for dedicated Solana RPC nodes | Self-operated nodes were dropped on 22 October 2026. The proposed specification also predates Alpenglow's client requirements. |
| FPGA hardware for ledger balancing | No workload needs it. The ledger invariant is a single database check. |
| Readiness scored at "52.5%", rising to "99.8%" | The percentages are not derived from any measurement and are not used. |

## Open

| Item | Status |
|---|---|
| Passkeys and API keys are held only in the key-value store | **Priority.** They are to move to PostgreSQL, with the key-value store as a cache. |
| Outbound traffic is not restricted to an allowlist of destinations | A cost decision (network firewall or egress proxy) before production. |
| A standby region | Needs a single-writer recovery design first. |
| Choice of database engine (Aurora Serverless v2 versus provisioned RDS) | A cost comparison, priced before any change. |
| Distributed tracing and payment-failure-rate alarms | The application must emit the metrics first. |
| Post-quantum signatures for long-lived evidence | Research; no date. |
