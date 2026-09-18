# FurlPay mobile security audit — Phase 0

**Date:** 6 September 2026
**Scope:** Android application (`native-app/`), backend signing/payment paths (`apps/web/`), iOS.
**Method:** source review against OWASP MASVS/MASTG categories, Android platform guidance, and the signing-boundary invariant.

---

## 0. Scope correction — read first

### There is no FurlPay iOS application.

A security review of an iOS app cannot be performed, because one does not exist. Verified:

| Check | Result |
|---|---|
| `native-app/ios` | **absent** — `expo prebuild` has never run for iOS |
| `eas.json` iOS build profile | **0 occurrences** of `"ios"` |
| `furlpay-ios/` | 28 Swift files: `Home`, `Pay`, `Scan`, `History`, `Settings` — a **Solana Pay demo**, not the product |
| `ios/FurlPayGuardian/` | 105 Swift files — a **different application** (security companion + Watch) |

**No iOS findings are reported below.** Any iOS finding I produced would be fabricated. The Android/iOS comparison table required by the brief is therefore a table of one platform plus an absence, and I have written it that way rather than inventing parity.

### Testing limitations

Static source review only. **Not performed:** dynamic analysis, Frida/Objection instrumentation, runtime interception, live API testing, device testing, or dependency CVE scanning against a resolved lockfile. Findings below are marked **CONFIRMED** (read in source) or **REQUIRES VALIDATION**.

---

## 1. Architecture and data flow

```
┌─ ANDROID (Expo RN, unpublished) ─────────────────────────────┐
│  Screens (57)                                                │
│  lib/signing/  ├─ deviceKeySigner  BIP-39 seed, SecureStore  │
│                ├─ mpcSigner        digest → server           │
│                └─ remoteMpcBackend POST /mpc/sign            │
│  lib/wallet.ts     SecureStore, biometric-gated              │
│  lib/security/     deviceIntegrity (local heuristics only)   │
│  lib/deepLinks.ts  furlpay:// + https://furlpay.com/l/       │
└────────────────────────┬─────────────────────────────────────┘
                         │ TLS + cert pinning (expires 2027-07-18)
┌────────────────────────▼─────────────────────────────────────┐
│  BACKEND (Next.js, 302 routes)                               │
│   middleware        session JWT, rate limit, correlation id   │
│   /api/mpc/sign     → Turnkey (custodial enclave signer)      │
│   /api/wallets/transfer  → demo ledger, 503 in production     │
│   /api/chain/rpc    → proxies eth_sendRawTransaction          │
│   lib/integrity/    canonical → boundary → authorization      │
└──────────────────────────────────────────────────────────────┘

iOS: ─────────────  DOES NOT EXIST  ─────────────
```

### Trust boundaries

1. **Device ↔ app** — biometric-gated SecureStore. Broken by a rooted device.
2. **App ↔ backend** — session JWT. **No device binding, no app-instance binding.** A stolen token works from anywhere.
3. **Backend ↔ signer** — the boundary hardened this cycle. Now the strongest link.
4. **Backend ↔ chain** — fee-payer key via KMS, flag-gated.

---

## 2. GREEN / YELLOW / RED matrix

