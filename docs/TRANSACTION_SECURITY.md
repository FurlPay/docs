# Transaction security — the unified authorization boundary

**Status:** boundary implemented, wired end to end, and tested (148 tests in `lib/integrity`). The Android client walks the full prepare → confirm → authorize → sign flow. Platform verification calls to Apple/Google are **not** implemented — credential-blocked, and both providers refuse to report a verdict they did not check. No simulation provider is configured, and the local pass never reports `clean`.
**Covers:** A2 (transaction simulation) + A9 (app attestation), built as **one control**.

---

## Why these are one control, not two

Adding "Blockaid" and "Play Integrity" as independent features leaves the gap *between* them, and the gap is where the attack lives:

```
simulate tx A → attest tx A → confirm tx A → sign tx B
```

Every individual control passes. The wrong transaction is signed. No cryptography is broken; nothing is bypassed. The controls simply pointed at four different transactions.

**The defence is that all four commit to the same bytes.** That is what `lib/integrity/` implements.

```
User intent
    │
    ▼
Canonical intent ──────► SHA-256 digest ◄─── the one value everything binds to
    │                          │
    ├──────────┬───────────────┼──────────────┐
    ▼          ▼               ▼              ▼
Simulation  Attestation    User confirm   About-to-sign
(keyed on   (requestHash / (recorded      (re-canonicalised
 digest)     clientDataHash) against       from the signer
                = digest)    digest)        path)
    │          │               │              │
    └──────────┴───────┬───────┴──────────────┘
                       ▼
              authorizeForSigning()
                       │
         all four agree? ──no──► BLOCK, with every refusal listed
                       │
                      yes
                       ▼
                  Risk engine  (integrity contributes a weight, not a verdict)
                       ▼
                    Signer
```

---

## The binding primitive — `lib/integrity/canonical.ts`

A `CanonicalIntent` is deliberately flat and **all strings**. Numbers are excluded because a float has no canonical decimal form — `10.10` could serialise differently on two platforms and break the binding silently.

Canonicalisation is **hand-rolled, not `JSON.stringify`**. JS object key order is insertion order, so two structurally identical intents built differently would serialise differently, client and server would compute different digests, and someone would eventually "fix" it by deleting the comparison.

Encoding: `v1` + for each field in an **explicit** order, `name US value RS` (0x1F/0x1E). Properties this buys:

| Property | Why it matters |
|---|---|
| Explicit field order | A sort depends on locale and key strings; a mismatch diverges the digests. Adding a field is also a reviewable act — adding one *without* listing it here would leave it outside the binding |
| Absent fields emitted empty, not skipped | Otherwise `{spender: undefined, calldata: "0xab"}` and `{spender: "0xab", calldata: undefined}` collide |
| Separators rejected inside values | A smuggled `0x1F` would shift every later field and forge a digest for a different intent |
| EVM addresses case-folded, non-EVM not | Base58 is case-significant; folding a Solana address produces a *different, wrong* address |
| Amount must be integer minor units | Matches `lib/money/money.ts`; a float here is the bug the whole module prevents |

`intentDigest()` → SHA-256 hex, 64 bytes. Fits Play Integrity's 500-byte `requestHash` cap with room to spare.

### `chainPayloadDigest` — binding the BYTES, not just the description

The intent above describes a payment. It does not, by itself, constrain the transaction that executes.

That gap is not theoretical. A client could submit `amountMinor: "1"` — passing every budget, every spend policy, every confirmation screen — alongside calldata transferring the entire balance to an attacker. Simulation would run on the calldata, the user would confirm the *summary*, and the two would describe different transactions. Every control passes and the wrong transaction is signed: the original vulnerability wearing a different hat.

So `CanonicalIntent` carries `chainPayloadDigest`, a SHA-256 over the canonicalised EIP-1559 payload (`lib/integrity/evmPayload.ts`). Folding it into the intent means the canonical digest covers the transaction bytes themselves.

A digest proves the bytes did not change. It says nothing about what they *mean* — so `verifyPayloadMatchesIntent()` additionally **decodes** the calldata and compares, field by field:

| Check | Refusal |
|---|---|
| Settlement domain matches the payload's chain id | `payload_chain_mismatch` |
| Payload is the one bound into the intent | `payload_digest_mismatch` |
| `to` is the asset's *canonical* token contract | `payload_token_mismatch` |
| Decoded recipient equals `destination` | `payload_recipient_mismatch` |
| Decoded spender equals `spender` (approvals) | `payload_spender_mismatch` |
| Decoded amount equals `amountMinor` | `payload_amount_mismatch` |
| No native value alongside a token transfer | `payload_native_value_unexpected` |
| Calldata is readable at all | `payload_not_verifiable` |
| Action has a verifier | `payload_action_unsupported` |

