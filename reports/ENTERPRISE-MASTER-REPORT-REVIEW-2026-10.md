# Review of the "Enterprise Architecture Master Report (2026–2027)", October 2026

> **What this is.** An October 2026 research report, "FurlPay × AWS × Solana Alpenglow: Enterprise Architecture Master Report", proposed an AWS architecture and a migration plan. It was reviewed line by line on 22 October 2026 against the application source, the database migrations, Solana mainnet and official vendor documentation. **The report itself is not published here.** Its useful findings are carried into the [enterprise architecture](../docs/ENTERPRISE-ARCHITECTURE-2026-2027.md) and its companions; this page records what was corrected and why.

## Kept

- Do not run validator-class Solana infrastructure before there is a measured need; use managed RPC providers with failover.
- PostgreSQL as the system of record; S3 for documents and evidence; SQS for asynchronous delivery with dead-letter queues.
- Alpenglow facts: activation at epoch 1064 on 21 October 2026; Votor's 80% one-round fast path and 60% two-round slow path; tolerance of 20% adversarial plus 20% offline stake; `confirmed` and `finalized` converging; buffering per `(slot, bank_id)` and reconciling across providers by blockhash; the Clock sysvar change. All confirmed against solana.com, Anza and Triton One.
- End-to-end payment latency is a sum of stages, and consensus finality is only one of them.
- Four application findings, confirmed in the source (see the [ledger document](../docs/LEDGER-SETTLEMENT-RECONCILIATION.md), gaps G1–G4): payment and audit event written as two statements; an in-memory path when durability is unavailable; whole-account write-back; `numeric` balances read into JavaScript `Number`.

## Corrected

| # | Report said | Correction | Evidence |
| :--- | :--- | :--- | :--- |
| 1 | The canonical USDC mint on Solana is `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` | That is the SPL Token **program**. The USDC **mint** is `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | Circle's USDC contract addresses; FurlPay's payment-chain registry |
| 2 | `settle_payment_atomic($1, $2, $3::jsonb)` takes three arguments | It takes eight: `p_payment_id, p_payer, p_tx_hash, p_chain, p_ledger_tx_id, p_memo, p_postings, p_actor_id`, and returns `settled` and `already` | Migration 0009 |
| 3 | `payment_events` has a `triggered_by` column | The columns are `payment_id, from_status, to_status, actor_id, reason, metadata` | Migration 0009 |
| 4 | Deposits are deduplicated in a `processed_transactions` table | No such table exists. Merchant receipts are deduplicated by the primary key `(chain, tx_hash, log_index)` of `merchant_payment_receipts` | Migration 0040 |
| 5 | The payments module falls back to memory "if the database is unreachable" | An unreachable database throws. The in-memory path is taken only when no database is configured or a payments migration is missing. Production refuses to boot without database credentials, so the realistic production risk is a missing migration | Payments module and boot configuration |
| 6 | A sample connection pool uses `ssl: { rejectUnauthorized: true }` | Without the Amazon RDS CA bundle that configuration cannot verify an RDS certificate, which is not in Node's default trust store. The CA must be supplied | AWS RDS TLS documentation |
| 7 | "Point all FurlPay code to `finalized`" | Correct after activation, and the merchant Solana path already reads `finalized`. Triton One's guidance is specifically not to switch before activation, because `finalized` then cost 12.8 s | Triton One |
| 8 | Total latency is 150 ms plus small terms | No end-to-end figure has been measured. FurlPay now measures credit latency (block time to credit, one-second resolution); 150 ms is a consensus target, not a payment time | [Alpenglow document §3](../docs/SOLANA-ALPENGLOW-FINALITY.md#3-consensus-finality-is-not-payment-latency) |
| 9 | Risk and AML run on two named third-party vendors in under 40 ms | Neither vendor is integrated or contracted; the source has only placeholder seams for them. Money routes screen addresses against the OFAC SDN list locally, add Circle's Compliance Engine where configured and Arc's USDC blocklist on Arc, and fail closed. No latency has been measured | Application source |
| 10 | The payment flow's edge is "AWS Cloudflare / API Gateway" | The planned edge is CloudFront → AWS WAF → ALB. API Gateway is not part of the design | Terraform in the application repository |
| 11 | Multi-tenant RLS "active and tested" is a gate to pass | RLS is enabled with no policies, and tenant isolation is enforced in the application and tested against real PostgreSQL. The migration keeps RLS on and adds a least-privilege role; database-enforced tenant policies are research | [PostgreSQL migration §5](../aws/AWS-POSTGRESQL-MIGRATION.md#5-tenant-isolation-roles-and-rls) |
| 12 | Stage 1 includes two named RPC providers with "99.99% SLA" | FurlPay does not hold such an SLA. The code fails over across configured endpoints; provider choice and SLA are commercial terms not in evidence | — |
| 13 | Instance families listed include `i7ie` for Solana | AWS's published guidance for Solana RPC nodes uses `r7a` instances; `i7ie` is not in it. Irrelevant to FurlPay, which runs no nodes | AWS Web3 blog: Run Solana nodes on AWS |

## Removed from public material

| Claim | Why |
| :--- | :--- |
| Named competitors' take rates, yields and "displacement" | Not verified, and FurlPay's public documentation does not name competitors |
| A 0.25–0.75% take rate and pass-through of 3.50–4.25% treasury yield to merchants | No such pricing or yield product exists. Paying yield on payment stablecoin balances raises regulatory questions that require legal advice in each jurisdiction |
| Monthly cost table (about $550, $2,150 and $26,250) and "$288,000/year saved" | Hypothetical; no AWS spend has been incurred and no usage measured. See the [enterprise architecture §13](../docs/ENTERPRISE-ARCHITECTURE-2026-2027.md#13-infrastructure-cost-assumptions) |
| "Non-custodial Safe smart accounts" as a FurlPay capability | Not integrated; under research |
| Workday payroll integration and float-yield distribution as gates FurlPay is about to pass | Not in the production source; Workday work exists only on unmerged branches |
| "Sub-50 ms query latency under 500 concurrent invocations" | Not measured |
| "Certified & Compiled", and a target branch of `extension-redesign` | Not a certification. The application repository's production line is not `extension-redesign` |

## Change log of material factual corrections in this repository (22 October 2026)

- The blog post on dedicated AWS infrastructure said FurlPay "deployed" dedicated Solana RPC nodes. **No nodes have been deployed;** the sentence is corrected in place and the post carries a status note.
- Documents specifying Agave v2.x nodes now carry a note: since Alpenglow, mainnet requires Agave v4.3.2, and dedicated nodes are no longer part of the plan.
- `site-pages/concepts/alpenglow.mdx` and `confirmation-vs-finality.mdx` were one-line stubs; both now state the post-activation facts.
- The verified-claims register gains dated Alpenglow, AWS-migration and payment-rail rows.