| # | Control | State | Evidence |
|---|---|---|---|
| 1 | Canonical transaction digest | 🟢 | `lib/integrity/canonical.ts`, 44 tests |
| 2 | Simulation ↔ digest binding | 🟢 | `boundary.ts`; refuses unbound |
| 3 | Confirmation ↔ digest binding | 🟢 | `boundary.ts` |
| 4 | Signing ↔ digest binding | 🟢 | `authorization.ts` re-derives from stored intent |
| 5 | Authorization single-use | 🟢 | Atomic `SET NX` **before** validation; concurrency test |
| 6 | Authorization ↔ user binding | 🟢 | `authorization_user_mismatch` |
| 7 | Authorization ↔ wallet binding | 🟢 | `authorization_wallet_mismatch` |
| 8 | Authorization expiry | 🟢 | 120 s TTL |
| 9 | TOCTOU / transaction version | 🟢 | `transaction_version_mismatch` |
| 10 | Audit trail for signing decisions | 🟢 | 3 events, full diagnostics |
| 11 | Client-safe refusals | 🟢 | All binding failures collapse to one message |
| 12 | Secure local storage | 🟢 | **Zero `AsyncStorage`**; SecureStore throughout |
| 13 | `allowBackup=false` | 🟢 | Manifest |
| 14 | Cleartext traffic disabled | 🟢 | `cleartextTrafficPermitted="false"` base config |
| 15 | Certificate pinning | 🟡 | 3 ISRG pins — **expire 2027-07-18** |
| 16 | Exported components | 🟢 | Only `MainActivity` (`singleTask`) |
| 17 | Deep-link resolution | 🟡 | Maps to routes with fallback; **no proof it cannot reach a money action** |
| 18 | Fail-closed on security errors | 🟢 | Swept for `catch → return true`: **none found** |
| 19 | Platform attestation (Play Integrity) | 🔴 | Server evaluator built; **no client, no credentials** |
| 20 | Platform attestation (App Attest) | 🔴 | No iOS app |
| 21 | Device binding | 🔴 | Session JWT has no device claim |
| 22 | App-instance binding | 🔴 | No concept exists |
| 23 | Server-issued signing challenge | 🔴 | Not implemented |
| 24 | Hardware key attestation | 🔴 | `deviceIntegrity.ts` is local heuristics; its own TODO says so |
| 25 | Biometric ↔ signature binding | 🔴 | Biometric gates *key read*, not *this transaction* |
| 26 | Display ↔ signing digest binding | 🔴 | Confirmation UI does not derive from the canonical object |
| 27 | Device revocation | 🔴 | No device registry |
| 28 | Screenshot protection on confirmation | 🔴 | `secureScreen.ts` exists; not applied to a signing screen |
| 29 | Submission idempotency | 🟡 | Route-level `withIdempotency`; not on the signing→broadcast pair |
| 30 | Dependency CVE posture | ⚪ | **Not assessed** — requires lockfile scan |

**🟢 14 · 🟡 4 · 🔴 11 · ⚪ 1**

---

## 3. Findings

### FURL-001 — Agent spend budget bypassed by client-supplied amount
**Severity: HIGH** · CVSS 3.1 **7.1** (`AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:H/A:N`) · Android + Backend · **CONFIRMED** · **FIXED this cycle**

`/api/mpc/sign` priced the agent spend policy from `context.amountUsd` in the request body. The Android client sends a hardcoded zero:

```ts
// native-app/lib/signing/remoteMpcBackend.ts:80
amountUsd: 0,
// "Advisory only … Sending a smaller number here buys nothing: the
//  client-side gate in MpcSigner already checked the real value"
```

The comment is the vulnerability, stated aloud. `authorizeAgentSpend({ amountUsd: 0 })` cannot exceed any budget, so **every agent-driven signature passed the per-transaction limit and consumed zero daily budget** — regardless of the transaction's real value. The compensating control named in the comment is *client-side*, and a client-side gate does not constrain an attacker who controls the client.

**Attack:** register an agent key capped at $50 → craft a wallet-draining transaction → send its digest with `amountUsd: 0` → policy passes → enclave signs.

**Root cause:** a security decision priced from attacker-controlled input.

**Fix (applied):** `/api/mpc/sign` now derives the amount from the **server-stored authorized intent** via `spendUsdFromIntent()`. An unpriceable asset **refuses** rather than defaulting to zero.

**Verify:** submit an authorization for 10 USDC under an agent key with a $5 cap → expect `policy_denied`.

---

### FURL-002 — Signing endpoint accepted an arbitrary digest
**Severity: HIGH** · CVSS 3.1 **7.5** · Backend · **CONFIRMED** · **FIXED this cycle**