The decoder is strict in two ways that matter. **Length is exact**: Solidity ignores trailing calldata, so a lenient decoder and the EVM would disagree about what the transaction is. **Address padding must be zero**: the EVM masks the top 12 bytes off, so dirty padding changes nothing on-chain while changing what a naive decoder reads — a free way to show one recipient and use another.

`swap`, `withdraw`, `contract_call` and `card_authorize` are **refused**. Their semantics cannot be read off the calldata, so an amount and a destination cannot be checked against them. A signing path that cannot explain what it is signing must not sign.

### Two different hashes, and why

| | `canonicalDigest` | `evmSigningHash` |
|---|---|---|
| What it is | SHA-256 over the canonical intent | keccak256 over the RLP-encoded unsigned EIP-1559 tx |
| What it is for | **Binding** — simulation, attestation, confirmation and the authorization all commit to it | **Signing** — the bytes the chain wants |
| Where it comes from | `intentDigest(intent)` | `evmSigningHash(storedPayload)` |

Signing the canonical digest would produce 65 valid-looking bytes over the wrong message. The client's own recovery check rejects them, and if it did not, the chain would. The two are tied together by `chainPayloadDigest`: it is folded into the intent, and `consumeAuthorization` re-derives it from the same stored payload the signing hash comes from.

---

## The three-call flow

The client never supplies bytes to be signed. It supplies a proposal once, and thereafter it can only point at what the server stored.

```
PREPARE    POST /api/transactions/prepare
           canonicalise -> verify calldata <-> intent -> simulate -> store
           returns  intentId, canonicalDigest, summary, confirmation challenge

CONFIRM    device-owner authentication over the SERVER's summary
           (nothing is authorised yet — a decline leaves no ticket minted)

AUTHORIZE  POST /api/transactions/authorize
           consume challenge -> verify integrity against OUR digest -> run the
           boundary -> mint a single-use, device-bound authorization
           returns  authorizationId, signing challenge

SIGN       POST /api/mpc/sign
           device check -> consume authorization -> consume challenge ->
           agent policy priced from the STORED intent -> sign evmSigningHash(payload)
```

Ordering is deliberate at two points. The **device check runs first** at both authorize and sign, so a device we were always going to refuse does not burn a single-use challenge — that would turn a refusal into a denial of service against the legitimate user. And **confirmation sits between prepare and authorize**, so a decline leaves nothing authorised rather than arriving after a ticket was minted.

`TransactionProposal.version` increments on every mutation and is frozen into the authorization. A proposal that moves after the security cycle ran cannot be signed against that authorization; it needs a new cycle. The version comes from the **stored** record at signing time, never from the request — taking it from the request would let the caller supply both sides of the comparison.

---

## Platform providers — `lib/integrity/providers.ts`

### What is implemented

The **verification order, request binding, replay defences and refusals** — the parts that are ours to get right and the parts an attacker probes.

`AndroidPlayIntegrityProvider.evaluatePayload()` is complete and fully tested. It runs the checks in the order the platform guidance requires — **bind first, then judge**:

1. Package name matches
2. **`requestHash` equals the digest the server computed independently** ← the check without which the token can be lifted from a $1 request onto a $10,000 one
3. Token freshness (5-minute window)
4. Device verdict, app verdict, Play Protect, app-access-risk

Verdict mapping:

| Condition | Verdict | Why |
|---|---|---|
| Hash mismatch, wrong package, stale, no verdict at all | `failed` | Token is about something else, or the device met no bar |
| Emulator (`MEETS_VIRTUAL_INTEGRITY` only) | `degraded` | CI, QA and accessibility tooling run in emulators |
| `UNRECOGNIZED_VERSION` | `degraded` | Enterprise and sideloaded builds are legitimate |
| Screen-capture / overlay apps detected | `degraded` | Real signal, not proof |
| All checks clean | `passed` | — |

### What is NOT implemented

The network calls: Apple's attestation-object verification against their root, and Google's token decode. Both need credentials this deployment does not have.

**Both providers return `not_configured`, never `passed`.** Writing a client that "works" against an unconfigured service would produce a `passed` verdict from a function that contacted nobody.

The Apple provider deliberately does **not** advance the assertion counter while unimplemented — burning the counter for a token nobody verified would cause a later real implementation to reject the genuine follow-up assertion.

### Assertion replay

`kvAssertionCounters` is a **durable, shared** compare-and-set on the KV layer. This must not be in-memory: on serverless, a per-instance counter means a replayed assertion lands on a different lambda and passes — defeating the only control that catches a captured assertion being re-presented.

`advance()` requires **strictly greater**. Equal is a replay, not a retry.

---

## The boundary — `lib/integrity/boundary.ts`

`authorizeForSigning()` collects **every** refusal rather than short-circuiting. During an incident, "stale simulation" and "signing payload mismatch" are a slow user versus an active attack, and finding them one request at a time wastes the window where that distinction is actionable.

### The deliberate asymmetry

