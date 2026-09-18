# FurlPay: 1-Year Master Infrastructure & AWS Migration Blueprint (2026–2027)
### Scaling from Vercel & Managed RPCs to Dedicated AWS Solana Infrastructure, Circle CCTP, and Rain Card Rails

**Published:** September 18, 2026  
**Author:** FurlPay Infrastructure & Security Architecture Team  
**Scope:** 12-Month Enterprise Migration Roadmap from Vercel & Managed Solana RPC Providers to AWS Multi-AZ Dedicated RPC Nodes, Next.js 15 App Router on AWS ECS Fargate, Circle CCTP/Arc Integration, and Rain JIT Card Issuing.

---

## 1. Executive Summary & Strategic Context

FurlPay is transitioning from consumer-facing fintech prototype to Tier-1 financial infrastructure. In the last 6 hours alone, FurlPay's USDC payment rails processed **58,000+ edge requests with a 0% error rate**. With our **Rain partnership** operational for physical and virtual card issuing, and **Circle (USDC, CCTP, and Arc)** as the next major integration milestone, our traffic and reliability demands have outgrown managed, multi-tenant serverless tiers.

### The Core Scaling Problem
1. **Managed RPC Bottlenecks & Noisy Neighbors:** Commercial managed RPC providers (Helius, QuickNode, Triton) enforce strict tier-based rate limits (compute unit caps, burst throttling) and introduce unpredictable cross-tenant latency spikes (150ms–450ms). In high-throughput payment settlement (EIP-3009 gasless transfers, Solana Actions/Blinks, and Rain JIT card authorizations), an RPC delay can cause an authorization timeout (<200ms budget) or dropped transaction.
2. **Serverless Cold Starts & Edge Routing:** Vercel Edge/Serverless functions introduce intermittent execution variability and egress overhead when communicating with external databases (Supabase, Upstash Redis) and blockchain networks.
3. **Escalating Unit Economics:** Managed RPC tiers bill on a per-query/compute-unit model. At 58K requests/6 hours (~7M+ queries/month scaling to 50M+ queries/month), managed RPC and Vercel bandwidth costs scale exponentially, whereas dedicated AWS infrastructure yields flat, predictable monthly costs with unlimited query headroom.

### The AWS Solution
By migrating to a **dedicated Solana RPC cluster built on AWS** using the official **AWS Blockchain Node Runners** blueprint paired with **Next.js 15 on Amazon ECS Fargate and CloudFront**, FurlPay captures:
- **Sub-5ms internal RPC latency:** Next.js API containers communicate directly with dedicated Solana RPC nodes over private AWS VPC subnets.
- **Zero rate-limiting on chain reads & streaming:** Yellowstone Dragon's Mouth gRPC Geyser streaming captures block state, mints, and account balances in real time with zero third-party rate limits.
- **Enterprise-Grade Compliance & Observability:** Direct CloudWatch, AWS WAF, VPC Flow Logs, and KMS Hardware Security Modules (HSM) fulfill strict SOC 2 Type II and PCI-DSS Level 1 compliance requirements.

---

## 2. Industry Benchmark: How Circle & Solana Scale on AWS

### A. Circle’s AWS Infrastructure
- **Exclusive Node Infrastructure:** Circle operates its primary blockchain node infrastructure—supporting USDC across 15+ chains, CCTP (Cross-Chain Transfer Protocol), and the Circle Payments Network—exclusively on AWS using Amazon EKS (Elastic Kubernetes Service) and multi-region EC2.
- **Circle Arc Layer-1 Integration:** Circle’s 2026 economic operating system, **Arc**, where gas is natively denominated in USDC, was developed with AWS participating in testnet node validation.
- **Amazon Bedrock AgentCore Payments:** AWS integrated Circle USDC directly into Bedrock AgentCore, enabling autonomous AI agents to initiate real-time micropayments via USDC—an initiative directly synergistic with FurlPay’s x402 HTTP micropayment protocol.

### B. Solana on AWS: The Node Runners Paradigm
AWS and the Solana Foundation maintain the **AWS Blockchain Node Runners** framework (open-source AWS Cloud Development Kit / CDK):
- **Validated Client Runtimes:** Full native support for **Agave v2.x** and **Frankendancer**.
- **Hardware Architecture:** 
  - **Compute:** AMD EPYC-based **`r7a.16xlarge`** (64 vCPUs, 512 GiB RAM, 3.7 GHz clock) or Storage-Optimized **`i7ie.12xlarge`** / **`i4i.16xlarge`**.
  - **Disk I/O Topology:** Separated storage pools to completely eliminate I/O blocking:
    - **AccountsDB:** Stored in a RAM disk (`tmpfs`) or dedicated ultra-fast NVMe partition (300 GiB+) to eliminate disk thrashing during state updates.
    - **Ledger:** Hosted on high-performance AWS `io2` Block Express (provisioned at 30,000+ IOPS) or local NVMe instance store in RAID-0.
  - **High Availability (HA) Pattern:** Multiple non-voting RPC validator nodes placed in an Auto Scaling Group behind an internal Application Load Balancer (ALB), with health checks verifying slot synchronization height (`slot_lag < 5`).