The route accepted any 32-byte digest with self-described "advisory" context. Its own comment: *"the digest is authoritative and we cannot verify that these fields describe it."* Anyone reaching the route could have the enclave sign arbitrary bytes.

**Fix (applied):** the request now presents a server-minted `authorizationId`; the server signs the digest **derived from its own stored intent**. `digest` is no longer accepted.

---

### FURL-003 — Android client is incompatible with the hardened endpoint
**Severity: MEDIUM (availability)** · Android · **CONFIRMED** · **FIXED**

`remoteMpcBackend.ts` posted `{ signWith, digest, context }` against a route that no longer accepted a digest, so MPC signing failed Zod validation. It failed **closed** — no signature was produced — but it was a breaking change and the client work was unfinished.

**Fix (applied):** the client now runs the full three-call flow.

```
PREPARE    POST /api/transactions/prepare
           the app proposes the transaction it BUILT. The server decodes the
           calldata itself, proves it means what the proposal claims, simulates,
           freezes it, and returns a digest + a single-use confirmation challenge.

CONFIRM    the user approves the SERVER's summary via device-owner auth.

AUTHORIZE  POST /api/transactions/authorize
           the app echoes the digest it DISPLAYED and returns the challenge.
           The boundary runs; a device-bound, single-use authorization is minted.

SIGN       POST /api/mpc/sign
           the app presents the authorization. The server re-reads its own
           stored payload, derives the EIP-1559 signing hash from THAT, signs it.
```

The app never says which bytes to sign, and `MpcSigner` still recovers the address from **our** digest — so if the server signed anything else, the recovery fails on the device rather than on-chain. That check is what makes it safe for the client to stop sending a digest.

**New client refusals, all before the network:** `describeTransaction()` refuses unknown calldata, an unrecognised contract, an unpinned chain, trailing calldata, non-zero address padding, and native value smuggled alongside a token transfer. A description the app cannot justify from the bytes would be shown to the user as fact.

**Verify:** `native-app/lib/__tests__/remoteMpcBackend.test.ts` (29 tests), `intentDescription.test.ts` (13).

---

### FURL-004 — No device or app-instance binding on the session
**Severity: MEDIUM** · CVSS 3.1 **6.5** · Both · **CONFIRMED**

The session JWT carries no device or app-instance claim. A token exfiltrated from a compromised device — via backup, malware, or an OS bug — replays from any client. Authorization is now wallet- and user-bound, but **not device-bound**, so the chain in the brief (`USER → DEVICE → APP INSTANCE → …`) is broken at its second link.

**Fix:** device registry + hardware-attested device key; add `deviceId`/`appInstanceId` to the session and to `SigningAuthorization`; verify at mint and consume.

---

### FURL-005 — Biometric gates key access, not the specific transaction
**Severity: MEDIUM** · Android · **CONFIRMED**

`deviceKeySigner.signTransaction()` triggers the biometric by *reading the key*. The prompt therefore authorises "use the key", not "send 10 USDC to this address". Malware that wins a race after unlock, or a UI overlay, can obtain a signature the user believes was for something else.

**Fix:** `BiometricPrompt` bound to a `CryptoObject` whose Keystore key signs *this* canonical digest — so a successful biometric produces a signature over that transaction and nothing else.

---

### FURL-006 — Certificate pins expire 2027-07-18
**Severity: LOW** · Android · **CONFIRMED** · **FIXED**

