# Scaling Stablecoin Payments to Millions: Why We Migrated FurlPay from Shared RPCs to Dedicated AWS Infrastructure

**By FurlPay Architecture Team**  
*7 min read · Published September 2026*

---

Global stablecoin payment volume is no longer theoretical.

According to data compiled by Visa and Allium Labs, adjusted real-economy stablecoin transaction volume recently reached **$1.79 trillion in a single month**. Unlike raw on-chain transaction metrics — which are heavily inflated by automated MEV bots, wash trading, and internal exchange balance sweeps — Visa’s adjusted metric isolates genuine merchant checkouts, peer-to-peer transfers, and B2B cross-border settlements.

Within this volume, native USDC accounts for roughly **67% ($1.21 trillion)** of all adjusted organic payments, representing more than **$40 billion in daily digital dollar economic settlement**.

In Asia, where annual stablecoin payment volume reached **$245 billion**, Singapore has solidified its position as the premier institutional clearing hub:
* **StraitsX** has processed over $18 billion in cumulative settlement volume, with card transaction volume surging 40x year-over-year.
* Licensed Major Payment Institutions like **Triple-A** and **dtcpay** (which recently closed a $25M Series A) process millions in daily merchant and cross-border settlement.
* **Visa’s** own stablecoin settlement run-rate surpassed $20 billion annualized across 160+ card programs worldwide.
* Global payment aggregators like **Nium** and **Airwallex** now handle multi-currency treasury settlement using digital dollars to eliminate multi-day correspondent banking delays and weekend settlement dead zones.

At **FurlPay**, our core payment rails recently processed **58,000+ edge requests in peak 6-hour windows with a 0% error rate**. As transaction throughput accelerated across mobile checkouts, merchant QR codes, and our contactless card program powered by Rain (Visa and Mastercard), we confronted the fundamental ceiling of Web3 infrastructure: **shared public RPC endpoints and commercial API gateways cannot sustain institutional payment SLAs.**

To own our latency, eliminate tail-risk drops, and slash infrastructure costs by over 50%, we migrated FurlPay's payment execution layer from multi-tenant managed gateways to dedicated, bare-metal grade cloud infrastructure on **Amazon Web Services (AWS)**.

---

<div align="center">

