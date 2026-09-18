# FurlPay Comprehensive Codebase Audit & Architecture Report

**Date:** September 18, 2026  
**Scope:** Full monorepo audit covering Android Native OS & Mobile, Web Application & API (Next.js 15), Solana Smart Contracts (Anchor), Multi-platform SDKs, and E-Commerce Plugins.

---

## Executive Summary

A comprehensive architectural and defensive security review of the entire FurlPay monorepo was conducted. The codebase exhibits a high level of engineering discipline in several areas:
- **Android Native & Mobile:** Strong hardware-backed key custody using BIP-39/BIP-44, `expo-secure-store` with biometrics (`requireAuthentication: true`), heap memory zeroing (`priv.fill(0)`), ProGuard/R8 dead-code & log stripping, network security configuration with certificate pinning (ISRG root), and a fully native 8-module Kotlin 2.1 Wear OS/Phone application (`guardian`).
- **Web Application & Backend:** Next.js 15 App Router with rigorous middleware enforcing edge-verified JWTs, CSRF defenses comparing `Origin` to `Host`, distributed Upstash Redis rate limiting, API version negotiation, and strict production assertions on secrets.

However, several critical and high-severity vulnerabilities were identified across the smart contract implementation, e-commerce plugins, SDK protocol alignment, and API fallback configurations:
1. **[CRITICAL] Solana Escrow Smart Contract Fund Theft via Unvalidated Accounts (`auto_release`)**: Anyone can call `auto_release` on mature escrows and pass an attacker-controlled token account to drain escrow funds.
2. **[HIGH] Solana Actions Fallback to System Program (`1111111...`)**: An unset `MERCHANT_SOLANA_WALLET` causes SOL checkout transactions to route funds to the System Program address, burning customer funds.
3. **[HIGH] E-Commerce Plugin Webhook Signature Bypass on Empty Secrets (WooCommerce & Magento)**: When the webhook secret is unconfigured or empty, `hash_hmac` with an empty string can match an attacker's signature, allowing unauthorized order completion.
4. **[MEDIUM] Rate Limit IP Spoofing via `X-Forwarded-For` Leftmost Parsing**: If `x-real-ip` is absent, taking `split(",")[0]` allows attackers to bypass IP-based rate limiting by rotating custom headers.
5. **[MEDIUM] Insecure Content Security Policy (`'unsafe-eval'` in production)**: Web application CSP permits `'unsafe-eval'` in `script-src`.
6. **[MEDIUM] Unencrypted Local Database on Android (`expo-sqlite`)**: Transaction histories and swap journals are stored in plaintext SQLite on device storage.
7. **[MEDIUM] Non-Constant-Time Signature Comparison in Swift SDK**: `WebhookVerifier.swift` uses `elementsEqual` instead of a constant-time comparison, creating a timing side-channel.
8. **[LOW] Protocol Inconsistency Across SDK Webhooks**: Kotlin and JS SDKs enforce `t=...,v1=...` timestamped payloads, while Python, Swift, WooCommerce, and Magento use raw unversioned HMACs without replay protection.

---

## Vulnerability & Risk Summary Matrix

