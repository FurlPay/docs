# Corrected architecture

**Status: authoritative. Supersedes `Furlpay Production-Ready Financial
Architecture.png` at the repository root.**

That diagram draws an Indian fiat rail through Rain:

```
FURLPAY → RAIN → USER'S BANK → UPI / IMPS / NEFT / RTGS → INR LEDGER
```

Every arrow in that line is wrong, and they are wrong in different ways.

## What the diagram claims, and what is actually here

| Diagram claim | Reality in this repository |
|---|---|
| Rain is the on/off-ramp provider | **Rain does not appear in this codebase.** A whole-repo search for `\brain\b` (excluding `node_modules`) returns four files: two travel/venue datasets, the OFAC SDN name list, and the market ticker universe. No adapter, no env var, no route, no SDK, no webhook. |
| Rain → user's bank → UPI | **Rain is not and cannot be a UPI processor.** UPI participation requires an NPCI membership held by a bank or an RBI-authorised PSP. Rain is a Visa principal member for card issuing. Its public materials name no India corridor, no INR, and no Indian bank rails. Treat all of that as UNVERIFIED — do not assume. |
| IMPS / NEFT / RTGS | RBI/NPCI domestic rails, reachable only through an Indian scheduled commercial bank or an RBI-authorised entity. Nothing in this repository touches any of them. |
| INR ledger | `lib/rails/registry.ts` enumerates every rail FurlPay has: x402, MPP, CCTP, LI.FI, gasless transfers. All five are crypto; all five are `custody: "non-custodial"`. There is no fiat rail. |
| "UPI" in the app | `native-app/lib/upi.ts` parses a `upi://pay` QR and hands it to the user's own PSP app (GPay / PhonePe / Paytm / BHIM). **FurlPay never sees the money, holds no mandate, and needs no PSP licence for it.** That is exactly why it was buildable — and it is not a rail. |

The README has always been more accurate than the poster: *"an architecture
reference and prototype, not a licensed financial product."* Trust the README.

---

## Production architecture (mainnet)

Rain is out of the INR path. Custody is stated as what it is.

```
┌─ CUSTOMER (India) ──────────────────────────────────────────────────────────┐
│  Android (Expo RN) · Web · [iOS — NOT BUILT: no native-app/ios, no iOS      │
│  profile in eas.json]                                                       │
│  Passkey / WebAuthn · biometric · device key in SecureStore                  │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ HTTPS
┌────────────────────────────────▼────────────────────────────────────────────┐
│  EDGE          middleware.ts — session, rate limit, correlation id          │
├─────────────────────────────────────────────────────────────────────────────┤
│  API / AUTH / RISK / KYC / AML                                              │
│   authz · kycTiers · velocity · deviceIntel · stepup                        │
│   KYC:          Persona OR Sumsub — needs India CDD (PAN, OVD, CKYC)        │
│   AML:          OFAC SDN today. NEEDS Chainalysis/TRM + UNSC + MHA §51A     │
│                 + PEP + adverse media.                                      │
│   TRAVEL RULE:  interface only — travelRule.ts transmits NOTHING.           │
│   FLAGS:        flags.ts — four gates + FURLPAY_MONEY_KILL_SWITCH  ✅        │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │  every money movement posts here FIRST
┌────────────────────────────────▼────────────────────────────────────────────┐
│  DOUBLE-ENTRY LEDGER — POSTGRES ONLY (migration 0009)                       │
│  Balance checked inside the write; postings append-only by trigger;         │
│  balances derived; settle_payment_atomic holds a row lock.                  │
│                                                                             │
│  ASSETS                        LIABILITIES                                  │
│   1000 Bank / PA escrow         2000:<user> Customer fiat liability         │
│   1100 Stablecoin treasury      2100:<user> Customer stablecoin liability   │
│   1200 Provider receivable      2200 Merchant payable                       │
│   1300 In-transit (fiat)        2300 TDS payable   2400 GST payable         │
│   1400 In-transit (chain)      EQUITY / P&L                                 │
│   1500 Gas float                3000 Opening equity                         │
│                                 4000 Fee REVENUE   5000 Gas expense         │
│                                                                             │
│  In production, no durable backend ⇒ postings REFUSE (LedgerDurabilityError)│
└──────┬──────────────────────────────────────────────────┬───────────────────┘
       │ FIAT LEG (INR)  — NONE OF THIS EXISTS YET        │ CRYPTO LEG
┌──────▼──────────────────────────────┐   ┌───────────────▼────────────────────┐
│ INR COLLECTION                      │   │ STABLECOIN TREASURY                │
│ RBI-authorised PAYMENT AGGREGATOR,  │   │ Circle Mint / OTC counterparty     │
│ contracted, with explicit           │   │ Hot / warm / cold separation       │
│ VDA-merchant approval               │   │ Signing: KMS ONLY (lib/kms.ts,     │
│   UPI intent+collect · cards · NB   │   │   Turnkey implemented)             │
│   Virtual accounts (VAN + IFSC)     │   └───────────────┬────────────────────┘
│ FUNDS SIT IN THE PA's ESCROW AT A   │                   │
│ SCHEDULED BANK — NOT WITH FURLPAY   │   ┌───────────────▼────────────────────┐
├─────────────────────────────────────┤   │ ON-CHAIN (MAINNET)                 │
│ INR PAYOUT                          │   │ Arbitrum One 42161 — PRIMARY       │
│ Bank-sponsored payouts partner      │   │   USDC 0xaf88…5831 (Circle-verified)│
│   IMPS · NEFT · RTGS · UPI-P2A      │   │ Base 8453 · Ethereum 1 · Polygon 137│
│   penny-drop / VPA validation       │   │ Solana — SEPARATE custody design;   │
│   terminal status + webhook         │   │   Safe is EVM-only and cannot cover │
├─────────────────────────────────────┤   │   it. Not in CHAIN_REGISTRY today.  │
│ SPONSOR BANK                        │   │ Private RPC + failover per chain    │
│   current a/c + statement API       │   │ x402 / EIP-3009 settler  ✅ REAL     │
└──────┬──────────────────────────────┘   │ CCTP V2 · LI.FI          ✅ REAL     │
       │                                  └───────────────┬────────────────────┘
┌──────▼──────────────────────────────────────────────────▼───────────────────┐
│  THREE-WAY RECONCILIATION (daily, alerting)                                 │
│   BANK / PA statement  ⇄  LEDGER  ⇄  CHAIN INDEXER                          │
│   Divergence ⇒ halt withdrawals for that asset + page on-call               │
│   Today: only the LEDGER↔LEDGER half exists (lib/reconciliation.ts), and it │
│   now raises alerts from /api/cron/reconcile.                               │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                           ┌──────────▼──────────┐
                           │ MERCHANT / RECIPIENT│
                           │ USDC wallet · x402  │
                           └─────────────────────┘

OUT OF THE INR PATH, contract-gated, optional, later:
  RAIN → card issuing ONLY, non-INR markets, and only after written
         confirmation of India/INR eligibility. Rain cannot touch UPI,
         IMPS, NEFT or RTGS.
```

