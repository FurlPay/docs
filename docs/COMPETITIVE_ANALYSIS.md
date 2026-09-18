# Competitive teardown — RedotPay, Tangem, FurlPay

**Date:** 6 September 2026
**Method:** RedotPay from decompiled APK (jadx + apktool + Dart AOT string extraction from `libapp.so`). Tangem from its **public source repositories** (`reference/tangem-app-android`, `reference/tangem-app-ios`) cross-checked against the shipped APK. FurlPay from this repository.

**Benchmarking only.** Nothing here is a licence to copy an implementation. The value is in the *shape* of what they built and what that implies we are missing.

---

## Correction to my earlier assessment

In an earlier session I told you, based on the **shipped Tangem APK**:

> "Tangem remains self-custody and does not operate bank accounts/cards. No custody, no cards, no bank accounts."

**That was wrong.** The shipped APK's Java layer showed only MoonPay and Express, so I concluded there was no card product. The **source repository** shows otherwise:

| Module | Files | What it is |
|---|---:|---|
| `features/tangempay/` | **292** | Full card product: account details, cashback, transactions, withdrawals, card view |
| `domain/visa/` | **147** | Visa card activation, PIN encryption, auth challenges, wallet-data signing |
| `features/virtual-accounts/` | 34 | Virtual account, add-funds flow |
| `domain/virtual-account/` | 6 | Eligibility, suitable-wallet selection |
| `features/kyc/` | 9 | Sumsub SDK integration (`TangemSNSTheme` — SNS is Sumsub's SDK prefix) |
| `domain/offramp/` | 5 | Off-ramp URL + pending state |
| iOS `TangemVisa/` | 69 Swift | Activation, API service, auth-token handler, payment-account interactor |
| iOS `Modules/TangemPay/` | 78 Swift | Authorization, availability, customer, KYC, order, networking |

**Tangem built a card + KYC + virtual-account product on top of a self-custody wallet.** That is much closer to FurlPay's ambition than I reported, and it is the single most important input to this comparison.

---

## Scale

| | RedotPay | Tangem | FurlPay |
|---|---:|---:|---:|
| Architecture | Flutter (Dart AOT), NetEase-packed | Kotlin (Android) + Swift (iOS), modular | Next.js + Expo RN |
| API endpoints | **619** | server-driven (Express aggregator) | **302** |
| UI files | **1,338** Dart views | ~46 feature modules × N | **57** screens + 65 components |
| Feature/domain modules | 32 view areas | **46 features / 52 domains** | ~101 lib modules |
| Native libs | 30 `.so` (incl. 31 MB `libapp.so`) | TangemSdk + blockchain libs | none |
| **iOS app** | ✅ shipping | ✅ shipping (full parity) | ❌ **no build config** |

RedotPay's 619 endpoints break down as: `user` 368 (card 59, security 51, asset 43, vba 36, creditV2 21, earn 18, loan 15, membership 14), `otc` 64, `send` 29, `remittance` 26.

---

## Feature matrix

Legend: ✅ shipping · ◐ partial · ⬜ absent · **n/a** not in that product's scope

### Identity & security

| Capability | RedotPay | Tangem | FurlPay | Gap |
|---|:--:|:--:|:--:|---|
| Passkey / WebAuthn | ✅ 9 endpoints | ✅ | ✅ | — |
| Hardware-key auth (bind/challenge/verify) | ✅ 4 endpoints | ✅ NFC card | ◐ interface only | **Signer registry exists; no NFC/BLE implementation** |
| Biometrics toggle | ✅ | ✅ | ✅ | — |
| Google Authenticator / TOTP | ✅ | — | ✅ | — |
| **Anti-phishing code** | ✅ save/get | ⬜ | ⬜ | **Missing** — user-set phrase shown in comms |
| Liveness / face check | ✅ 3 endpoints | ✅ via Sumsub | ⬜ | Vendor-blocked |
| KYC provider | ✅ Sumsub | ✅ Sumsub | ◐ webhook only | Vendor-blocked |
| **EDD / CRA questionnaire** | ✅ `vba/cra/*`, 9 `edd` views | ⬜ | ⬜ | **Missing** |
| Device fraud SDK | ✅ ThreatMetrix | ⬜ | ◐ own collector | Hybrid decision |
| App-integrity / anti-tamper | ✅ NetEase packer | ✅ `DeviceSecurityInfoProvider` | ◐ `deviceIntegrity.ts` | **No packer, no attestation** |
| **Transaction simulation before signing** | ⬜ | ✅ **Blockaid** | ⬜ | **Missing — I flagged this in our own audit** |
| dApp safety check | ⬜ | ✅ Blockaid | ⬜ | Missing |

### Cards

| Capability | RedotPay | Tangem | FurlPay | Gap |
|---|:--:|:--:|:--:|---|
| Card issuance | ✅ | ✅ Visa activation SM | ⬜ gated | **Issuer-blocked** |
| Card activation state machine | ✅ | ✅ 8-state | ⬜ | **Code gap** — copyable pattern |
| PIN set / verify / formats | ✅ 3 endpoints | ✅ encrypted PIN | ⬜ | Code + issuer |
| Freeze / unfreeze | ✅ | ✅ | ◐ local only | Issuer-blocked |
| Limits (get/change) | ✅ | ✅ | ◐ local only | Issuer-blocked |
| 3DS | ✅ 2 endpoints | ✅ | ◐ designed | Issuer-blocked |
| **Apple Pay provisioning** | ✅ `ifSupportApplePay` | ✅ | ⬜ TODO | **Entitlement + issuer** |
| **Google Pay provisioning** | ✅ `registerGooglePay` + `tapandpay.issuer` | ✅ | ⬜ TODO | **Entitlement + issuer** |
| Physical card shipping | ✅ 5 endpoints | ✅ | ⬜ | Code + issuer |
| Card themes / skins | ✅ | ✅ | ◐ gradient only | Cosmetic |
| Report loss / reissue | ✅ | ✅ | ◐ | Code gap |
| Cashback | ✅ rebate | ✅ `CashbackBlock` | ◐ rewards | Partial |
| Statement export | ✅ `bill/export` | ✅ | ⬜ | **Missing** |

### Money movement

| Capability | RedotPay | Tangem | FurlPay | Gap |
|---|:--:|:--:|:--:|---|
| On-ramp | ✅ Paybis/dLocal/Mesh | ✅ Express aggregator | ◐ CDP + demo | **Provider-blocked** |
| Off-ramp | ✅ | ✅ | ◐ quote only | **Provider-blocked** |
| **Virtual bank account (IBAN)** | ✅ 36 endpoints | ✅ 40 files | ⬜ 503 | **Partner-blocked** |
| **Remittance / cross-border** | ✅ 26 endpoints, dynamic recipient fields | ⬜ | ⬜ | **Missing entirely** |
| SWIFT / MT103 | ✅ `MT103Info.dart` | ⬜ | ⬜ | **Missing** |
| Deposit recall handling | ✅ `FiatDepositRecallDetail` | ⬜ | ◐ lifecycle models it | Partial |
| Crypto deposit / withdraw | ✅ | ✅ | ✅ | — |
| Swap | ✅ | ✅ Express | ✅ LI.FI | — |
| **P2P / OTC marketplace** | ✅ 64 endpoints, 131 views | ⬜ | ◐ in-memory | **Not durable** |
| Exchange transfer (Binance/Bybit) | ✅ dedicated | ⬜ | ⬜ | Missing |
| Agentic payments / authorizations | ✅ `me/authorizations/*`, `mpp_charges` | ⬜ | ✅ x402 + MPP | **FurlPay ahead** |

### Financial products

| Capability | RedotPay | Tangem | FurlPay | Gap |
|---|:--:|:--:|:--:|---|
| Earn / yield | ✅ 18 endpoints | ✅ `yield-supply` 57 files | ◐ Morpho gated | Partial |
| Staking | ⬜ | ✅ 86 files | ⬜ | Missing |
| **Credit / collateralised lending** | ✅ 21 endpoints + risk engine | ⬜ | ⬜ screen only | **Missing** |
| Loan | ✅ 15 endpoints | ⬜ | ⬜ | Missing |
| Membership tiers | ✅ 14 endpoints | ⬜ | ◐ | Partial |
| NFT | ⬜ | ✅ 23 files | ⬜ | n/a |
| Prediction markets | ⬜ | ✅ Polymarket 62 files | ⬜ | n/a |
| Investing / equities | ⬜ | ⬜ | ✅ | **FurlPay ahead** |
| Travel booking | ⬜ | ⬜ | ✅ | **FurlPay unique** |

### Platform

| Capability | RedotPay | Tangem | FurlPay | Gap |
|---|:--:|:--:|:--:|---|
| **iOS app** | ✅ | ✅ | ❌ | **No `native-app/ios`, no iOS EAS profile** |
| Android app | ✅ | ✅ | ◐ unpublished | Play upload pending |
| Wear OS | ⬜ | ⬜ | ✅ | FurlPay unique |
| Web app | ⬜ | ⬜ | ✅ | FurlPay unique |
| Browser extension | ⬜ | ⬜ | ✅ | FurlPay unique |
| In-app chat / support | ✅ Intercom + RongCloud | ✅ Usedesk | ◐ own | Partial |
| **AI agent support** | ✅ 10 endpoints | ⬜ | ◐ copilot | Partial |
| Push notifications | ✅ | ✅ | ✅ | — |
| Referral | ✅ | ✅ | ✅ | — |
| Address book | ✅ | ✅ | ✅ | — |
| Multi-language | ✅ 9 locales | ✅ | ◐ | Partial |

---

## What FurlPay is missing — ranked by whether we can fix it

### A. Pure code gaps — no external dependency, buildable now

| # | Gap | Evidence | Effort |
|---|---|---|---|
| A1 | **iOS build configuration** | `native-app/ios` does not exist; `eas.json` has android-only profiles. `furlpay-ios/` is a 28-file Solana Pay demo, not the app | L |
| A2 | **Transaction simulation before signing** | Tangem uses Blockaid; our own audit flagged `/api/mpc/sign` accepts an opaque digest | M |
| A3 | **Card activation state machine** | Tangem's 8-state model is a clean pattern; ours has no state machine at all | M |
| A4 | **Anti-phishing code** | RedotPay `saveAntiPhishingCode`/`getAntiPhishingCode`. Cheap, high-value anti-social-engineering control | S |
| A5 | **P2P durability** | Ours is `globalThis`; RedotPay has 64 endpoints and escrow | M |
| A6 | **Statement / bill export** | Both competitors have it; regulators expect it | S |
| A7 | **EDD questionnaire flow** | RedotPay `vba/cra/*` + 9 `edd` views. Required by FIU-IND guidelines | M |
| A8 | **Hardware-wallet signer implementation** | Our `SignerPicker` advertises NFC/BLE; no implementation behind it | L |
| A9 | **App attestation / anti-tamper** | Both competitors ship it; we have `deviceIntegrity.ts` only | M |
| A10 | **Deposit recall / reversal UI** | RedotPay models it explicitly; our lifecycle has the states, no surface | S |

### B. Partner-blocked — code can be built, cannot be switched on

| Gap | Blocked on |
|---|---|
| Card issuance, PIN, 3DS, freeze, limits | Issuer + BIN sponsor |
| Apple Pay / Google Pay provisioning | Issuer entitlement + Apple/Google approval |
| Virtual bank account (IBAN) | Banking partner |
| On-ramp / off-ramp execution | Contracted provider |
| Remittance corridors | Bank-sponsored payout partner |
| Liveness / document verification | KYC vendor |
| Credit / lending | Capital + licence |

### C. Deliberately not doing

Staking, NFT, prediction markets. Tangem has all three; none is on the path to a payments platform.

---

## The three lessons worth taking

**1. Tangem's modular boundary is better than ours.** 46 feature modules and 52 domain modules with `api` / `impl` / `mock` splits per feature. `features/kyc/mock` exists as a first-class module — the mock is a *build target*, not a runtime branch. Our `INTEGRATION_MODE=live` is a global runtime flag; theirs is a compile-time swap. Theirs cannot leak a mock into production.

**2. RedotPay's façade pattern is the one we already chose.** Every provider (Paybis, dLocal, Mesh, Binance Pay, Bybit) sits behind RedotPay's own `/api/v1/payin/onramp/*` and `/api/v1/onramp/*`. Their own order ids, their own state. That is exactly `lib/providers/types.ts`. Confirmation we picked right.

**3. Both ship transaction-safety we don't have.** Tangem uses Blockaid for pre-signature simulation and dApp checks. RedotPay ships ThreatMetrix plus an anti-tamper packer. Our Trust Layer is arguably better-designed than either at the *scoring* layer — and it has no data feeding it, and no simulation in front of the signer.

---

## Where FurlPay is genuinely ahead

Not many places, and worth being precise:

- **Agentic payments.** x402 / EIP-3009 with a real settler, confirmation-depth gating, replacement handling. RedotPay has `me/authorizations/*` and `mpp_charges` — the same idea — but ours is the more complete protocol implementation.
- **Ledger correctness.** Neither competitor's client reveals a double-entry ledger with evidence-gated state transitions. Ours refuses to call an HTTP 200 a settlement. That is invisible to a user and decisive to an auditor.
- **Provider independence by construction.** Capability matrix gated on signed contracts; failover that cannot double-pay.
- **Surface breadth.** Web + extension + Wear OS + travel. Neither competitor has any of these.

**But:** they move real money in 90+ countries and we move none. Architecture quality does not settle a payment.

---

## Recommended order

1. **A1 — iOS.** The single largest product gap. Both competitors ship iOS; we have no build.
2. **A2 — transaction simulation.** Security gap we already identified independently.
3. **Populate the Trust Layer graph.** Built and unfed (see `docs/TRUST_LAYER.md`).
4. **A4, A6, A10** — small, high-value, no dependencies.
5. **A3, A7** — state machine and EDD; needed the day a partner signs.
6. **A5, A8, A9** — larger, lower urgency.