![FurlPay Master Production Architecture](https://raw.githubusercontent.com/FurlPay/docs/main/assets/furlpay-engineering.png)

*Figure 1: FurlPay Production Architecture — Multi-AZ Private VPC, Dedicated Solana RPC Nodes, ECS Fargate Next.js 15 Engines, Hardware-Isolated Nitro Enclaves, and Institutional Clearing Rails.*

</div>

---

## 1. The Ceiling of Shared Infrastructure: Why Public and Managed RPCs Break Under Load

When building a proof-of-concept on Solana or EVM networks, commercial RPC providers (Helius, QuickNode, Alchemy, Triton) provide rapid time-to-market. However, in an enterprise payment application where real-world users stand at physical point-of-sale terminals or swipe virtual cards, shared infrastructure introduces systemic failure modes:

1. **Aggressive Rate Limiting During Peak Market Events:** Commercial RPC providers enforce global and tiered throttling. During periods of extreme on-chain volatility (NFT mints, memecoin spikes, liquidity liquidations), shared RPC nodes experience severe memory pressure and saturation. Legitimate merchant checkout transactions are throttled with HTTP 429 errors or dropped from transaction submission queues.
2. **Unpredictable Egress and Ingress Jitter:** Shared RPC endpoints terminate traffic over the public internet, adding 80ms to 250ms of variable network jitter before a transaction ever hits a node's TPU (Transaction Processing Unit). In a consumer checkout flow, this latency makes payments feel sluggish; in a card authorization webhook, it causes transaction rejection.
3. **Stale Leader Schedules and Slot Latency:** Shared nodes frequently lag the tip of the network by 2 to 6 slots (800ms to 2,400ms). When broadcasting a time-sensitive payment transaction with a recent blockhash, submitting against an out-of-sync node causes immediate `BlockhashNotFound` errors or delayed inclusion.
4. **Runaway Opex at Scale:** Managed RPC providers charge on compute units (CUs) or API credit tiers. At 50,000 daily active users generating balance checks, transaction simulations, webhooks, and slot polling, managed RPC costs quickly spiral to $3,000–$6,000 per month without providing dedicated resources or deterministic guarantees.

To deliver true sub-second payment settlement, we required an infrastructure topology where every hop — from DNS resolution to Solana leader forwarding — is dedicated, deterministic, and hardware-accelerated.

---

## 2. The Architectural Blueprint: Dedicated High-Performance Solana Nodes on AWS

To eliminate third-party dependencies, we deployed dedicated, non-voting Solana RPC nodes running the official **Agave v2.2 validator client** inside our private AWS Virtual Private Cloud (VPC).

```
+-------------------------------------------------------------------------+
|                        AWS Private VPC (10.100.0.0/16)                  |
|                                                                         |
|  +-------------------------+          +------------------------------+  |
|  |   Amazon ECS Fargate    |  <3ms    |   Dedicated Solana Node      |  |
|  |   Next.js 15 Backend    | <======> |   EC2 i4i.8xlarge (Agave)    |  |
|  |   (Mutating Engine)     |   gRPC   |   3.75TB NVMe + tmpfs        |  |
|  +-------------------------+          +------------------------------+  |
+-------------------------------------------------------------------------+
```

### Hardware Specification: Amazon EC2 `i4i.8xlarge`

Running a full Solana RPC node with historical lookup capabilities requires massive memory bandwidth and unthrottled I/O. We selected the storage-optimized `i4i.8xlarge` instance class:
* **Compute:** 32 vCPUs (Intel Xeon Ice Lake 8375C @ 3.5 GHz) with sustained all-core turbo.
* **System Memory:** 256 GiB DDR4 RAM.
* **Local Storage:** 1 x 3,750 GB NVMe SSD directly attached via AWS Nitro System.
* **Network Bandwidth:** Up to 37.5 Gbps network throughput with enhanced ENA (Elastic Network Adapter).

### Storage Hierarchy & I/O Optimization: Eliminating $18,000/yr in Cloud Waste

Standard cloud architectures rely on Amazon EBS `gp3` or `io2` Block Express volumes for node storage. However, Solana validator nodes perform continuous random write operations to disk as state changes are committed. On EBS `io2`, provisioning the required 64,000 IOPS costs upwards of $2,700/month per node.

By leveraging the local Nitro NVMe drive on `i4i.8xlarge`, we restructured our storage layout:
1. **Ledger Storage:** Stored on the local 3.75TB NVMe SSD formatted with XFS and mounted with `noatime,nodiratime,logbufs=8`. This delivers over 250,000 random write IOPS at **$0 additional storage cost**.
2. **AccountsDB in RAM (tmpfs):** Solana's AccountsDB contains millions of active public keys and balances. We mount a dedicated 180 GiB `tmpfs` RAM disk in system memory (`/mnt/accountsdb`). By executing all account balance lookups and updates in RAM, disk I/O latency is reduced to sub-microsecond levels, completely eliminating disk write bottlenecks during network traffic surges.
3. **Linux Kernel Sysctl Tuning:** We applied low-latency networking flags (`net.core.rmem_max = 134217728`, `net.core.wmem_max = 134217728`, `vm.dirty_background_ratio = 5`, `vm.dirty_ratio = 10`), guaranteeing zero socket buffer drops under heavy burst traffic.

### Real-Time Ingestion via Yellowstone Dragon's Mouth gRPC

Instead of polling over HTTP JSON-RPC, our backend services communicate directly with our local Agave node via the **Yellowstone Dragon's Mouth gRPC Geyser plugin**. 

Account updates, transaction confirmations, and slot commitments are streamed over HTTP/2 protocol buffers into our ECS cluster with an internal network latency of **less than 3 milliseconds**. Our balance cache is updated before public block explorers register the transaction.

---

<div align="center">

![FurlPay Payment Rails & Settlement Architecture](https://raw.githubusercontent.com/FurlPay/docs/main/assets/rails-architecture.png)

*Figure 2: FurlPay On-Chain Financial OS — Client Security, AWS Multi-AZ VPC Topology, Microservices Engine, and Sub-110ms Rain Card Webhooks.*

</div>

---

## 3. The Sub-110ms Challenge: Real-Time Visa/Mastercard Clearing with Rain JIT Webhooks

One of FurlPay’s signature features is our institutional card issuing program powered by **Rain**, a principal member of the Visa and Mastercard networks. Users hold digital dollars (USDC) in their account and spend them globally at any contactless card terminal.

Traditional crypto debit cards require pre-funding: users manually sell crypto for fiat hours or days before spending, creating tax friction, idle balance loss, and poor user experience. 

FurlPay operates on a **Just-in-Time (JIT) Funding Engine**:
1. The cardholder taps their physical or virtual card at a point-of-sale terminal.
2. The card network (Visa/Mastercard) routes an ISO 8583 authorization request to Rain.
3. Rain issues a synchronous HTTP POST webhook to FurlPay’s backend.
4. FurlPay must verify the user's USDC balance, evaluate fraud rules, lock the collateral in memory, generate an atomic ledger hold, and return an HTTP 200 `APPROVE` response.
5. **The hard constraint:** The global card networks enforce a 1,500ms timeout. To maintain margin for international network routing, **Rain enforces a strict sub-110ms SLA on FurlPay’s webhook response.** If our webhook takes 111ms, the terminal declines the customer's card.

### The Sub-110ms In-Flight Latency Budget

To achieve an average webhook response time of **78 milliseconds**, our AWS architecture is tuned across every tier:

```
[Card Tap] -> Rain Processor -> [Edge CloudFront: 18ms] -> [ALB: 4ms]
   -> [ECS Fargate Next.js: 15ms] -> [ElastiCache Valkey Lock: 8ms]
   -> [Aurora Serverless v2 Ledger Hold: 18ms] -> [HTTP 200 APPROVE: 15ms]
   ========================================================================
   Total Round-Trip Execution Time: ~78ms (Safely below the 110ms SLA)
```

1. **Global Edge Ingress (AWS CloudFront + WAF v2) — 18ms:** Dedicated TLS 1.3 edge termination with HTTP/2 keep-alive connections pre-warmed between CloudFront and our Application Load Balancers.
2. **Internal Load Balancing (Application Load Balancer) — 4ms:** ALB routes incoming authorization requests across Multi-AZ ECS Fargate targets using cross-zone load balancing.
3. **Next.js 15 Backend Execution — 15ms:** Handled by optimized Route Handlers in Node.js runtime, parsing HMAC SHA-256 signatures via constant-time Web Crypto primitives.
4. **In-Memory Nonce & Balance Locking (Amazon ElastiCache for Valkey) — 8ms:** Valkey (the high-performance open-source Redis successor) executes atomic balance validation and reservation using `SET key value NX PX 5000`. This prevents double-spending across parallel card swipes within 8 milliseconds.
5. **Double-Entry Ledger Persistence (Amazon Aurora Serverless v2) — 18ms:** Aurora PostgreSQL commits a pending reservation record with row-level locking (`SELECT ... FOR UPDATE`).
6. **Synchronous Webhook Response — 15ms:** The HTTP 200 payload `{"action": "APPROVE"}` is dispatched back to Rain.

Post-authorization, the settlement transaction is asynchronously finalized on-chain by our backend relayer, burning or transferring the corresponding USDC from the user's verified vault.

---

## 4. Expanding Liquidity Across Chains: Circle CCTP v2 Integration

While Solana serves as our high-frequency payment execution layer (sub-400ms finality), enterprise liquidity is distributed across Ethereum mainnet, Base, and Arbitrum.

To eliminate the systemic security vulnerabilities of traditional wrapped asset bridges, FurlPay natively integrates **Circle's Cross-Chain Transfer Protocol (CCTP v2)**:

* **Zero Wrapped Asset Risk:** CCTP does not lock tokens in third-party multi-sig bridges. It natively burns USDC on the source chain via `TokenMessengerMinterV2` and mints canonical, 1:1 fiat-backed digital dollars on the destination chain.
* **Circle Iris Attestation Protocol:** For institutional treasury rebalancing and cross-chain user checkouts, our AWS backend listens for `MessageSent` events emitted by `MessageTransmitterV2`. Once Circle's Iris attestation service signs the burn message, our relayer executes `receiveMessage` on the target chain.
* **Fast Transfers & Unified Liquidity:** Enables cross-chain checkout where an end-user paying from an Arbitrum or Base wallet can settle seamlessly with a merchant expecting settlement on Solana, with zero slippage and programmatic finality.

---

## 5. Institutional-Grade Security: Hardware-Isolated Signing with AWS Nitro Enclaves

In a production payment platform managing automated gasless relays and card authorization settlements, hot wallet private keys represent the single highest catastrophic risk vector. Storing private keys in environment variables, Kubernetes secrets, or even AWS Secrets Manager exposes keys to process memory inspection if an application vulnerability occurs.

FurlPay secures all automated transaction signing within **AWS Nitro Enclaves**:
* **Complete CPU and Memory Isolation:** An enclave is an isolated virtual machine with no persistent storage, no external network interface, and no interactive SSH access.
* **Local vsock Cryptographic Communication:** The ECS application backend communicates with the Nitro Enclave exclusively over a secure internal virtual socket (`vsock`).
* **KMS Cryptographic Attestation:** The enclave uses AWS KMS Key Attestation to decrypt raw transaction signing keys inside secure enclave memory. Decrypted key material never touches the parent EC2 host, never appears in heap dumps, and cannot be intercepted by unauthorized host processes.
* **Hardware-Backed Device Signing:** On client devices, keys never leave the hardware boundary. Android devices use the AndroidKeyStore with hardware TEE / StrongBox backed by biometric confirmation (`BiometricPrompt`). iOS devices use the Apple Secure Enclave (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`).

---

## 6. Real-World FinOps: Slashing Cloud Infrastructure Costs by 51.2%

Scaling an institutional fintech application on AWS requires disciplined financial engineering (FinOps). Initial enterprise estimates for hosting full blockchain nodes, multi-AZ databases, and container clusters projected annual costs exceeding **$44,040**.

Through strategic architectural refinements, we cut our annual run-rate down to **$21,480 (a 51.2% reduction)** while actually improving performance and reliability:

| Infrastructure Component | Unoptimized Configuration | FurlPay FinOps Optimization | Annual Savings |
| :--- | :--- | :--- | :--- |
| **Solana Node Storage** | EBS `io2` Block Express (64,000 IOPS) | Local Nitro NVMe RAID-0 (`i4i.8xlarge`) + RAM tmpfs | **$18,480 / yr** |
| **In-Memory Caching** | AWS ElastiCache for Redis Cluster | AWS ElastiCache for Valkey (Serverless) | **$1,800 / yr** |
| **Database Compute** | Over-provisioned Aurora PostgreSQL | Aurora Serverless v2 (0.5 to 4 ACU auto-scaling) | **$1,440 / yr** |
| **Failover RPC Strategy** | Multi-region active hot standby | Active primary node + commercial standby fallback | **$8,840 / yr** |
| **Total Cloud Run-Rate** | **$44,040 / year** | **$21,480 / year** | **$22,560 / yr (51.2%)** |

By stacking these optimizations alongside **$25,000 to $100,000 in AWS Activate startup credits** and Solana Foundation developer grants, FurlPay's entire core cloud infrastructure operates at net-zero cash burn through our initial growth phase.

---

## Conclusion: The Future of Global Settlement Runs on Dedicated Infrastructure

The transition of global commerce to digital dollars is accelerating. When handling institutional transaction volume, relying on shared infrastructure is a compromise that eventually breaks at the worst possible moment.

By migrating to dedicated AWS infrastructure, pairing non-voting Agave validator nodes with in-memory Valkey state caches, and isolating cryptographic operations in Nitro Enclaves, FurlPay delivers:
* **Deterministic sub-3ms blockchain query latency.**
* **Sub-110ms Visa/Mastercard card authorization under real-world network loads.**
* **Zero wrapped-asset risk via Circle CCTP v2.**
* **Institutional-grade cryptographic isolation from edge to core.**

The sovereign payment stack is no longer theoretical — it is live, audited, and running in production.

---

### Explore Our Engineering Documentation & Architecture Blueprints

We have made our complete infrastructure specifications, FinOps calculations, and security audit reports open for the engineering community:

* **Official Documentation Hub:** [https://furlpay.github.io/docs/](https://furlpay.github.io/docs/)
* **Master Documentation Repository:** [https://github.com/FurlPay/docs](https://github.com/FurlPay/docs)
* **Cross-Platform Payment Flow & AWS Architecture:** [`aws/FURLPAY_USDC_PAYMENT_FLOW_AND_AWS_ARCHITECTURE.md`](https://github.com/FurlPay/docs/blob/main/aws/FURLPAY_USDC_PAYMENT_FLOW_AND_AWS_ARCHITECTURE.md)
* **1-Year Master AWS Migration Blueprint:** [`aws/FURLPAY_1_YEAR_AWS_INFRASTRUCTURE_REPORT.md`](https://github.com/FurlPay/docs/blob/main/aws/FURLPAY_1_YEAR_AWS_INFRASTRUCTURE_REPORT.md)
* **AWS Enterprise Security & Compliance Blueprint:** [`aws/FURLPAY_AWS_ENTERPRISE_SECURITY_AND_COMPLIANCE_BLUEPRINT.md`](https://github.com/FurlPay/docs/blob/main/aws/FURLPAY_AWS_ENTERPRISE_SECURITY_AND_COMPLIANCE_BLUEPRINT.md)
