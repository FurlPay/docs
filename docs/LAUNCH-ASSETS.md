# FurlPay: Help Center Content & App Store Asset Specification

> Launch checklist: the 15 help-center articles mapped to shipped features, and store asset dimensions verified against Google/Apple requirements, July 2026.

---

## Part 1: Help Center articles (15)

Each article maps to a live feature — nothing here documents vaporware. The
support page (`/support`) categories match this structure one-to-one.

### Onboarding & Security
| # | Article | Grounded in |
| :--- | :--- | :--- |
| 1.1 | Getting Started with FurlPay | Signup + OTP/passkey login, first deposit flow |
| 1.2 | Identity Verification (KYC) Guide | `kycTier` 0-3 progressive tiers, Sumsub/Persona flows, co-pilot "check my KYC status" |
| 1.3 | Passkeys & Biometrics | Web passkeys + native Credential Manager passkeys (`android:apk-key-hash` origin), biometric app lock |
| 1.4 | Privacy & GDPR | Data storage summary, deletion requests via support ticket (KYC category routes to compliance feed) |

### Transactions & Stablecoins
| # | Article | Grounded in |
| :--- | :--- | :--- |
| 2.1 | Understanding Transaction Statuses | Real states: `prepared`, `pending`, `settled`, `failed`, `rejected` (ops event feed) — NOT "escrow", which is not a ledger state |
| 2.2 | Gasless Transfers | The EIP-3009 gasless rail (`/api/transfers/gasless`) — fee payer sponsors gas; explorer links via `/api/chain/tx/{hash}` |
| 2.3 | Deposits & Withdrawals | Deposit detection webhooks, off-ramp via `/api/offramp` |

### Virtual Cards
| # | Article | Grounded in |
| :--- | :--- | :--- |
| 3.1 | Managing Virtual Cards | `/api/cards/settings`: labels, per-purchase/daily limits, channels |
| 3.2 | Freezing & Unfreezing | Card freeze via app, web, or co-pilot (human-confirmed); unfreeze requires the same confirmation |
| 3.3 | Troubleshooting Declines | Frozen card, limit exceeded, 3DS2 approval pending (`/approvals`, push notification when FCM is configured) |

### Developer Support & x402
| # | Article | Grounded in |
| :--- | :--- | :--- |
| 4.1 | SDK & API Integration | Published SDKs (`@furlpay/*` on npm), webhook HMAC verification rules |
| 4.2 | x402 Monetization | `@furlpay/gateway` (npm) — v1 `X-PAYMENT` and v2 `PAYMENT-SIGNATURE`/`PAYMENT-REQUIRED` headers, facilitator at `/api/x402/facilitator` |

### Travel & Refunds
| # | Article | Grounded in |
| :--- | :--- | :--- |
| 5.1 | Booking Travel with Stablecoins | Web funnel + native app's one-time-code browser handoff |
| 5.2 | Cancellations & Refunds | Refund windows per rate type; stablecoin credit handling |

Plus one index page. Publish under the existing docs portal so articles are
version-controlled with the code that they describe.

---

## Part 2: App store visual assets (verified July 2026)

### Google Play (Android) — required for the pending `com.furlpay.app` upload

| Asset | Dimensions | Requirements |
| :--- | :--- | :--- |
| App icon | 512 x 512 px | 32-bit PNG with alpha, max 1 MB |
| Feature graphic | 1024 x 500 px | REQUIRED to publish. JPEG or 24-bit PNG, no alpha |
| Phone screenshots | 1080 x 1920 px recommended | Min 2, max 8. JPEG or 24-bit PNG, no alpha. Each side 320-3840 px, max 2:1 aspect ratio |
| 7-inch tablet | 1080 x 1920 px | Optional but recommended |
| 10-inch tablet | 1200 x 1920 px | Optional |

### Apple App Store (iOS) — future; the app is Android-only today

| Asset | Dimensions | Requirements |
| :--- | :--- | :--- |
| App icon | 1024 x 1024 px | PNG, RGB, NO alpha channel |
| 6.9-inch iPhone screenshots | 1320 x 2868 px portrait | Required class (2026 flagship: iPhone 17 Pro Max, not 16 Pro Max as older drafts said). Also accepted: 1290 x 2796, 1260 x 2736. Min 1, max 10. PNG or JPEG only, exact dimensions, no alpha |
| 13-inch iPad screenshots | 2064 x 2752 px | Only if iPad support is enabled |

Rejection triggers to avoid: alpha channels where prohibited, off-spec
dimensions (Apple has zero tolerance), device bezels in Play screenshots
combined with text-heavy overlays.

### Visual standards

- Background: pure black `#000000` (OLED), matching the app's `bg-oled` theme.
- Primary accent: the brand red used for logo elements and primary actions.
- Positive states: the neon accent used in-app for success/settled indicators.
- Colors in this section must be taken from `apps/web/tailwind.config.ts` and
  `native-app/lib/theme.ts` — not hardcoded in design tools from memory.
- Screenshots: full-bleed real UI (no fake data on screen: use the demo
  account), no device bezels, one short benefit caption per screenshot.
- App icon: existing `native-app/assets/icon.png` is the source of truth; the
  512 px Play icon is exported from it.

---

Sources: [Google Play screenshot/graphic requirements](https://appscreens.com/google-play-screenshot-sizes), [Play feature graphic specs](https://appscreenshotstudio.com/blog/play-store-feature-graphic-vs-screenshots-2026-specs), [Apple screenshot specifications](https://developer.apple.com/help/app-store-connect/reference/app-information/screenshot-specifications/), [App Store sizes 2026](https://screenhance.com/blog/app-store-screenshot-dimensions-2026).