| ID | Component | Severity | Description | Remediation |
|---|---|---|---|---|
| **VULN-01** | `furlpay-programs/escrow` | **CRITICAL** | Missing owner check on `merchant_token_account` in `auto_release` allows arbitrary token drain | Add Anchor constraints checking `merchant_token_account.owner == escrow.merchant` and mint |
| **VULN-02** | `furlpay-programs/escrow` | **HIGH** | Unconstrained `token_vault` & accounts in `release_escrow` and `resolve_dispute` | Store `token_vault` PDA in `Escrow` struct and constrain accounts |
| **VULN-03** | `apps/web/lib/actions` | **HIGH** | `merchantWallet()` falls back to `111111...11` when `MERCHANT_SOLANA_WALLET` is unset | Fail closed in production by throwing when `MERCHANT_SOLANA_WALLET` is empty |
| **VULN-04** | `furlpay-plugin-woocommerce` | **HIGH** | Webhook verification fails open on empty `furlpay_webhook_secret` | Reject verification immediately if `empty($secret) \|\| empty($signature)` |
| **VULN-05** | `furlpay-plugin-magento` | **HIGH** | Webhook verification fails open on unconfigured secret | Explicitly check `$webhookSecret` presence before executing `hash_hmac` |
| **VULN-06** | `apps/web/lib/rateLimit` | **MEDIUM** | `clientIp()` reads leftmost `X-Forwarded-For`, enabling IP spoofing | Use trusted reverse proxy headers or parse the rightmost proxy-added client hop |
| **VULN-07** | `apps/web/next.config.js` | **MEDIUM** | `'unsafe-eval'` enabled in production Content Security Policy | Remove `'unsafe-eval'` in production builds |
| **VULN-08** | `native-app/lib/db.ts` | **MEDIUM** | SQLite database (`kv_cache`, `transactions`, `swaps`) stored unencrypted | Migrate to encrypted SQLite driver (`@op-engineering/op-sqlite` / SQLCipher) |
| **VULN-09** | `furlpay-swift` | **MEDIUM** | Swift `WebhookVerifier` uses non-constant-time string comparison | Use CryptoKit's constant-time comparison or HMAC byte equality |
| **VULN-10** | `furlpay-plugin-shopify` | **LOW** | `JSON.stringify(req.body)` used for HMAC check rather than raw request buffer | Capture raw request body stream before JSON parsing in Express |

---

## Detailed Component Audits

### 1. Android Native OS & Mobile Architecture

The repository contains two primary active Android/mobile components, plus supporting modules:
- **`native-app/`**: The canonical mobile app published to Google Play (`com.furlpay.app`), built with React Native 0.79.6 and Expo SDK 53 (New Architecture, Hermes enabled).
- **`guardian/`**: A native Kotlin 2.1 Android (phone) and Wear OS (Google Pixel Watch) app (`com.furlpay.guardian`) composed of 8 modular Gradle modules.
- **`apps/android/`**: The legacy Bubblewrap TWA. Appropriately deprecated via `DEPRECATED.md` and guarded in `build.sh`.

#### A. Key Management & Self-Custody (`native-app/lib/wallet.ts`)
- **Strengths:**
  - Standardized BIP-39 mnemonic generation using `Crypto.getRandomBytes(16)` (128-bit entropy).
  - SLIP-0010 Ed25519 (`m/44'/501'/0'/0'`) for Solana and BIP-44 Secp256k1 (`m/44'/60'/0'/0/0`) for EVM.
  - Storage backed by Android KeyStore via `expo-secure-store` with `requireAuthentication: true` and `WHEN_UNLOCKED_THIS_DEVICE_ONLY`.
  - Memory hygiene: `priv.fill(0)` is executed in `finally` blocks immediately after signing to erase secret key material from the heap.
  - Pinned token contract registry: Signs EIP-712 / EIP-3009 transfer authorizations only against hardcoded, pinned USDC contracts per network (`USDC_BY_NETWORK`), preventing malicious quote contracts from inducing blind token transfers.
- **Areas for Improvement:**
  - **Unencrypted SQLite Storage (`native-app/lib/db.ts`):** While private keys and session tokens live in Android KeyStore/SecureStore, financial metadata (transaction history, pending actions, swap journals) is stored unencrypted in local SQLite. On a rooted device or via physical extraction, cached transaction details are readable.

