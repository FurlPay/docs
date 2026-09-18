# Furlpay Security Architecture

Furlpay implements a defense-in-depth model combining cryptographic identity,
a 2-of-2 MPC signing scheme, real-time risk scoring, step-up MFA, and an
off-chain policy guard for on-chain actions. This document describes the model
and how it maps to the code. Integrations run in mock/sandbox mode by default.

## 1. Cryptographic identity layer

- Passwordless authentication using FIDO2/WebAuthn passkeys. The private key
  never leaves the device secure enclave and is unlocked by local biometrics.
  No passwords, no SMS OTP (SIM-swap resistant). Session is an HttpOnly cookie
  minted on passkey login (`/api/auth/webauthn`), enforced by middleware.
  Live seam: Dynamic.xyz / Web3Auth.

- 2-of-2 MPC key management (`@furlpay/security/mpc`):
  - Key Share A is client-side, bound to the passkey/device.
  - Key Share B is server-side in an HSM.
  - `clientPartialSign` produces the user partial; `serverCoSign` produces the
    HSM partial and combines them. The full key is never assembled.
  - The server share refuses to co-sign unless the off-chain policy gate
    approved the action. Live seam: Turnkey / Fireblocks WaaS.

## 2. Step-up multi-factor authentication

High-value or sensitive actions require a second factor before signing
(`@furlpay/security/stepUp`):

- Transfers or withdrawals at or above USD 5,000.
- Setting card limits, changing settlement priority, adding a payee, or
  revealing card credentials.

Factors:

- FIDO2/WebAuthn hardware keys (YubiKey, Titan) as phishing-resistant MFA.
- OATH-TOTP authenticator apps as backup (`@furlpay/security/totp`), a
  dependency-free RFC 6238 implementation verified against the RFC test vector.
  Enrollment and verification at `/api/security/mfa`; the TOTP secret is never
  returned to the client.

Enforced today in `POST /api/wallets/transfer`: transfers >= USD 5,000 are
rejected with `stepUpRequired` until a valid TOTP code is supplied.

## 3. Real-time fraud and risk engine

`@furlpay/security/risk` (Sardine seam) scores each sensitive action from
device and behavioral signals:

- Behavioral biometrics: automation / remote-access tool detection.
- Device fingerprinting: emulator/VM, VPN/proxy, new-device signals.
- Outcome gates signing: allow, challenge, or block. Automation/remote-access
  hard-blocks (tech-support scam pattern).

Compliance screening is handled by `@furlpay/compliance` (AML wallet screening,
MiCA/GENIUS routing, FATF Travel Rule). Live seams: Sardine, TRM Labs,
Chainalysis, Persona/Sumsub, Notabene.

## 4. On-chain smart contract safeguards

Learning from the June 2026 Gnosis Pay exploit (an on-chain delay-module
signature-verification flaw that drained Safe wallets), Furlpay uses a hybrid
guard:

- Off-chain policy guarding: the server MPC share only co-signs after the
  policy engine passes (spending limits, merchant categories, velocity,
  sanctions). A client-side exploit cannot move funds alone.
- Standardized, audited Safe{Core} contracts only; no custom unvetted delay
  modules.
- Continuous monitoring aligned to BSSC 2026 standards, with static analysis
  (Slither, Mythril) before any upgrade.

## 5. Web application shields

- Session gating via Next.js middleware; public allowlist for auth, webhooks,
  checkout, and signup KYC.
- Outbound transfers require a valid signature (shape + payload binding).
- Card credential reveal requires biometric re-authentication.
- Webhooks are signature-verified (HMAC-SHA256) and idempotent by event id.
- API responses project only required fields; secrets (TOTP) and device data
  are never exposed through read endpoints.
- Production edge: Cloudflare WAF (DDoS mitigation, bot protection, TLS 1.3).

## Integration summary

| Layer | Provider (live seam) | Benefit |
| --- | --- | --- |
| Authentication | Dynamic.xyz / Web3Auth | Passwordless, phishing-resistant WebAuthn |
| Key custody | Turnkey / Fireblocks | 2-of-2 MPC, no single key leak |
| Fraud and risk | Sardine | Device fingerprinting, behavioral biometrics |
| Compliance | TRM Labs / Chainalysis | Live AML risk scoring |
| Edge | Cloudflare WAF | DDoS, bot protection, TLS 1.3 |

## Reporting

This is a prototype, not a production financial product. To report a security
issue in this repository, open a private security advisory or contact the
maintainer directly. Do not submit secrets through any form or issue.