---

## 3. Target FurlPay AWS Target Architecture

```
                                      [ Internet Clients & Edge Users ]
                                                      │
                                                      ▼
                                       [ AWS Route 53 DNS + Anycast ]
                                                      │
                                                      ▼
                                       [ Amazon CloudFront CDN + WAF ]
                                         (DDoS Shield, Edge Caching)
                                                      │
                         ┌────────────────────────────┴────────────────────────────┐
                         ▼                                                         ▼
                 [ Static Assets ]                                      [ HTTPS API Requests ]
               (Amazon S3 Bucket)                                                  │
                                                                                   ▼
                                                                [ Internet-Facing Application Load Balancer ]
                                                                                   │
                                  ┌────────────────────────────────────────────────┼───────────────────────────────┐
                                  │                                                │                               │
                                  ▼                                                ▼                               ▼
                     [ ECS Fargate: Next.js API ]                    [ ECS Fargate: Next.js API ]        [ ECS Fargate: Next.js API ]
                            (AZ-1a, Private)                                (AZ-1b, Private)                    (AZ-1c, Private)
                                  │                                                │                               │
            ┌─────────────────────┴────────────────────────┬───────────────────────┴───────────────────────────────┤
            ▼                                              ▼                                                       ▼
  [ Amazon ElastiCache ]                        [ Aurora Serverless v2 ]                                [ Internal Network Load Balancer ]
 (Valkey / Redis 7 Cluster)                      (PostgreSQL Ledger DB)                                            │
   • Session tokens & auth                        • Double-entry ledger                                            │
   • Sliding-window rate limits                   • Card auth events                                               │
   • 3DS2 challenge nonces                        • Reconciled balances                                            ▼
                                                                                                 ┌───────────────────────────────────┐
                                                                                                 │ Dedicated Solana RPC HA Cluster   │
                                                                                                 │ (AWS Blockchain Node Runners)     │
                                                                                                 ├───────────────────────────────────┤
                                                                                                 │ • Primary: EC2 r7a.16xlarge       │
                                                                                                 │   - 64 vCPU, 512 GB RAM           │
                                                                                                 │   - tmpfs (AccountsDB, 300 GB)    │
                                                                                                 │   - io2 Block Express (Ledger)    │
                                                                                                 │ • Secondary / Failover: r7a.16xl  │
                                                                                                 │ • Yellowstone gRPC Streamer       │
                                                                                                 │ • Direct Validator Gossip peering │
                                                                                                 └───────────────────────────────────┘
```

---

## 4. 1-Year Comprehensive Phased Migration Roadmap

### Phase 1 (Months 1–3, Q1): Dedicated AWS Solana RPC Cluster & Hybrid Bridge
**Objective:** Deploy FurlPay’s proprietary Solana RPC infrastructure on AWS while maintaining zero downtime on the existing Vercel deployment.

1. **VPC Foundation & Networking:**
   - Deploy multi-AZ Virtual Private Cloud (VPC) in `us-east-1` (3 Public subnets, 3 Private Application subnets, 3 Isolated Database/Blockchain subnets).
   - Configure AWS Transit Gateway and VPC Flow Logs with CloudWatch logging.
2. **Solana RPC Cluster Deployment (AWS Blockchain Node Runners):**
   - Synthesize AWS CDK Solana blueprint on AWS EC2 `r7a.16xlarge` (Agave v2.2+).
   - Configure storage mounts:
     - Root: 200 GB `gp3`.
     - AccountsDB: 350 GB `tmpfs` RAM disk (`/mnt/accounts-db`).
     - Ledger: 2 TB `io2` Block Express with 32,000 Provisioned IOPS.
   - Install and configure **Yellowstone Dragon’s Mouth** gRPC Geyser plugin for zero-lag account and transaction streaming.
   - Configure health-checking daemon: checks `curl -s http://localhost:8899/health` and compares slot height against network cluster tip.
3. **Hybrid Ingestion & Backend Connection:**
   - Place RPC nodes behind an internal AWS Network Load Balancer (NLB) with AWS Global Accelerator endpoint.
   - Update `apps/web/src/lib/services/chainRegistry.ts` to add the dedicated AWS RPC URL as primary, with Helius as secondary fallback:
     ```ts
     solana: {
       rpcUrl: process.env.SOLANA_DEDICATED_AWS_RPC || "https://rpc.internal.furlpay.com",
       fallbackRpcUrl: process.env.HELIUS_RPC_URL,
     }
     ```
   - Benchmark latency: Verify drop in Solana account balance fetch and signature confirmation times from ~240ms to <25ms.

