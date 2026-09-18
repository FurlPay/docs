# Production gap closure — status

**Updated:** 6 September 2026 · Tracks the A1–A10 gaps from `docs/COMPETITIVE_ANALYSIS.md`.

Nothing is marked DONE because an interface exists. DONE means implemented, tested, and verified.

| Gap | Status | Files | Tests | External dependency | Remaining risk |
|---|---|---|---|---|---|
| **A1 iOS** | NOT STARTED | — | — | Apple Developer account, signing certs, App Attest capability | **Largest product gap.** No `native-app/ios`, no iOS EAS profile. Blocks the client half of A9 |
| **A2 Simulation** | **PARTIAL — boundary DONE** | `lib/integrity/{canonical,boundary,types,providers}.ts` | **44** | Simulation vendor or self-hosted `eth_call` simulator | Boundary is built and **not yet wired into any route** |
| **A3 Card state machine** | NOT STARTED | — | — | Issuer (for the live path; the model itself is ours) | Card state is scattered booleans |
| **A4 Anti-phishing code** | NOT STARTED | — | — | none | Pure code. Cheap, high value |
| **A5 P2P durability** | NOT STARTED | `lib/services/p2p.ts` is `globalThis` | — | none | Per-instance state; orders vanish on recycle |
| **A6 Statement export** | NOT STARTED | — | — | none | Pure code, ledger-derived |
| **A7 EDD** | NOT STARTED | — | — | none for the framework | Required by FIU-IND 2026 guidelines |
| **A8 Hardware signer** | INTERFACE ONLY | `native-app/lib/signing/` | existing | Hardware SDK licences | `SignerPicker` advertises NFC/BLE with nothing behind it |
| **A9 App attestation** | **PARTIAL — server DONE** | `lib/integrity/providers.ts` | **44** (shared with A2) | Apple team id + App Attest env; Google service account | Providers report `not_configured`, never `passed`. Client SDK blocked behind A1 |
| **A10 Recall UI** | NOT STARTED | — | — | Provider recall APIs | Lifecycle models the states; no surface |

---

## What was implemented this cycle

**A2 + A9 as one control**, per the architectural correction: attestation and simulation bind to the same canonical digest, so the four-control gap (`simulate A → attest A → confirm A → sign B`) is closed structurally rather than by convention.

- `canonical.ts` — deterministic intent serialisation + SHA-256 digest. Explicit field order, separator rejection, EVM-only case folding, integer-minor-unit enforcement.
- `types.ts` — `AppIntegrityProvider`, normalised `AppIntegritySignal`, `AssertionCounterStore`, advisory risk weighting.
- `providers.ts` — Play Integrity payload evaluation (complete, tested), Apple App Attest scaffold, durable assertion-counter store.
- `boundary.ts` — `authorizeForSigning()`. Collects every refusal; asymmetric on integrity.
- 44 tests including every red-team scenario from the brief.

**Security properties achieved**

1. Transaction substitution between confirmation and signing is **blocked** — the digest is recomputed from what is about to be signed.
2. A valid attestation token replayed onto a different request is **blocked**.
3. A simulation or confirmation of a different transaction is **blocked**.
4. Assertion replay is defeated by a **durable, strictly-increasing** counter (in-memory would fail on serverless).
5. `requestHash` is **independently recomputed server-side**, per platform guidance.
6. Neither provider can report `passed` without actually verifying.
7. Missing attestation **does not block** a customer — it weights the risk engine.

## Verification

```
tsc --noEmit                   exit 0
next lint                      0 warnings, 0 errors
check-route-security.mjs       pass
vitest                         3,333 passed | 18 skipped | 0 failed
```

## Highest remaining risks

1. **The boundary is not wired in.** `/api/mpc/sign` still accepts an opaque digest with advisory context only. Built ≠ enforced.
2. **No iOS app.** Blocks A9's client half entirely.
3. **A5 P2P is per-instance.** Money-adjacent state in `globalThis`.
4. **A7 EDD absent.** A named regulatory requirement.

## Next

1. Wire `authorizeForSigning()` into `/api/mpc/sign` and `/api/wallets/transfer`.
2. A4 + A6 — small, no dependencies, immediate value.
3. A5 — durable P2P state machine.
4. A1 — iOS, the largest single piece.