## Testnet / staging architecture

```
┌─ TEST USER ────────────────────────────────────────────────────────────────┐
│ Android internal track · localhost web · NO real money, ever               │
└──────────────────────────────┬─────────────────────────────────────────────┘
┌──────────────────────────────▼─────────────────────────────────────────────┐
│ STAGING   FURLPAY_ENV=staging  (NOT NODE_ENV — Next sets that to           │
│           "production" for preview builds too; see flags.ts:44)            │
│   SEPARATE Supabase project · SEPARATE Redis · SEPARATE keys               │
│   SEPARATE fee-payer. Nothing shared with production, ever.                │
│   INTEGRATION_MODE=mock; X402_ENFORCE_SIGNATURE=1 exercises the live       │
│   signature path without flipping the global flag.                        │
└──────┬─────────────────────────────────────────────┬───────────────────────┘
       │ FIAT (does not exist)                       │ CRYPTO (real testnet)
┌──────▼──────────────────────────┐   ┌──────────────▼────────────────────────┐
│ There is no fiat sandbox,       │   │ Arbitrum Sepolia 421614               │
│ because there is no fiat        │   │   USDC 0x75faf114eafb1BDbe2F0316DF893 │
│ partner. Do NOT simulate one    │   │        fd58CE46AA4d                    │
│ with fabricated account         │   │ Base Sepolia 84532                    │
│ numbers — that was the          │   │   USDC 0x036CbD53842c5426634e7929541e │
│ /api/banking defect, and the    │   │        C2318f3dCF7e   ← ADD to registry│
│ route now returns 503 in every  │   │ Ethereum Sepolia 11155111             │
│ environment rather than         │   │   USDC 0x1c7D4B196Cb0C7B01d743Fbc6116 │
│ inventing an IFSC.              │   │        a902379C7238   ← ADD to registry│
└─────────────────────────────────┘   │ Solana Devnet                         │
                                      │   USDC 4zMMC9srt5Ri5X14GAgXhaHii3GnPA │
                                      │        EERYPJgZJDncDU ← ADD to registry│
                                      │ Fee-payer: faucet-funded only         │
                                      │ Deploy the 6 contracts HERE first     │
                                      └───────────────────────────────────────┘

All USDC addresses above are from Circle's official contract-address
documentation. Never copy a token address from a blog.

HARD RULES:
  1. Separate keys, RPCs, tokens, databases and Redis. No shared secret.
  2. FURLPAY_ENV — never NODE_ENV — decides what may move money.
  3. A mock may never produce a settlement artifact in an env labelled prod.
  4. Local chain first (GO-LIVE-MONEY.md §3a), then PUBLIC testnet, then
     mainnet with a low cap. The public-testnet step has NOT been done: the
     2026-07-10 evidence is a local hardhat fork on chainId 421614.
```