---

### Phase 2 (Months 4–6, Q2): Web Tier & Database Migration (Vercel to AWS ECS Fargate)
**Objective:** Move the Next.js 15 web application, caching, and background jobs from Vercel to AWS ECS Fargate.

1. **Containerization & Build Pipeline:**
   - Create production multi-stage `Dockerfile` for `apps/web` utilizing Next.js standalone output mode (`output: 'standalone'`).
   - Configure automated GitHub Actions CI/CD to build, test, scan with Trivy/Gitleaks, and push container images to Amazon Elastic Container Registry (ECR).
2. **Compute & Caching Architecture:**
   - Provision Amazon ECS Cluster on AWS Fargate with Auto Scaling policies (target tracking: 60% CPU / 70% Memory).
   - Deploy **Amazon ElastiCache for Valkey / Redis 7** (Multi-AZ with auto-failover), replacing Upstash Redis:
     - Sub-millisecond latency for rate limiting (`apps/web/src/lib/rateLimit.ts`).
     - Global session validation (`apps/web/src/lib/session.ts`).
     - Fast nonces for 3DS2 card authorization challenges.
3. **Database & Secret Management:**
   - Migrate Supabase operational ledger to **Amazon Aurora Serverless v2 (PostgreSQL 16)** with read replicas.
   - Transition application secrets from Vercel Environment Variables to **AWS Secrets Manager** with automatic rotation.
4. **Weighted DNS Traffic Cutover:**
   - Deploy CloudFront distribution with AWS WAF rules (rate limiting, bad bot blocking, SQLi/XSS prevention).
   - Configure Amazon Route 53 DNS weighted routing:
     - Week 1: 10% traffic to AWS CloudFront, 90% Vercel.
     - Week 2: 25% AWS, 75% Vercel.
     - Week 3: 50% AWS, 50% Vercel.
     - Week 4: 100% AWS. Decommission Vercel production deployment.

---

### Phase 3 (Months 7–9, Q3): Rain Card Issuing Hardening & Circle CCTP Integration
**Objective:** Leverage low-latency AWS infrastructure to optimize card issuing webhooks and cross-chain stablecoin settlement.

1. **Rain JIT (Just-In-Time) Card Authorization Engine:**
   - Connect Rain Card Webhook (`apps/web/src/app/api/webhooks/card-auth`) directly through CloudFront / ALB into dedicated ECS workers.
   - Total decision budget: **180ms**. 
     - Network transit: 40ms.
     - Balance check on ElastiCache / Aurora: 15ms.
     - Velocity & risk check (`lib/security/riskEngine.ts`): 25ms.
     - Response return: 30ms.
     - Net SLA: ~110ms (well within Visa/Mastercard 200ms bounds).
2. **Circle CCTP v2 Native Mint/Burn Relayer:**
   - Deploy dedicated CCTP relayer service on ECS listening to Circle attestation services and Solana burn events via Yellowstone gRPC.
   - Zero RPC lag allows instantaneous detection of `CircleMessageTransmitter` events across Arbitrum, Base, Ethereum, and Solana.
3. **Turnkey KMS / AWS KMS Financial Vault:**
   - Fully integrate AWS CloudHSM / KMS with Turnkey for non-custodial and custodial treasury authorizations, strictly enforcing the rule: *No private key material ever enters application container memory*.

---

### Phase 4 (Months 10–12, Q4): Multi-Region Expansion, Arc Layer-1 & Enterprise Optimization
**Objective:** Global financial corridor expansion, Circle Arc integration, and cost lock-in.

1. **Circle Arc Layer-1 Protocol Evaluation:**
   - Deploy dedicated Arc validator / RPC node within AWS VPC upon Arc mainnet release.
   - Integrate native USDC gas settlement into FurlPay checkout rails.
2. **Multi-Region Read Corridors:**
   - Deploy read-only RPC nodes and CloudFront edge points in `eu-west-1` (Dublin - MiCA compliance) and `ap-southeast-1` (Singapore - APAC luxury travel corridor).
3. **Enterprise Cost Optimization:**
   - Commit to 1-Year or 3-Year **AWS Compute Savings Plans** and **EC2 Reserved Instances** for the `r7a.16xlarge` instances.
   - Achieve an immediate 45% to 58% discount on compute infrastructure.
4. **SOC 2 Type II & PCI-DSS Level 1 Certification:**
   - Conduct external compliance audit utilizing AWS Audit Manager, AWS Security Hub, and AWS CloudTrail logs.

---

## 5. Financial & TCO Analysis: Vercel/Managed vs. AWS Dedicated

Below is a 1-year Total Cost of Ownership (TCO) model assuming traffic scales from 10M requests/month to 60M requests/month.

