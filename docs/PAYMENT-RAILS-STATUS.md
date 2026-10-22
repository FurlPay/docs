# Payment Rails and Partner Status

> **Status, 22 October 2026.** Taken from the application's feature-flag registry and source code at the production commit. A rail is **LIVE** only if it is enabled on furlpay.com. **Signed agreements are listed only where one exists; there are none for card issuing, banking or fiat ramps.** Status labels are defined in the [enterprise architecture](ENTERPRISE-ARCHITECTURE-2026-2027.md).

Every money capability sits behind a feature flag. A flag marked *unimplemented* in the registry cannot report `on`, whatever the environment says, so "off because nobody enabled it" and "off because there is nothing to enable" are never confused.

| Rail | What exists | Status | External dependency |
| :--- | :--- | :--- | :--- |
| **Solana USDC** (merchant checkout) | Single-use destinations, finalized-commitment scan, Solana Pay reference keys, sanctions screening, atomic credit with ledger and webhook outbox | **IMPLEMENTED, switched off** in production | Production RPC capacity; enabling settlement assets |
| **EVM USDC** (Base, Arbitrum, Polygon, Ethereum, Arc) and **Sui USDC** | Same model; per-chain confirmation depth from a reviewed deposit policy; endpoints must prove chain identity | **IMPLEMENTED, switched off** | As above |
| **Circle CCTP V2** cross-chain USDC | Burn, Iris attestation tracking, destination mint; on-chain indexer of CCTP events | **IMPLEMENTED** | Circle Iris API (public) |
| **x402** (HTTP 402 agent payments) | Facilitator verifying EIP-3009 authorizations, resource binding, single-use settlement records | **IMPLEMENTED** | — |
| **MPP-style machine payments** | A paid FX endpoint using the same HTTP 402 pattern | **IMPLEMENTED** | — |
| **Agent-initiated payments** | Agent identity and delegation (KYA), mandates, limits, human approval | **IMPLEMENTED, TESTED** | — |
| **Payroll and payouts** | Payroll drafts, schedules, contractor self-service, fee ledger | **IMPLEMENTED** | Settlement assets enabled |
| **Treasury** | Settlement workspace with business onboarding and compliance cases | **IMPLEMENTED** | — |
| **Card issuing** | Card authorization design, an issuer adapter seam and a [JIT rails plan](VIRTUAL-CARD-JIT-RAILS-PLAN.md) | **BLOCKED**: flag marked unimplemented | **No card issuer contracted** |
| **Banking (ACH, SEPA, others)** | — | **BLOCKED**: flag marked unimplemented | No banking partner |
| **Fiat on-ramp and off-ramp** | Provider adapters and sandbox paths | **BLOCKED**: flags marked unimplemented | No provider contracted |
| **Bank-network rail (Kinexys)** | Adapter tested against J.P. Morgan's public mock API; production access disabled in code | **RESEARCH** | No production credential or banking relationship |
| **Cross-border transfers** | Achievable on-chain between stablecoin wallets; no fiat leg without a ramp partner | **IMPLEMENTED** (on-chain leg only) | Ramp partner for fiat legs |

**What may be said externally:** that FurlPay has built and tested these rails in software, and which are switched off. **What may not be said:** that FurlPay issues cards, holds banking relationships, offers fiat ramps, or has partner agreements for any of them. The [verified-claims register](VERIFIED-CLAIMS.md) governs wording.
