# Forensic audit — Tangem, RedotPay, FurlPay

**Date:** 4 September 2026
**Phase:** 1 — audit only. No production code changed.
**Method:** source-code tracing where source exists; artifact inspection where it does not.

---

## 0. Evidence quality — read this before trusting anything below

The brief assumed both reference implementations were available as source. Only one is.

| Reference | Location | Form | Can it be source-audited? |
|---|---|---|---|
| **Tangem Android** | `reference/tangem-app-android` | Kotlin source, ~290 modules, 8,179 files | **Yes** — full source |
| **Tangem iOS** | `reference/tangem-app-ios` | Swift source, 6,844 files | **Yes** — full source |
| **RedotPay** | `~/Desktop/APK_Decompile/redotpay` | Decompiled APK | **No** — see below |

### Why RedotPay cannot be source-audited

Traced, not assumed:

1. `source/sources/com/redotpay/` contains **exactly one file**: `R.java`, the generated resource-ID class. No business logic.
2. Total Java files in the decompile: **32**. Of those, 29 are the NetEase SDK, 1 is `R.java`, 1 is an obfuscated stub.
3. `resources/assets/flutter_assets/` is present → **RedotPay is a Flutter application.** All business logic is Dart.
4. `flutter_assets/kernel_blob.bin` is **absent** → this is a **release AOT build**. Dart is compiled to native machine code, not to any recoverable intermediate.
5. The APK contains **no `libapp.so` and no `libflutter.so`** — it is the base APK of a split bundle; the Dart machine code is not even in this artifact.

**Conclusion: there is no RedotPay business logic to read.** No account model, no ledger, no authorization-hold implementation, no settlement state machine. Any claim in this document about RedotPay's *internal* architecture would be fabrication, so none is made.

### What RedotPay evidence *does* support

Observable from the manifest and bundled assets only — client-side posture, nothing about the backend:

| Signal | Artifact | What it evidences |
|---|---|---|
| **fingerprintjs** | `assets/com/fingerprintjs` | Device fingerprinting on the client, for fraud/risk correlation |
| **NetEase Yidun** | `com/netease` (29 files), `nedata.db`, `nedig.properties`, `assets/motion/libxt4.0.1_*.so` | Third-party risk-control / anti-fraud / captcha stack, with native libraries |
| **On-device liveness** | `face_detection_short_range.tflite`, `assets/motion/` | KYC liveness capture runs on-device |
| **On-device doc scan** | `mlkit_barcode_models` | Document/barcode capture on-device |
| **AppsFlyer** | `assets/com/appsflyer` | Attribution SDK |
| **In-app AI support** | `flutter_assets/packages/gpt_markdown`, `flutter_chat_ui` | LLM-backed support surface |

The single transferable inference: **RedotPay treats the device as an untrusted participant and fingerprints it**, and pushes KYC capture to the device while (necessarily) verifying server-side. That is relevant to FurlPay's device-binding question in §17 of the brief and to nothing else.

**A compensating substitute was found.** `reference/tangem-app-android/libs/visa/` and `domain/visa/` contain a readable card-payment integration with balances, spend limits, OTP thresholds and scheduled limit changes. This is the closest *readable* analogue to the RedotPay domain the brief wanted, and §3 below uses it in RedotPay's place.

---

## 1. The two questions the brief ends on

### "What does Tangem understand about protecting signing authority?"

Four properties, each traced to source:

**1. The application layer cannot produce a signature — only request one.**

`app/src/main/java/com/tangem/tap/domain/TangemSigner.kt` implements the `TransactionSigner` interface, but every method body delegates to `tangemSdk.startSessionWithRunnable(...)`. The private key is on the card. The app has no code path that produces a signature; it has a code path that *asks*, and the answer arrives asynchronously from a session it does not control.

**2. The request is bound to a specific key holder.**