## The seven flows

**A. INR → USDC → merchant** — does not exist. When built: PA collects into its
escrow; webhook posts `Dr 1000 escrow / Cr 2000:<user>`; §194S TDS posted to
2300; OTC converts; `Dr 1400 in-transit / Cr 2000`; on-chain transfer; `Dr 2200
merchant payable / Cr 1400`. A failure at any step leaves the money in the PA
escrow and refunds to source. The customer's INR is never converted before the
ledger records the liability.

**B. USDC → INR → Indian bank** — does not exist. When built: confirmations
observed; `Dr 1100 treasury / Cr 2100:<user>`; AML + Travel Rule; OTC converts;
`Dr 1300 in-transit fiat / Cr 2100`; payout partner credits a penny-drop-verified
beneficiary; `Dr 2000 / Cr 1300` on confirmation. A failure after conversion
leaves INR in 1300 and is retried or returned — **never silently re-credited as
USDC**.

**C. USDC → USDC merchant payment** — ✅ **works today.** x402 / EIP-3009, gas
sponsored, non-custodial, no fiat leg, no bank, no PA. **Launch this first.**

**D. Card payment** — blocked on an issuer, a BIN sponsor and a PCI decision.
`CARD_ISSUING_LIVE` carries an `unimplemented` reason, so no env var can turn it
on; every card route returns 503 in production.

**E. Crypto withdrawal** — self-custody: the user signs, FurlPay screens both
parties and broadcasts. Turnkey path: **custodial** — needs a withdrawal
allowlist and a time-lock before it carries real value.

**F. Crypto deposit** — `webhooks/deposit-detect` is real (HMAC + timestamp
freshness + optional IP allowlist + per-transaction ordering mark). Still needs
confirmation depth scaled by amount and a source-wallet risk screen.

**G. Cross-chain USDC** — CCTP V2 builds calldata, the user signs, the Iris
attestation is polled. Needs an attestation-timeout policy.

## What changed in the code on 6 Sep 2026

- `/api/banking` no longer fabricates payable account details. It returned a
  routing number attributed to Cross River Bank, a Wise IBAN/SWIFT pair, a Wise
  sort code, and `UPI: ashutosh@furlpay` with IFSC `YESB0000001` — with no
  production guard at all. It now returns 503 in every environment.
- The Bridge, Wise and Marqeta webhook endpoints are deleted.
- Card issuing, ordering, activation, reissue, PIN, reveal, device-wallet
  provisioning and spend are all gated behind `CARD_ISSUING_LIVE`.
- The chart of accounts above replaces `1000:<user> user cash (asset)` /
  `2000 external world` / `4000 fees (expense)`.
- Ledger postings are durable-first; production refuses without Postgres.
- The x402 settler resolves a KMS-backed account, refuses a raw env key in
  production, and is gated on `ONCHAIN_SETTLEMENT_LIVE`.
- `safeAddress` is no longer presented as a deposit address anywhere.
- Both parties are sanctions-screened on `/api/wallets/transfer`.
- Replaced transactions are followed; abandoned settlements are terminal.
- A webhook cannot demote a verified user's KYC.
- `/api/cron/reconcile` raises alerts on ledger imbalance.
- `npm run deploy:prod` refuses to deploy a SHA that CI has not passed.

## What no amount of code will fix

FIU-IND registration · a Payment Aggregator contract with explicit VDA-merchant
approval · a bank-sponsored payouts partner · a sponsor bank's written
acceptance of VDA flows · an escrow account under the RBI PA Directions, 2025 ·
INR⇄USDC liquidity · Chainalysis/TRM · a Travel Rule vendor · a card issuer and
BIN sponsor · a smart-contract audit · a penetration test · an AML officer ·
§194S/GST tax infrastructure · a DPDP assessment.

**The Payment Aggregator contract is the hardest item and it is not technical.
Most Indian PAs decline VDA merchants. Budget 6–12 months and expect refusals.
If it cannot be cleared, the INR product does not exist and no engineering will
make it exist.**
