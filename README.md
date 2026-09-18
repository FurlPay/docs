# FurlPay Documentation & Architecture Blueprints

<div align="center">

![FurlPay Banner](https://raw.githubusercontent.com/FurlPay/.github/main/profile/banner.png)

### The Sovereign Digital Dollar Payment & Card Infrastructure

[![Solana](https://img.shields.io/badge/Solana-9945FF?style=for-the-badge&logo=solana&logoColor=white)](https://solana.com)
[![AWS](https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com)
[![Circle](https://img.shields.io/badge/Circle_USDC_&_CCTP-007AFF?style=for-the-badge&logo=circle&logoColor=white)](https://circle.com)
[![Next.js 15](https://img.shields.io/badge/Next.js_15-000000?style=for-the-badge&logo=next.js&logoColor=white)](https://nextjs.org)
[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![React Native](https://img.shields.io/badge/React_Native-61DAFB?style=for-the-badge&logo=react&logoColor=black)](https://reactnative.dev)
[![Kotlin](https://img.shields.io/badge/Kotlin_2.1-7F52FF?style=for-the-badge&logo=kotlin&logoColor=white)](https://kotlinlang.org)
[![Swift](https://img.shields.io/badge/Swift_6.0-F05138?style=for-the-badge&logo=swift&logoColor=white)](https://swift.org)
[![Rust](https://img.shields.io/badge/Rust_Anchor-000000?style=for-the-badge&logo=rust&logoColor=white)](https://www.rust-lang.org)
[![PostgreSQL](https://img.shields.io/badge/Aurora_PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://aws.amazon.com/rds/aurora/)
[![Redis Valkey](https://img.shields.io/badge/ElastiCache_Valkey-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://valkey.io)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com)
[![Terraform](https://img.shields.io/badge/Terraform_IaC-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)](https://www.terraform.io)
[![Linux](https://img.shields.io/badge/Ubuntu_Linux-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com)

**Enterprise-grade technical documentation, AWS cloud migration roadmaps, dedicated Agave v2.x Solana RPC specifications, Circle CCTP v2 cross-chain rails, Rain JIT card issuing blueprints, and comprehensive security audits.**

</div>

---

## 🏛️ System Architecture Overview

```
+-------------------------------------------------------------------------------------------------------------------------+
|                                          FURLPAY END-TO-END USDC PAYMENT TOPOLOGY                                        |
+-------------------------------------------------------------------------------------------------------------------------+
|                                                                                                                         |
|   +--------------------------+    +--------------------------+    +--------------------------+                          |
|   |   Android Mobile App     |    |    Wear OS Companion     |    |      iOS Mobile App      |                          |
|   |     (React Native /      |    |   (Kotlin Compose Wear   |    |    (Swift SDK / Expo     |                          |
|   |    AndroidKeyStore)      |    |     Data Layer Sync)     |    |     Secure Enclave)      |                          |
|   +------------+-------------+    +------------+-------------+    +------------+-------------+                          |
|                |                               |                               |                                        |
|                +-------------------------------+-------------------------------+                                        |
|                                                | TLS 1.3 / Certificate Pinned (mTLS)                                    |
|                                                v                                                                        |
|   +-----------------------------------------------------------------------------------------------------------------+   |
|   | AWS Ingress Layer: Amazon CloudFront Global Edge + AWS WAF v2 (DDoS, Bot Control, Token Bucket Rate Limiting)   |   |
|   +----------------------------------------------------+------------------------------------------------------------+   |
|                                                        |                                                                |
|                                                        v                                                                |
|   +-----------------------------------------------------------------------------------------------------------------+   |
|   | Application Load Balancer (ALB) - Multi-AZ Private VPC Ingress (Sub-5ms Health Checks, HTTP/2 & gRPC Multiplex)  |   |
|   +----------------------------------------------------+------------------------------------------------------------+   |
|                                                        |                                                                |
|                                                        v                                                                |
|   +-----------------------------------------------------------------------------------------------------------------+   |
|   | AWS ECS Fargate: Next.js 15 App Router Backend Cluster (apps/web)                                              |   |
|   |   - /api/transfers/gasless (EIP-3009 Relayer Engine)       - /api/actions/pay/[orderId] (Solana Blinks)             |   |
|   |   - /api/payments/create & execute (Payment Intents)      - /api/webhooks/card-auth (Rain JIT Card Engine)         |   |
|   +------------+-------------------------------+-------------------------------+------------------------------------+   |
|                |                               |                               |                                        |
|                v                               v                               v                                        |
|   +-------------------------+   +-----------------------------+   +-------------------------------------------------+   |
|   | AWS ElastiCache Valkey  |   | Amazon Aurora Serverless v2 |   | AWS Nitro Enclaves (Isolated Signing vsock)     |   |
|   | - Nonce Claims (kvSetNx)|   | - Double-Entry Ledger       |   | - Gasless Relayer Private Keys                  |   |
|   | - Rain JIT Balance Locks|   | - Idempotency Records       |   | - Circle CCTP Mint Execution Keys               |   |
|   | - Sub-1ms Session State |   | - Multi-AZ KMS Encrypted    |   | - Cryptographic Attestation Decrypt via KMS     |   |
|   +-------------------------+   +-----------------------------+   +-------------------------------------------------+   |
|                                                                                                |                        |
|                        +-----------------------+-----------------------+-----------------------+                        |
|                        |                                               |                                                |
|                        v                                               v                                                |
|   +-----------------------------------------+     +-----------------------------------------------------------------+   |
|   | Dedicated Solana Agave v2.x Node (AWS)  |     | Circle & External Card Clearing Rails                           |   |
|   | - EC2 i4i.8xlarge (32 vCPU, 256GB RAM)  |     | - Circle CCTP v2 (TokenMessengerMinterV2 / MessageTransmitter)  |   |
|   | - 3.75TB NVMe RAID-0 + 180GB tmpfs      |     | - Circle Iris Attestation API (<10s Soft Finality)              |   |
|   | - Yellowstone Dragon's Mouth gRPC Geyser|     | - Circle Mint API (Fedwire, ACH, SEPA Real-Time Settlement)    |   |
|   | - Sub-5ms Internal VPC Latency          |     | - Rain JIT Card Rails (Visa/Mastercard Authorization <110ms)    |   |
|   +-----------------------------------------+     +-----------------------------------------------------------------+   |
|                                                                                                                         |
+-------------------------------------------------------------------------------------------------------------------------+
```

---

## 📚 Master Documentation Index

### 🚀 1. AWS Cloud & Dedicated Infrastructure Blueprints (`aws/`)

* [**Cross-Platform USDC Payment Engine & AWS Architecture (2026–2027)**](aws/FURLPAY_USDC_PAYMENT_FLOW_AND_AWS_ARCHITECTURE.md)  
  *End-to-end USDC payment flow across Android Native (`native-app`), Wear OS (`guardian`), iOS (`furlpay-swift`), and AWS dedicated infrastructure. Details hardware Keystore/Secure Enclave signing, EIP-3009 gasless transfers, Solana v0 transactions, sub-110ms Rain JIT card authorizations, Circle CCTP v2 burn-and-mint, and AWS Nitro Enclaves.*
* [**1-Year Master Infrastructure & AWS Migration Blueprint (2026–2027)**](aws/FURLPAY_1_YEAR_AWS_INFRASTRUCTURE_REPORT.md)  
  *Complete 12-month engineering roadmap moving FurlPay from Vercel edge functions and managed RPC providers (Helius/QuickNode) to a private Multi-AZ VPC on Amazon Web Services (AWS).*
* [**AWS Infrastructure Cost Optimization & FinOps Reduction Report**](aws/FURLPAY_AWS_COST_OPTIMIZATION_REPORT.md)  
  *FinOps engineering analysis slashing annual AWS infrastructure expenditures from **\$44,040/year** down to **\$21,480/year (51.2% reduction)** by eliminating `io2` Block Express IOPS charges in favor of local Nitro NVMe SSDs (`i4i.8xlarge`), adopting ElastiCache for Valkey, and optimizing standby RPC failover.*
* [**Zero-Cash AWS Blueprint & Production Solana Node Specification**](aws/FURLPAY_AWS_FREE_TIER_NODE_CONFIG_REPORT.md)  
  *Running 100% free on AWS via capital stacking (\$25,000–\$100,000 AWS Activate credits + Solana Foundation grants). Contains official Agave v2.2 validator arguments, Linux sysctl tuning (`21-agave-validator.conf`), dual NVMe RAID-0 storage scripts, tmpfs AccountsDB setup, Yellowstone Dragon's Mouth gRPC configuration, and automated slot health monitoring.*

---

### 🛡️ 2. Security Audits & Vulnerability Assessments (`audit/`)

* [**Master Monorepo Security Audit Report**](audit/FULL_CODEBASE_AUDIT_REPORT.md)  
  *Exhaustive code audit across Solana Anchor programs, Next.js API routes, Android/Wear OS native applications, and client SDKs. Uncovers and remediates critical vulnerabilities including the Solana escrow token drain in `auto_release.rs`, merchant wallet fallback to `1111...1111`, WooCommerce/Magento HMAC bypass, Swift timing side-channel leaks, and Android SQLite encryption.*
* [**Mobile Security Audit & Hardening Guide**](docs/MOBILE_SECURITY_AUDIT.md)  
  *Analysis of Android and iOS cryptographic boundaries, reverse engineering protections, anti-hooking checks, and biometric authentication gates.*
* [**x402 Micropayment Attack Vectors & Mitigation**](docs/security/x402-attacks.md)  
  *Facilitator-layer hardening for HTTP 402 AI agent payment rails, closing cross-resource substitution (F1) and duplicate settlement race conditions (F2).*
* [**Transaction Security Standards**](docs/TRANSACTION_SECURITY.md)  
  *Formal verification of transaction signing, nonce isolation, and replay defense mechanisms.*

---

### 📱 3. Mobile, Wearable & Client Protocols (`docs/`)

* [**Self-Custodial Wallet Architecture**](docs/SELF-CUSTODIAL-WALLET-DESIGN.md)  
  *BIP-39, SLIP-0010 (Solana Ed25519), and BIP-44 (EVM Secp256k1) key generation, memory zeroization, and multi-account derivation.*
* [**Android Native Release & Signing Runbook**](docs/ANDROID-RELEASE.md)  
  *Google Play Console signing keys, APK optimization, and ProGuard/R8 obfuscation rules.*
* [**Google Play Store Signing Runbook**](docs/PLAY-SIGNING-RUNBOOK.md)  
  *Step-by-step key registration, upload keystores, and app integrity verification.*
* [**Certificate Pinning & Rotation Runbook**](docs/CERT-PIN-ROTATION.md)  
  *Zero-downtime HPKP / mTLS certificate rotation for mobile network security configs.*
* [**Hardware Wallet Research & Integration**](docs/HARDWARE-WALLET-RESEARCH.md)  
  *Tangem, Ledger, and NFC hardware-token signing integrations for high-net-worth accounts.*

---

### ⚡ 4. Protocols, Settlement & Custody Models (`docs/`)

* [**Production Go-Live Money Runbook**](docs/GO-LIVE-MONEY.md)  
  *Financial operation guidelines, liquidity provisioning, and treasury hot/warm/cold balance thresholds.*
* [**Ledger Reconciliation & Invariant Checks**](docs/RECONCILIATION.md)  
  *Continuous double-entry verification between off-chain balances and on-chain token states.*
* [**Balance Custody Mapping**](docs/BALANCE-CUSTODY-MAP.md)  
  *Taxonomy of self-custody vs. custodial escrow accounts across multi-chain rails.*
* [**Custody Disclosure & Regulatory Disclaimers**](docs/CUSTODY-DISCLOSURE.md)  
  *Compliant disclosure language for retail and institutional stablecoin holders.*
* [**API v1 Surface Specification**](docs/API-V1-SURFACE.md)  
  *Public REST & WebSocket interface contracts for merchant integrations.*
* [**Verified Protocol Claims**](docs/VERIFIED-CLAIMS.md)  
  *Benchmarked performance figures: sub-500ms settlement, 58,000+ edge requests, 0% error rate.*

---

### 🌐 5. Expansion Blueprints & Strategy (`reports/`)

* [**Singapore Expansion Master Blueprint**](reports/furlpay_singapore_expansion_master_blueprint.md)  
  *MAS Major Payment Institution (MPI) compliance roadmap, SGQR / PayNow integration, and Southeast Asian cross-border settlement.*
* [**USDC Travel Market & Tourism Rails**](reports/furlpay_singapore_usdc_travel_market_report.md)  
  *Stablecoin travel spend analysis, merchant discount rate (MDR) disruption, and hotel direct-settlement protocols.*
* [**USDC Travel Architecture & Security Blueprint**](reports/furlpay_usdc_travel_architecture_and_security_blueprint.md)  
  *Offline travel vouchers, MCC-locked virtual cards, and airline ticketing integration.*
* [**Algorithmic Settlement & Competitive Disruption**](reports/furlpay_algorithmic_settlement_competitive_report.md)  
  *Comparative analysis against Stripe, RedotPay, and traditional acquiring networks.*
* [**Partner Onboarding Master Report**](reports/furlpay_partner_onboarding_master_report.md)  
  *Technical guidelines for payment service providers (PSPs), independent sales organizations (ISOs), and payment gateways.*

---

## 🛠️ Technology Stack Breakdown

| Layer | Technologies & Protocols |
| :--- | :--- |
| **Cloud & Infrastructure** | **Amazon Web Services (AWS)**: EC2 `i4i.8xlarge`, ECS Fargate, CloudFront, ALB, Aurora PostgreSQL Serverless v2, ElastiCache for Valkey, AWS Nitro Enclaves, AWS KMS, Amazon SNS, CloudWatch, Terraform. |
| **Dedicated Blockchain Node** | **Agave v2.2.x** (Solana Validator Engine), **Yellowstone Dragon's Mouth gRPC Geyser**, Ubuntu 24.04 LTS, Nitro NVMe RAID-0, Linux tmpfs AccountsDB. |
| **Cross-Chain & Stablecoins**| **Circle USDC**, **Circle CCTP v2** (`MessageTransmitterV2`, `TokenMessengerMinterV2`), Circle Iris Attestation API, Circle Mint API, Circle Programmable Wallets, Circle Arc. |
| **Card Issuing Rails** | **Rain Cards** (Visa/Mastercard Principal Member), Sub-110ms Just-in-Time (JIT) Funding Webhooks, ISO 8583 Authorization Bridge. |
| **Backend Application** | **Next.js 15 (App Router)**, Node.js 22, TypeScript 5.5, Zod, Viem, `@solana/web3.js`, EIP-3009 Gasless Relay Engine, EIP-712 Typed Signing. |
| **Mobile Applications** | **React Native / Expo** (Android & iOS), **Kotlin 2.1** (Android & Wear OS Compose), **Swift 6.0** (iOS / Apple Watch), Google Play Integrity API, AndroidKeyStore, Apple Secure Enclave. |
| **Security & Cryptography** | Hardware-backed Biometric Authentication (Class 3 / Strong), Ed25519, Secp256k1, AES-256-GCM, Constant-Time HMAC-SHA256, Zeroization. |

---

## 🤝 Community & Support

- **GitHub Organization**: [github.com/FurlPay](https://github.com/FurlPay)
- **Developer Documentation**: [github.com/FurlPay/docs](https://github.com/FurlPay/docs)
- **Security Inquiries**: [security@furlpay.com](mailto:security@furlpay.com)

---

<div align="center">
  <sub>Copyright © 2026 FurlPay Inc. All rights reserved. Sovereign financial rails for the next billion users.</sub>
</div>