`TangemSigner` takes `cardId` in its constructor and passes it to every session. The app cannot silently obtain a signature from a different card than the one intended.

**3. The signing authority counts its own use, and the counter is not writable by the requester.**

```kotlin
data class TangemSignerResponse(
    val totalSignedHashes: Int?,
    val remainingSignatures: Int?,
    ...
)
```

The **card** maintains `totalSignedHashes`. The app receives it and cannot reset it. This makes unauthorised use *detectable after the fact*: if the counter advances more than the app's own records explain, something signed that the app did not ask for.

This is the deepest idea in the codebase and the one FurlPay is furthest from having.

**4. Authorization is separated from construction.**

`sign(hashes: List<ByteArray>, publicKey)` — the signing authority receives opaque digests. It does not parse, interpret or validate the transaction. Construction is the app's job; authorization is the card's. (The known cost is blind signing; Tangem's mitigation is the UI confirmation and `initialMessage` shown during the NFC session — a human-readable statement of what is being approved, presented by a layer the transaction builder does not control.)

### "What does the card-payment lifecycle understand about authorization → reservation → settlement?"

From `libs/visa/src/main/kotlin/com/tangem/lib/visa/model/VisaContractInfo.kt` — the readable substitute for RedotPay:

```kotlin
data class Balances(
    val total: BigDecimal,
    val verified: BigDecimal,
    val available: Available,
    val blocked: BigDecimal,      // ← authorization holds
    val debt: BigDecimal,
) {
    data class Available(
        val forPayment: BigDecimal,
        val forWithdrawal: BigDecimal,
        val forDebtPayment: BigDecimal,
    )
}

data class Limits(
    val spendLimit: Limit,                  // ← carries limit AND spent
    val noOtpLimit: Limit,                  // ← step-up threshold
    val singleTransactionLimit: BigDecimal,
    val expirationDate: Instant,
    val spendPeriodSeconds: BigInteger,     // ← window is data, not constant
) {
    data class Limit(val limit: BigDecimal, val spent: BigDecimal)
}

// and on the parent:
val oldLimits: Limits
val newLimits: Limits
val limitsChangeDate: Instant
```

Five patterns, each of which FurlPay currently collapses:

1. **Balance is never one number.** `total`, `verified`, `available`, `blocked`, `debt` are distinct. FurlPay has a single running spend counter.
2. **Availability is purpose-scoped.** Money available *to pay* is not the same quantity as money available *to withdraw*. FurlPay has no such distinction and will need one the moment refunds exist.
3. **A limit carries both its ceiling and its consumption.** `Limit(limit, spent)` makes `spent <= limit` checkable from the record itself. FurlPay stores only the running total, so the invariant is only checkable against a *separate* policy lookup — and if the two disagree, nothing notices.
4. **The step-up threshold is a limit like any other.** `noOtpLimit` — below it, no OTP; above it, human interaction. That is the human-approval threshold expressed as data rather than as a branch.
5. **Limit changes are scheduled, not instantaneous.** `oldLimits`, `newLimits`, `limitsChangeDate` — both versions are visible simultaneously. This is the structural answer to the brief's "race between authorization and revocation": a limit change with an effective date is not a race, because both readings are well-defined at every instant.

---

## 2. Architecture matrix

`ADOPT` = take the pattern. `ADAPT` = take the principle, different mechanism. `REJECT` = deliberately not applicable.

| Capability | Tangem (source-traced) | RedotPay (artifact only) | FurlPay (today) | Action |
|---|---|---|---|---|
| **Asset identity** | `CryptoCurrency.Coin`/`.Token` — network, decimals, contractAddress all on the type; validated in `init` | — | `asset: string` + separate `network: string`; `USDC_DECIMALS = 6` global | **ADOPT** — P0 |
| **Amount representation** | On-chain `BigInteger` atomic → `BigDecimal` for display via `movePointLeft(decimals)`. Never float. | — | Mixed: atomic strings in payment-intent, `number` cents in agentPolicy, `number` minor units in ledger | **ADAPT** — P0 |
| **Unknown value** | `Amount.value: BigDecimal?` — null is a real state | — | payment-intent has UNKNOWN; UsageEvent (planned) did not | **ADOPT** |
| **Signing authority** | Isolated in card + SDK session; app can only request | — | Ed25519 agent keys; FurlPay holds only public half | **already aligned** |
| **Authority use counter** | `totalSignedHashes` maintained by the card | — | none | **ADOPT** — P1 |
| **Authority binding** | `cardId` on every session | — | keyId on signature verify | **already aligned** |
| **Balance decomposition** | `total`/`verified`/`available`/`blocked`/`debt` | — | single spend counter | **ADOPT** — P0 |
| **Purpose-scoped availability** | `forPayment`/`forWithdrawal`/`forDebtPayment` | — | none | **ADAPT** — P2 |
| **Limit shape** | `Limit(limit, spent)` together | — | counter only, policy separate | **ADOPT** — P0 |
| **Step-up threshold** | `noOtpLimit` as data | — | trading engine only (orphaned, wrong domain) | **ADAPT** — P1 |
| **Limit change** | `oldLimits`/`newLimits`/`limitsChangeDate` | — | instantaneous overwrite | **ADOPT** — P1 |
| **Spend window** | `spendPeriodSeconds` as data | — | hardcoded UTC day | **ADOPT** — P2 |
| **Module boundary** | API/Impl split; impl unreachable across modules | — | packages exist but `x402-guard`, `settlement.releaseDecision` unreferenced | **ADOPT** — P1 |
| **Device risk** | — | fingerprintjs + NetEase Yidun + on-device liveness | none | **REJECT** — FurlPay's caller is an agent, not a phone |
| **KYC capture** | — | on-device tflite liveness | out of product boundary | **REJECT** |
| **Attribution SDK** | AppsFlyer present | AppsFlyer present | n/a | **REJECT** |
| **Card issuing** | Visa integration | core product | out of boundary | **REJECT** |
| **NFC / hardware** | central | — | n/a | **REJECT** — principle already extracted above |

---

## 3. P0 finding: asset identity is structurally unsound in FurlPay

**Severity:** P0 — money correctness
**Status:** confirmed by inspection, not yet exploited by a test

### Evidence

`packages/gateway/src/protocol.ts`:

```ts
asset: string;                    // lines 96, 131, 342 — three separate declarations
export const USDC_DECIMALS = 6;   // a global constant
export const NETWORK_USDC: Record<string, string> = {
  arbitrum: "0xaf88...", "arbitrum-sepolia": "0x75fa...",
  base: "0x8335...",     "base-sepolia": "0x036C...",
  polygon: "0x3c49...",  solana: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
};
```

Three defects follow:

1. **`solana-devnet` is absent from the map**, while both EVM testnets are present. `examples/agent-paid-api/src/config.ts` therefore hardcodes `DEVNET_USDC_MINT = "4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU"` independently — **two sources of truth for the same fact**, which is precisely how a mainnet mint reaches a devnet code path.

2. **Asset and network are sibling strings.** `{ asset: "EPjFWdd5…" /* mainnet */, network: "solana-devnet" }` is representable and typechecks. The brief's requirement — "make the system incapable of accidentally treating mainnet USDC and devnet USDC as the same asset" — is currently unmet at the type level.

3. **Decimals are global.** `USDC_DECIMALS = 6` is correct for USDC and wrong for any 18-decimal asset. The system cannot currently represent a second asset correctly.

### Tangem's answer

`CryptoCurrency.Token` carries `network`, `decimals` and `contractAddress` **on the type**, with `init { checkProperties(); require(contractAddress.isNotBlank()) }`. A token that disagrees with its own network cannot be constructed. `CryptoCurrency.ID` is composite (prefix + body + suffix) with the components `private` — the string form is *derived*, so it cannot be forged by concatenation.

### Recommended fix

```ts
interface AssetId {
  readonly chain: string;      // "solana" | "base" | …
  readonly network: string;    // "mainnet-beta" | "devnet" | …
  readonly address: string;    // mint / contract — never a symbol
  readonly decimals: number;   // per-asset, never global
  readonly symbol: string;     // DISPLAY ONLY, never compared
}
```

with a single constructor that validates, a registry keyed by `chain:network:address`, and equality defined over `(chain, network, address)` — never over `symbol`.

**Regression test that must fail without the fix:** constructing a payment whose asset address belongs to a different network than the payment's network must throw.

---

## 4. P0 finding: three money representations

| Location | Type | Unit |
|---|---|---|
| `packages/payment-intent` | `string` | integer atomic |
| `packages/ledger` | `number` | integer minor units |
| `apps/web/lib/agentPolicy.ts` | `number` | cents |
| `packages/agent-trust` | `number` (Micros) | micros |
| `packages/settlement` | `number` | USD |

None is floating-point-corrupt today — `ledger` validates `Number.isInteger`, `agentPolicy` uses `Math.round(x*100)`. The defect is **conversion at every boundary**, and `Math.round(amountUsd * 100)` in `authorizeAgentSpend` *is* a float operation on the way in: `Math.round(0.29 * 100)` is 29, but the general pattern is exactly how cents are lost.

`number` also caps exact integers at 2^53 ≈ 9.007e15 — 9 billion USDC at 6 decimals. Acceptable now; not acceptable as a stated ledger guarantee.

**Canonical decision (see `money-model.md`):** integer atomic units as decimal strings at every persistence and API boundary; `BigInt` for arithmetic; a formatter for display. This matches Tangem's `BigInteger` on-chain → `movePointLeft(decimals)` for display, adapted to a language without `BigDecimal`.

---

## 5. P1 finding: security assets exist but are not on the running path

Confirmed by grep across the tracked tree:

- **`packages/x402-guard`** — 809 lines, 38 tests, published to npm, implements the defences for all five flaw classes in arXiv:2605.30998. **Imported by no running code.**
- **`packages/settlement` `releaseDecision`** — 52 tests, decides when it is safe to release a resource given chain finality. **Called by no running code.**

Tangem's module system makes this class of mistake hard: `features:foo:api` / `features:foo:impl` split means an implementation is *unreachable* from another module by construction, so wiring is forced to be explicit. FurlPay's flat package layout permits a fully-tested security module to sit inert.

This is the cheapest high-value work available: two modules, already finished and tested, currently protecting nothing.

---

## 6. P1 finding: the authority-use counter has no FurlPay equivalent

Tangem's card reports `totalSignedHashes`. FurlPay's budget store reports a running total that the *caller* causes and can, in principle, reconcile against nothing.

**Proposed:** `BudgetEnvelope` issues a monotonic `reservationSeq` per agent, maintained by the durable store. Every `UsageEvent` carries it. A gap or a duplicate in the sequence is then detectable evidence of a reservation that was issued but never recorded — i.e. exactly the crash-between-reserve-and-record case in the brief's failure matrix, made *observable* rather than merely handled.

---

## 7. Reject list, with reasons

| Pattern | Source | Why rejected |
|---|---|---|
| Device fingerprinting | RedotPay | FurlPay's caller is a server-side agent with a signing key. Fingerprinting a datacentre is noise. |
| On-device KYC/liveness | RedotPay | Outside the product boundary — FurlPay does not onboard consumers. |
| NFC / hardware element | Tangem | No hardware in FurlPay's path. The *principle* (authority isolated from the app layer) is adopted; the mechanism is not. |
| Blind hash signing | Tangem | FurlPay's agent authorization binds method, path, query, body and amount. Signing an opaque digest would be a regression. |
| Card issuing / Visa rails | Both | Explicit non-goal. |
| Attribution SDKs | Both | No consumer acquisition surface. |
| `BigDecimal` domain type | Tangem | No stdlib equivalent in TypeScript. Integer atomic strings + BigInt achieve exactness without the dependency. |

---

## 8. Ranked plan

### P0 — money correctness and authority
1. **Canonical `AssetId`** — composite, network-bound, per-asset decimals. Delete `USDC_DECIMALS`; fold `NETWORK_USDC` and `DEVNET_USDC_MINT` into one registry.
2. **One money representation** — integer atomic decimal strings across boundaries; BigInt arithmetic; formatter for display.
3. **Balance decomposition** — `available` / `reserved` / `settled` as distinct quantities in `BudgetEnvelope`, not one counter.
4. **`Limit(limit, spent)` shape** — so the invariant is checkable from the record.

### P1 — enforcement actually running
5. **Wire `x402-guard` into the gateway path.**
6. **Wire `settlement.releaseDecision` into the release path.**
7. **Monotonic `reservationSeq`** on the budget store.
8. **Scheduled limit changes** — `oldLimits`/`newLimits`/`effectiveAt`.
9. **Step-up threshold as data** (`noApprovalBelow`), replacing a branch.

### P2 — later
10. Purpose-scoped availability (needed when refunds land).
11. Configurable spend windows (`spendPeriodSeconds` equivalent).
12. API/Impl package split to make inert security modules structurally impossible.

### REJECT
Everything in §7.

---

## 9. Proposed commit sequence

Revised from the brief after inspecting the tree. Each commit builds, passes tests, and is independently revertible.

| # | Commit | Gate |
|---|---|---|
| 01 | `asset-identity` — canonical `AssetId`, one registry | mainnet-mint-on-devnet must throw |
| 02 | `money-model` — atomic strings + BigInt, boundary conversions removed | no `number` money in any new path; existing suites green |
| 03 | `budget-unification` — one `BudgetEnvelope`, decomposed balances, `Limit(limit,spent)`, `reservationSeq` | 10 × $20 vs $50 → exactly 2, single- and multi-instance |
| 04 | `wire-x402-guard` — F1–F5 defences on the live path | each defence has a test that fails when removed |
| 05 | `wire-release-decision` — finality gate on resource release | release blocked below required confirmations |
| 06 | `payments-policy` — `allow/deny/require_approval`, intent-bound approval | cross-intent approval replay denied |
| 07 | `usage-event` — written with the reservation commit | one invocation → one event; replay adds none |
| 08 | `ledger-durability` + storage decision (measured) | orphaned reservation recovered after crash |
| 09 | `settlement-evidence` — observed, never copied from intent | evidence disagreeing with intent is preserved verbatim |
| 10 | `reconciliation` — deterministic comparator | all 10 statuses reachable |
| 11 | `real-settlement` — one funded devnet transfer | real signature, or documented blocker; no synthetic pass |
| 12 | `cross-rail` + `statement` | byte-identical export across runs |
| 13 | `adversarial-tests` + `docs` | every defence has a removal-fails test |

**Deliberate reordering vs the brief:** asset identity and the money model come *first*, before budget unification. Unifying two budget engines while they still disagree about what a dollar is would bake the conversion bugs into the merged engine.

---

## 10. What this audit did not establish

Stated so nothing here is over-read:

- **RedotPay's backend architecture.** Not knowable from the artifact. No claim made.
- **Tangem's iOS implementation.** 6,844 Swift files present but not traced; Android was traced and is the basis of every Tangem claim above.
- **Runtime behaviour of any Tangem flow.** Nothing was built or executed; all Tangem findings are static reads of source.
- **Whether the P0 asset-identity defect is currently exploitable.** It is structurally possible; no test proves a live path reaches it. Commit 01's regression test should be written to fail *before* the fix, which will settle it.