### Current Managed Architecture (Vercel + Helius + Upstash)
| Line Item | 10M Reqs/Mo | 30M Reqs/Mo | 60M Reqs/Mo | Annual Total (Weighted) |
|---|---|---|---|---|
| **Vercel Pro/Enterprise** (Fast Data Transfer, Edge Compute, Serverless execution) | \$450/mo | \$1,400/mo | \$3,200/mo | \$18,500 |
| **Managed Solana RPCs** (Helius / QuickNode Enterprise tiers, compute units) | \$1,500/mo | \$3,800/mo | \$7,500/mo | \$48,000 |
| **Upstash Redis + Supabase Enterprise** | \$250/mo | \$650/mo | \$1,300/mo | \$8,200 |
| **Total Estimated Spend** | **\$2,200/mo** | **\$5,850/mo** | **\$12,000/mo** | **\$74,700/year** |

### Dedicated AWS Architecture (Node Runners + ECS + Aurora + ElastiCache)
*(With 1-Year Compute Savings Plans & CloudFront Volume Discounts)*

| Component | AWS Resource Specification | Monthly Cost | Annual Cost |
|---|---|---|---|
| **Solana RPC Primary** | EC2 `r7a.16xlarge` (64 vCPU, 512GB RAM) + 2TB `io2` Block Express (32K IOPS) | \$1,680/mo | \$20,160 |
| **Solana RPC Standby / Failover** | EC2 `r7a.8xlarge` (32 vCPU, 256GB RAM) + 2TB `gp3` (Warm snapshot recovery) | \$850/mo | \$10,200 |
| **Web Tier (ECS Fargate)** | 4 to 12 Tasks (2 vCPU, 4GB RAM) Auto-scaling | \$280/mo | \$3,360 |
| **Caching Tier** | Amazon ElastiCache for Valkey (Cluster Mode, 2 nodes) | \$140/mo | \$1,680 |
| **Database Tier** | Amazon Aurora Serverless v2 (PostgreSQL 16, 2–8 ACUs) | \$350/mo | \$4,200 |
| **Networking & CDN** | Amazon CloudFront (10TB/mo) + AWS WAF + ALB / NLB | \$210/mo | \$2,520 |
| **Observability & Security** | AWS KMS + Secrets Manager + CloudWatch Logs + GuardDuty | \$160/mo | \$1,920 |
| **Total AWS Dedicated Spend** | **Fixed Capacity: Up to 150M+ requests/month** | **\$3,670/mo** | **\$44,040/year** |

### Key Economic Takeaways
- **Breakeven Point:** At **~22M requests/month**, the AWS dedicated architecture becomes significantly cheaper than Vercel + managed RPCs.
- **Net 1-Year Savings at Scale:** **\$30,660+ saved annually** at projected growth rates.
- **Zero Query Ceiling:** On AWS, executing 100M+ RPC queries costs **\$0 in incremental RPC provider fees**.

---

## 6. Security, Compliance, and SRE Risk Register

| Risk Event | Likelihood | Impact | Mitigation Strategy |
|---|---|---|---|
| **Solana RPC Node Slot Lag** | Medium | High | Node Runners automated health check monitors slot distance. If `slot_lag > 5`, NLB automatically drains traffic and fails over to secondary node, with managed Helius RPC as external failback. |
| **NVMe Disk Exhaustion** | Low | Critical | Automated ledger prune script retains the latest 200,000 slots (~24 hours of ledger history). Cold historical data archived to Amazon S3 Glacier. |
| **Vercel Cutover Service Interruption** | Low | High | Route 53 weighted canary deployment (10% increments) over 4 weeks. Automated CloudWatch synthetic monitors trigger instant DNS rollback if 5xx rate > 0.05%. |
| **Rain 3DS2 Webhook Timeout** | Low | High | ElastiCache Redis memory-first verification for nonces and balances; asynchronous ledger reconciliation post-authorization. |

---

## 7. Immediate Next Steps (Day 1 Execution Checklist)

1. [ ] **Provision AWS Organization & Accounts:** Create separate AWS accounts for `Production`, `Staging`, and `Shared Services` under AWS Control Tower.
2. [ ] **Clone & Test AWS Blockchain Node Runners:**
   ```bash
   git clone https://github.com/aws-samples/aws-blockchain-node-runners.git
   cd aws-blockchain-node-runners
   npm install
   ```
3. [ ] **Synthesize Solana RPC Staging Stack:** Deploy single `r7a.8xlarge` node in `us-east-1` with Agave v2.2 to establish baseline slot synchronization performance.
4. [ ] **Create Standalone Dockerfile for `apps/web`:** Validate container build locally using `docker build -t furlpay-web apps/web`.
5. [ ] **Present Blueprint to Executive Leadership:** Align on Phase 1 timeline and budget commitments.
