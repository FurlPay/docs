# Custody disclosure — how FurlPay signs, and who can move your money

**Last verified against the code: 2026-09-13.**

This document exists because the public site said something about your money that
the code does not do. It said FurlPay uses **"2-of-2 MPC custody"** — that your
signing key is split, and that neither FurlPay nor your device can move funds
alone. That claim appeared on the trust pages, the pricing pages, the comparison
pages, the careers page, and — most seriously — in the **Terms of Service**, the
**Risk Disclosure** and the **Refund Policy**.

It was not true, and `README.md` and `PROGRESS.md` had already recorded that
internally. This file is the correction, and the public pages now point here.

---

## What actually happens when something is signed

FurlPay has **two** signing paths. They have **different custody models**, and
which one you are using depends on which client you are using.

### 1. Mobile app — on-device key. Genuinely self-custodial.

`native-app/lib/signing/deviceKeySigner.ts`

A BIP-39 seed is generated on your phone from device entropy and stored in
biometric-gated secure storage (Android Keystore / iOS Keychain). **It is never
transmitted.** FurlPay's servers never see it and cannot reconstruct it.

- **Who can move funds:** you, with your device and your biometric.
- **If FurlPay disappears:** your key still works. The seed is yours.
- **If you lose the device and your backup:** the funds are unrecoverable. That
  is the real cost of self-custody, and it is not softened anywhere in this app.
- **Known limitation:** the phone both holds the key *and* draws the
  confirmation screen, so `trustedDisplay` is `false`. A compromised phone can
  show you one thing and sign another. A hardware signer is what fixes this; we
  do not have one.

### 2. Web app — remote enclave signer (Turnkey). **CUSTODIAL.**

`apps/web/src/lib/services/turnkey.ts` → `/api/mpc/sign`

Turnkey signs a digest inside a secure enclave and returns a **complete
signature**. Your device contributes **no key share**. The device verifies that
the returned signature recovers to the expected address — that is a correct and
worthwhile check, but it is *verification*, not *participation in a threshold
scheme*.

- **Who can move funds:** anything holding the Turnkey organisation credential,
  subject to Turnkey's own enclave policy. That includes FurlPay.
- **Therefore: this is custodial.** Not "self-custody leaning", not "2-of-2",
  not "no single party can move funds alone".
- **Why the distinction is not cosmetic:** custodial VDA custody changes
  FIU-IND scope, insurance requirements, and balance-sheet treatment. It also
  changes what you should be willing to hold with us.

---

## What *is* 2-of-2 — the policy path, not the key

The thing that genuinely has multiple independent gates is **authorisation**,
not key material. A signature request must survive three checks, each of which
can fail without opening the others:

1. **The client** refuses to request an out-of-policy signature.
2. **`/api/mpc/sign`** refuses to forward one (`apps/web/src/lib/services/policyEngine.ts`).
3. **Turnkey's enclave policy** refuses to produce one.

That is a real defence-in-depth property and it is worth describing. It is just
not key-share splitting, and calling it "2-of-2 MPC" conflated the two.

---

## Passkeys: what they do and do not do

WebAuthn passkeys (`apps/web/src/lib/webauthn.ts`) are used for
**authentication** — login, session establishment, and step-up re-auth for
sensitive actions such as card credential reveal.

Passkeys **do not currently sign transactions.** The passkey proves it is you
asking; the signature itself still comes from one of the two paths above. Any
copy implying your passkey *is* your signing key is wrong.

---

## Balances shown in the web app are ledger entries

`docs/BALANCE-CUSTODY-MAP.md` covers this in full, and it remains true:

`user.safeAddress` is **not a deployed Safe and not an account anyone holds keys
to**. It is `"0x" + sha256("furlpay:safe:" + key).slice(0, 40)` — a deterministic
placeholder (`apps/web/src/lib/store.ts`). Web-app balances are internal
double-entry ledger numbers, not on-chain holdings, and there is no automated
reconciliation between the two yet.

---

## The honest one-liner

> FurlPay uses passkey authentication and a three-layer signing policy. On
> mobile, your key is generated and held on your device. On the web, signing is
> performed by a policy-gated secure enclave operated by Turnkey — which means
> web signing is custodial.

Use that. Do not reintroduce "2-of-2 MPC", "no single party can move funds
alone", "your key is split", or "self-custody" as a description of the **web**
path.

---

## What would make the original claim true

Real 2-of-2 threshold signing needs a key share on the user's device that is
required to produce a signature — e.g. a TSS/MPC scheme where the device share
and a server share each compute a partial signature and neither is sufficient.
That is Phase 5 of the roadmap and it is **not built**. Until it ships and is
verified end-to-end, the language above stands.

A test guards this: `apps/web/src/lib/__tests__/custodyClaims.spec.ts` fails the
build if "2-of-2 MPC" custody language reappears on a public page.