| Situation | Result | Reasoning |
|---|---|---|
| Integrity signal **absent** | **Allow** + warn | Apple's guidance is gradual onboarding with graceful handling of unsupported devices |
| Device **unsupported** | **Allow**, weight ≈ 0.05 | The customer's hardware genuinely cannot attest. Not their fault |
| Verification **failed**, not a mismatch | **Allow**, weight 0.6 | Broken Play Services, enterprise build — high risk, not proof |
| Verdict **`not_configured`** | **Allow**, weight **0** | Our gap must never add risk to a customer's score |
| Token **bound to a different digest** | **BLOCK** | A *valid* token being replayed onto another request — worse than no token |
| Signing payload ≠ confirmed intent | **BLOCK** | Something rewrote the transaction |

> **A missing attestation is weak evidence and does not block. A mismatched binding is proof that something rewrote the transaction and always blocks.**

Hard-gating on attestation blocks old hardware, de-Googled Android and enterprise builds — real customers — while an attacker simply moves to whichever platform passes.

### Simulation policy

| Outcome | Action | Reasoning |
|---|---|---|
| `malicious` | BLOCK | — |
| `warning` | Allow + surface | A simulator that blocks on every warning trains users to route around it |
| `unavailable` | Allow + warn | Not evidence of danger; not evidence of safety either |
| Missing / stale / unbound | BLOCK | Chain state moves; a simulation of a different tx is not evidence about this one |
| Unlimited approval | Allow + prominent warning | The user should decide, having been told plainly |

### Language

`simulationSummary()` **never** returns "transaction is safe". A clean result means *one provider detected no known issue against current chain state*. Stating it as a guarantee transfers our uncertainty to the user as false confidence — the thing they will point at afterwards.

```
✅ "Simulation completed. No known issues were detected."
❌ "Transaction safe"
```

A test asserts the word "safe" never appears.

---

## Red-team coverage

Every scenario from the brief, implemented literally as a test that must **BLOCK**:

| Attack | Test | Result |
|---|---|---|
| Client approves $10, attacker changes to $10,000 | `aboutToSign` amount differs | `signing_payload_mismatch` |
| Simulation recipient A, signature recipient B | `aboutToSign` destination differs | `signing_payload_mismatch` |
| Transfer rewritten as an approval | action + spender differ | `signing_payload_mismatch` |
| Chain switch | chain differs | blocked |
| Valid integrity token replayed onto another request | `boundDigest` differs | `integrity_unbound` |
| Simulation of a different transaction | `boundDigest` differs | `simulation_unbound` |
| User confirmed a different transaction | `boundDigest` differs | `confirmation_unbound` |
| Expired intent replayed | `expiresAt` past | `expired` |
| Separator smuggled into a field | canonicalisation | throws |
| Float amount | canonicalisation | throws |

---

## Still blocked

| Blocker | Needs |
|---|---|
| Apple attestation verification | Apple team id + App Attest environment + attestation-object verification against Apple's root |
| Play Integrity token decode | Google Cloud service account with the Play Integrity API enabled |
| Simulation provider | A transaction-security vendor, or a self-hosted `eth_call`-based simulator |
| Client SDKs | `DCAppAttestService` (iOS), Play Integrity token provider (Android). iOS is blocked behind A1 — there is no iOS app |
| Android key attestation | A Keystore/StrongBox key generated with an attestation challenge, and the Google hardware-attestation roots to validate its chain against |

`native-app/lib/security/deviceIntegrity.ts` already carries the matching TODO on the client side and correctly reports `not_configured`. This work builds the server half it names.

The device key the Android app enrols today (`native-app/lib/device/registration.ts`) is a secp256k1 key generated in JS and held in SecureStore, which on Android is Keystore-encrypted. That protects it **at rest**. It is not a StrongBox-attested hardware key, and nothing about it proves to the server that the private key is non-extractable — which is why the registry keeps such a device `pending` rather than trusting a `hardwareBacked: true` flag that would mean nothing.

## Next

1. **Android Keystore / StrongBox key generation + attestation-chain verification.** The device registry is complete and its state machine is enforced, but no device can reach `hardware_trusted` until a certificate chain can be validated to a Google root. Until then `recordAttestation` records `attestation_missing`, and policy treats that as untrusted rather than promoting on a client-supplied flag.
2. **Play Integrity client + credentials.** The server evaluator is written and tested; nothing on the device produces a token, so the app sends no `integrity` block at all. That is the honest state — a fabricated token would be treated as tampering, and a missing one is a weighted risk signal the boundary already supports.
3. **A simulation provider.** `lib/integrity/simulation.ts` never returns `clean`, and it says why in the file: a control that reports success without doing the work is worse than no control, because it silences the warning that would otherwise be shown. The local pass reads the calldata for unlimited approvals, burn addresses and self-sends. It cannot know that an address drained a pool yesterday.
4. **Verifiers for `swap` / `withdraw` / `contract_call`.** Currently refused outright.
5. **A1 — iOS app.** The client half of App Attest cannot exist without it.