Three ISRG pins with a hard expiry. Past that date the `pin-set` stops being enforced (Android's documented behaviour is to *ignore* an expired pin-set, so it fails **open** to normal CA validation rather than bricking the app — the safer of the two failure modes, and also the dangerous one, because the control disappears with no symptom).

**Fix (applied):** `assertPinsNotNearExpiry()` runs inside the config plugin's `withDangerousMod`, so it executes on every `expo prebuild` and therefore on every EAS build and CI run.

- expiry in the past → **throws**, the build fails
- expiry within 180 days → `console.warn`, build succeeds

The window warns rather than throws deliberately: a hard failure inside the rotation window would block an incident hotfix over pins that are still valid. Rotation procedure: `docs/CERT-PIN-ROTATION.md`.

**Verify:** `native-app/plugins/__tests__/networkSecurityConfig.test.ts` — including a test that asserts it *throws* one day past expiry.

---

### FURL-007 — Cleartext permitted for localhost in the release manifest
**Severity: LOW** · Android · **CONFIRMED** · **FIXED**

`cleartextTrafficPermitted="true"` for `localhost`, `10.0.2.2`, `127.0.0.1` shipped in the release config. The in-file comment argued these "only exist on a developer machine" — on a *device*, `localhost` is the device itself, and a malicious co-resident app can bind a port. Exploitation additionally required influencing the base URL, which is compiled in, so this was hardening rather than a live path.

**Fix (applied):** the exception is now written into the `debug` source set (`app/src/debug/res/xml/`), which Android merges for debug variants and never for release. Release inherits a base config that forbids cleartext everywhere. Debug builds keep pinning for `furlpay.com` — the relaxation is scoped to the dev hosts, not to production, and a test asserts both configs pin the **same** anchors, because divergent sets are how a bad pin reaches production unnoticed.

The fix lives in the Expo config plugin, not in `android/` — that directory is gitignored and regenerated by `expo prebuild`, so an edit there would be silently reverted on the next build.

**Verify:** `native-app/plugins/__tests__/networkSecurityConfig.test.ts`.

---

### FURL-008 — Deep links resolve to routes without a proven money-action barrier
**Severity: MEDIUM** · Android · **REQUIRES VALIDATION**

`resolveDeepLink()` maps arbitrary inbound paths to Expo Router routes with a fallback, and a route-existence test guards the contract. What I could **not** establish from source is whether any reachable route can initiate a money movement from link parameters alone. `MainActivity` is `singleTask`, so links land in the running task.

**Fix:** enumerate every reachable route and assert none can move money without a fresh server-side authorization. Then test it.

---

### FURL-009 — `/api/chain/rpc` proxies `eth_sendRawTransaction`
**Severity: INFORMATIONAL** · Backend · **CONFIRMED** · **accepted risk**

The proxy is authenticated, rate-limited (120/min/user) and method-allowlisted. It is **not** a meaningful bypass: anyone holding a signed transaction can broadcast it to any public RPC. This is the architectural reason the boundary must sit at **signing**, not at broadcast — once a signature exists, the money is gone. Documented so a future reviewer does not "fix" it by locking the proxy and believing that stopped anything.

---

## 4. Android vs iOS comparison

| Area | Android | iOS |
|---|---|---|
| Application exists | ✅ built, unpublished | ❌ **does not exist** |
| Secure storage | 🟢 SecureStore, zero AsyncStorage | n/a |
| Network security | 🟡 pinning, expires 2027 | n/a |
| Platform attestation | 🔴 server evaluator only | 🔴 no app |
| Biometric binding | 🔴 gates key, not transaction | n/a |
| Deep links | 🟡 unvalidated | n/a |
| Exported components | 🟢 MainActivity only | n/a |

**Every "shared" vulnerability is Android-only by default, because the second platform has no code to be vulnerable.**

---

## 5. Scorecard

| Category | Score | Basis |
|---|---:|---|
| Authentication | 6/10 | Passkey + biometric; **no device binding** |
| Authorization | 8/10 | Strong post-hardening; not device-bound |
| Session management | 5/10 | Durable revocation; portable token |
| Cryptography | 7/10 | Standard primitives, KMS boundary; no custom crypto |
| Secure storage | 8/10 | SecureStore throughout, `allowBackup=false` |
| Network security | 7/10 | Pinning present, expiring; dev cleartext ships |
| API security | 7/10 | Manifest-enforced auth, rate limits, idempotency |
| Android security | 6/10 | Config sound; **no attestation, no anti-tamper** |
| **iOS security** | **n/a** | **No application exists** |
| Privacy | 6/10 | Not fully assessed |
| Dependency security | ⚪ | **Not assessed** |
| Secrets management | 8/10 | KMS boundary; env-key signing refused in production |
| Payment security | 7/10 | Digest binding strong; client not yet wired |
| Business logic | 6/10 | Lifecycle + evidence gating; velocity present |
| Anti-tampering | 3/10 | Local heuristics only, honestly labelled as such |

**Overall: 6.4/10** for Android + backend. Not scored for iOS.

---

## 6. Remediation roadmap

**Done**
- FURL-001 — agent cap priced from the server-stored authorized intent; unpriceable assets refuse.
- FURL-002 — the route no longer accepts a digest; it signs what it authorised.
- FURL-003 — Android client on the prepare → confirm → authorize → sign flow.
- FURL-006 — build-time pin-expiry guard + written rotation procedure.
- FURL-007 — dev-host cleartext scoped to the `debug` source set.
- Device + app-instance registry, server-minted ids, five-state machine, revocation via key-version bump.
- Server-issued single-use signing challenge (RED #23).
- Calldata↔intent verification: the amount and recipient shown to the user are decoded from the bytes that will execute.

**Immediate (0–7 days)**
1. FURL-004 (remainder) — carry `deviceId`/`appInstanceId` in the session claims, not only in the signing path.
2. FURL-005 — `CryptoObject`-bound biometric signing.

**Short term (1–4 weeks)**
3. Android Keystore/StrongBox key generation + attestation-chain verification, so a device can actually reach `hardware_trusted`.
4. FURL-008 — deep-link money-action audit + test.
5. Play Integrity client + credentials; wire the built evaluator.

**Medium term (1–3 months)**
6. Display-digest binding (RED #26) — confirmation UI rendered from the canonical object.
7. A real simulation provider. The local pass is structural only and says so.
8. Screenshot protection on the signing screen; dependency CVE scan.
9. Verifiers for `swap` / `withdraw` / `contract_call`, which are currently refused outright.

**Long term**
10. iOS application, then App Attest.
11. Property-based canonicalisation fuzzing.
12. External penetration test.

---

## 7. Honest answer to the brief's final question

> *If a capable attacker targeted FurlPay's mobile app today, what could they realistically compromise?*

**Before this cycle:** an attacker with an agent key could have the enclave sign an arbitrary digest while reporting `amountUsd: 0` — full bypass of the spend policy, bounded only by wallet balance. That is FURL-001 + FURL-002, and both were live.

**After this cycle:** that path is closed at the server, and the client now walks it. Signing requires a proposal the server decoded, a confirmation the user gave, a challenge only that device holds, and an authorization bound to the device, the app instance, the device key version and the proposal version. The digest is never supplied and never accepted.

What remains reachable:

1. **A compromised device is still a compromised device.** The biometric gates key access, not the transaction (FURL-005). No attestation detects the compromise (RED #19, #24), and no device can currently reach `hardware_trusted` because the attestation-chain verifier is not written — the registry says so rather than promoting on a client-supplied flag.
2. **A stolen session token still replays from anywhere** for non-signing endpoints (FURL-004 remainder). It can no longer produce a signature: the device binding is enforced at prepare, authorize and sign, and a token alone carries neither the enrolled device key nor the app instance.
3. **Nothing checks the transaction against the world.** No simulation provider is configured. The local pass reads the calldata for unlimited approvals and burn addresses; it cannot know an address drained a pool yesterday, and it never returns "clean".

**FurlPay is not secure.** The signing boundary is now enforced end to end and the chain `USER → DEVICE → APP INSTANCE → TRANSACTION → SIGNATURE` holds along its whole length — but the DEVICE link rests on a software key, because nothing yet proves the key is in hardware.