#### B. Native Hardening & Manifest Configuration (`native-app/android/`)
- **Manifest:**
  - `android:allowBackup="false"` prevents adb backup extraction.
  - `android:networkSecurityConfig="@xml/network_security_config"` enforces `cleartextTrafficPermitted="false"` across the app.
  - Pins Let's Encrypt / ISRG root anchors (`fk6IOKit1ild...`) for `furlpay.com` with expiration in 2027.
  - Dangerous permissions (`READ_EXTERNAL_STORAGE`, `WRITE_EXTERNAL_STORAGE`, `SYSTEM_ALERT_WINDOW`) are explicitly removed using `tools:node="remove"`.
- **ProGuard / R8 (`native-app/android/app/proguard-rules.pro`):**
  - Production builds enable minification and resource shrinking.
  - `assumenosideeffects` strips `Log.v`, `Log.d`, and `Log.i` to prevent sensitive financial addresses or request parameters from reaching `logcat`.
  - Reflection rules preserve WebAuthn/Passkeys (`androidx.credentials`, `com.google.android.gms.fido`) and SQLite/MapLibre JNI descriptors.
- **Signing Pipeline (`native-app/build-android.sh`):**
  - Protects against Expo's default debug signing in release builds by injecting `android.injected.signing.*` properties at build time.
  - Signing keystore and credentials reside in an external directory (`C:/Users/ashut/.furlpay-signing`), preventing accidental repository commits.

#### C. FurlPay Guardian — Kotlin Native & Wear OS (`guardian/`)
- **Architecture:** 8 clean modules separating domain, network, security, sync, and UI.
- **Key Storage (`guardian/core/security/KeystoreTokenStore.kt`):**
  - Native implementation of AES-256/GCM via `AndroidKeyStore`.
  - Thread-safe key generation with synchronized lock to avoid displacement race conditions.
  - Uses 12-byte random IVs and 128-bit authentication tags.
- **Manifest & Services (`guardian/app/wear/` & `guardian/app/mobile/`):**
  - Standalone Wear OS metadata (`com.google.android.wearable.standalone = true`).
  - Google Play tap-to-pay is protected; NFC and camera permissions declare `android:required="false"` to prevent device delisting on Google Play.

---

### 2. Smart Contract Audit (`furlpay-programs/programs/furlpay-escrow`)

The Solana program manages buyer-merchant escrows with automatic release timeouts and dispute mechanisms. Significant vulnerabilities were identified in the account validation logic:

#### Vulnerability Analysis: VULN-01 & VULN-02 (Arbitrary Token Withdrawal)
**Location:** [instructions/auto_release.rs](file:///c:/Users/ashut/OneDrive/Documents/Payment%20App/furlpay-programs/programs/furlpay-escrow/src/instructions/auto_release.rs#L6-L25) and [instructions/release_escrow.rs](file:///c:/Users/ashut/OneDrive/Documents/Payment%20App/furlpay-programs/programs/furlpay-escrow/src/instructions/release_escrow.rs#L7-L28)

**Vulnerable Code in `AutoRelease`:**
```rust
#[derive(Accounts)]
pub struct AutoRelease<'info> {
    #[account(
        mut,
        seeds = [b"escrow", escrow.payment_id.as_bytes()],
        bump = escrow.bump,
        constraint = !escrow.released @ EscrowError::EscrowAlreadyReleased,
        constraint = !escrow.disputed @ EscrowError::Unauthorized
    )]
    pub escrow: Account<'info, Escrow>,

    #[account(mut)]
    pub token_vault: Account<'info, TokenAccount>,

    #[account(mut)]
    pub merchant_token_account: Account<'info, TokenAccount>,

    pub token_program: Program<'info, Token>,
}
```

**Attack Vector:**
1. Once `auto_release_at` timestamp passes, anyone can call `auto_release` (no signer required).
2. The account `merchant_token_account` has no constraint verifying that its owner is `escrow.merchant` or that its mint is `escrow.mint`.
3. An attacker supplies their own token account as `merchant_token_account`.
4. The program performs CPI `token::transfer` using the escrow PDA signer, transferring `escrow.amount` directly to the attacker.
5. Similarly, in `create_escrow.rs`, `token_vault` is created without seeds and its address is not recorded in the `Escrow` struct, allowing arbitrary vault substitution in `release_escrow` and `resolve_dispute`.

**Remediation:**
Update `Escrow` struct in `state/escrow.rs` to persist `pub token_vault: Pubkey;`. In `AutoRelease`, add constraints:
```rust
#[account(
    mut,
    constraint = token_vault.key() == escrow.token_vault @ EscrowError::InvalidVault,
)]
pub token_vault: Account<'info, TokenAccount>,

#[account(
    mut,
    constraint = merchant_token_account.owner == escrow.merchant @ EscrowError::Unauthorized,
    constraint = merchant_token_account.mint == escrow.mint @ EscrowError::InvalidMint,
)]
pub merchant_token_account: Account<'info, TokenAccount>,
```

---

### 3. Web Application & API Audit (`apps/web`)

#### A. Edge Middleware & Authentication (`apps/web/src/middleware.ts`)
- **Authentication:** Edge JWT validation via `jose` with `tokenFromRequest` reading either `furlpay_session` cookie or `Authorization: Bearer`.
- **CSRF Defense:** `crossSiteCookieForgery()` compares `req.headers.get("origin")` to `req.headers.get("host")` (supporting apex and www domains) for mutating requests that authenticate via cookie.
- **Baseline Rate Limiting:** Applied to all mutating requests (150 req/min/IP) using Redis `kvIncr` sliding window, failing open gracefully during KV disruptions.

#### B. API Rate Limiter IP Extraction (`apps/web/src/lib/rateLimit.ts`)
**Location:** [lib/rateLimit.ts:87-93](file:///c:/Users/ashut/OneDrive/Documents/Payment%20App/apps/web/src/lib/rateLimit.ts#L87-L93)
```ts
export function clientIp(req: NextRequest | undefined): string {
  return (
    req?.headers.get("x-real-ip") ||
    req?.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ||
    "unknown"
  );
}
```
**Risk:**
When deployed in environments where `x-real-ip` is not set or when upstream proxies append client IPs to the right, reading `split(",")[0]` selects the client-controlled leftmost IP. An attacker can supply a spoofed `X-Forwarded-For: <random-ip>` header to bypass IP-based rate limiting on sensitive routes (e.g. OTP initiation, fee checks).

#### C. Solana Actions Merchant Address Fallback (`apps/web/src/lib/actions/solana.ts`)
**Location:** [lib/actions/solana.ts:169-173](file:///c:/Users/ashut/OneDrive/Documents/Payment%20App/apps/web/src/lib/actions/solana.ts#L169-L173)
```ts
function merchantWallet(): PublicKey {
  const configured = process.env.MERCHANT_SOLANA_WALLET;
  return new PublicKey(configured || "11111111111111111111111111111111");
}
```
**Risk:**
If `MERCHANT_SOLANA_WALLET` is not configured in production, `merchantWallet()` defaults to the System Program address (`11111111111111111111111111111111`). During a SOL checkout, `buildPaymentTransaction()` routes funds to `1111...1111`, resulting in permanent loss of funds.
**Remediation:**
Enforce fail-closed behavior in production:
```ts
if (!configured) {
  if (process.env.NODE_ENV === "production") {
    throw new Error("MERCHANT_SOLANA_WALLET must be set in production");
  }
  return new PublicKey("11111111111111111111111111111111");
}
```

#### D. Content Security Policy (`apps/web/next.config.js`)
**Location:** [next.config.js:12](file:///c:/Users/ashut/OneDrive/Documents/Payment%20App/apps/web/next.config.js#L12)
`script-src` includes `'unsafe-eval'`. While frequently used in development toolchains, leaving `'unsafe-eval'` in production weakens browser defenses against DOM-based XSS attacks.

---

### 4. Multi-Platform SDKs & Plugins Audit

#### A. WooCommerce Plugin (`furlpay-plugin-woocommerce/includes/class-furlpay-webhook.php`)
```php
public static function handle() {
    $payload = file_get_contents('php://input');
    $signature = $_SERVER['HTTP_X_FURLPAY_SIGNATURE'] ?? '';
    $secret = get_option('furlpay_webhook_secret', '');
    
    if (!self::verifySignature($payload, $signature, $secret)) {
        wp_die('Unauthorized', 'Webhook Error', 401);
    }
```
**Vulnerability:**
If the store administrator has not yet configured `furlpay_webhook_secret` (value is empty string `''`), `hash_hmac('sha256', $payload, '')` produces a predictable HMAC over the payload using empty key `""`. Any attacker calculating `hash_hmac('sha256', $payload, '')` or sending an empty signature will pass verification and mark orders as paid.
**Remediation:**
Fail closed when the secret is unset:
```php
if (empty($secret) || empty($signature)) {
    wp_die('Unauthorized', 'Webhook Error', 401);
}
```

#### B. Magento Plugin (`furlpay-plugin-magento/Model/FurlPayClient.php`)
Identical vulnerability: `verifyWebhookSignature` computes `hash_hmac` without checking whether `$webhookSecret` is empty.

#### C. Shopify Plugin (`furlpay-plugin-shopify/src/routes/webhook.ts`)
The webhook handler verifies against `const body = JSON.stringify(req.body)`. Because `JSON.stringify` does not guarantee original byte order or whitespace matching the raw HTTP payload, HMAC verification will intermittently fail on valid payloads, or open up parsing discrepancies. The raw body buffer must be used directly.

#### D. Swift SDK (`furlpay-swift/Sources/FurlPay/Webhooks/WebhookVerifier.swift`)
```swift
let expected = Data(mac).map { String(format: "%02x", $0) }.joined()
return expected.elementsEqual(signature)
```
`elementsEqual` is an early-terminating character comparison subject to timing attacks. CryptoKit's constant-time validation or byte buffer equality should be used instead.

#### E. SDK Signature Format Discrepancies
- **`furlpay-sdk-js` & `furlpay-kotlin`**: Implement the secure standard: `t=<timestamp>,v1=<hmac>` over `<timestamp>.<payload>` with 300-second drift tolerance.
- **`furlpay-python-sdk`, `furlpay-swift`, WooCommerce, Magento**: Expect a raw hex HMAC directly over `payload`, with no timestamp prefix or replay protection.
- **Recommendation:** Standardize all SDKs and plugins on the timestamped `t=...,v1=...` scheme.

---

## Actionable Remediation Roadmap

1. **Immediate (P0 - Security Critical):**
   - Fix Anchor constraints in `furlpay-programs/programs/furlpay-escrow/src/instructions/auto_release.rs`, `release_escrow.rs`, and `resolve_dispute.rs`. Bind `merchant_token_account.owner == escrow.merchant` and persist `token_vault` in the account state.
   - Patch `merchantWallet()` in `apps/web/src/lib/actions/solana.ts` to throw in production if `MERCHANT_SOLANA_WALLET` is empty.
   - Update WooCommerce and Magento webhook handlers to reject verification if the secret is empty.

2. **High Priority (P1 - Security & Reliability):**
   - In `apps/web/src/lib/rateLimit.ts`, resolve client IP using trusted reverse proxy configurations rather than picking the leftmost `X-Forwarded-For` address.
   - Remove `'unsafe-eval'` from `apps/web/next.config.js` CSP for production builds.
   - Update `furlpay-swift` WebhookVerifier to use constant-time comparison.

3. **Medium Priority (P2 - Architecture & Alignment):**
   - Harmonize webhook verification across Python SDK, Swift SDK, and plugins to use the timestamped `t=...,v1=...` format.
   - Evaluate migrating `native-app/lib/db.ts` SQLite storage to `@op-engineering/op-sqlite` with SQLCipher encryption.
