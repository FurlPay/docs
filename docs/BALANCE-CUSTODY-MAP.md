# FurlPay — Balance Source-of-Truth & Custody Map

> Canonical answer to: for every balance the app shows, where does it come from, what backs it, who has custody, and which key can move it.
>
> **Revised 2026-09-17.** The original 2026-07-14 revision is preserved below under "Historical finding (July 2026)" because its conclusion is still the reason this document exists — but two of its central claims are now out of date, and repeating them would misdirect the next audit.

---

## What changed since July 2026

| July 2026 claim | Status now |
| --- | --- |
| "`user.safeAddress` is the account's on-chain identity" | **Superseded.** `safeAddress` is now explicitly DEPRECATED in `store.ts` — a truncated sha256 that controls nothing. A real, signature-proven `User.linkedWallet` replaced it. |
| "The device wallet is unlinked to the account" | **Superseded.** `/api/wallet/link/challenge` + `/api/wallet/link/verify` exist: the server mints a single-use challenge, the device signs EIP-191, the server recovers the signer and requires it to equal the claimed address. Consumption is atomic (`kvSetNx`). |
| "Every balance is an internal ledger number" | **Still true for displayed balances**, and still the single most important fact on this page. Phase 2 reads exist in code but are stage-gated off. |

---

## The self-custody rollout ladder

Gated by `FURLPAY_SELF_CUSTODY_STAGE` (`lib/walletLink.ts::selfCustodyStage`). Stages are additive; each assumes the one below it is live.

| Stage | Name | State | What it changes |
| :-- | --- | --- | --- |
| **0** | Off | — | Balances are the ledger cache. No behaviour change. |
| **1** | Link device key | **Built** | `/api/wallet/link/{challenge,verify}` record `User.linkedWallet`, signature-proven server-side. **Changes no displayed money.** |
| **2** | On-chain reads | **Built, gated off** | `lib/services/usdcBalances.ts` reads live USDC on Base / Arbitrum / Solana for the linked address, tagged `asOf.source="chain"`, degrading per chain to a tagged cache when an RPC is unreachable. |
| **3** | Per-user deposits | **NOT built** | `webhooks/deposit-detect` still credits the shared demo singleton, not a per-user address. |
| **4** | Real gasless send | **Code complete, unconfigured** | Send tab spends from the linked address via device-signed EIP-3009. Gated additionally by the `self_custody_send` client flag. **Requires a funded fee-payer** — see below. |
| **5** | Separation guarantees | **NOT built** | Custody types kept as distinct rows; net worth an explicit aggregation. |

### What actually blocks a real USDC send today

Not cryptography. The signing path is real and complete:
`native-app/lib/wallet.ts` holds a secp256k1 key in Keystore/Keychain with a BIP-39 backup, `signTransferAuthorization` produces a genuine EIP-3009 authorization behind a biometric gate, and `x402Settler.ts::createEvmSettler` broadcasts `transferWithAuthorization` with confirmation-depth waiting and replacement-transaction tracking.

It is blocked on **configuration and funding**:

- `FURLPAY_KMS_PROVIDER` + `FACILITATOR_EVM_ADDRESS` — unset
- `ONCHAIN_SETTLEMENT_LIVE` flag — off (requires the two above)
- A **funded fee-payer wallet** to pay gas
- `FURLPAY_SELF_CUSTODY_STAGE >= 4` and the `self_custody_send` client flag

Until those exist, `/api/transfers/gasless` returns a clean 503 (`no on-chain relayer configured`) and the app surfaces it honestly. That is the correct behaviour, not a bug.

### Verified on-chain constants (2026-09-17)

Checked against live RPCs, not copied from a document:

| Chain | USDC | decimals |
| --- | --- | :--: |
| Base | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 6 |
| Arbitrum | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` | 6 |
| Solana | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | 6 |

The Solana reader **sums every token account** for the mint rather than reading `[0]` — an owner may hold several, and taking the first under-reports the balance.

### Mobile surfaces

`native-app/` (Expo, builds both iOS and Android) is the **only** mobile client. Two Kotlin/Swift prototypes that fabricated settlement (`"mock_signature_123"`, and a `submitPayment()` that slept and returned `true`) were archived out of the workspace on 2026-09-17 — see `../furlpay-prototypes-archive-2026-09-17/README.md`.

---

## Historical finding (July 2026)

*Preserved for context. Read the sections above first — where they disagree, they are newer.*

---

## TL;DR (the finding that changes everything)

**Every balance in FurlPay today is an internal ledger number with no on-chain custody backing.** There is no address anyone holds a key to that contains these funds. Specifically:

1. **All displayed balances are ledger entries**, seeded static for the demo account, empty for new users. Source of truth = the in-memory `UserAccount` store (Supabase write-through). Not on-chain, not custodial-with-reserves.
2. **`user.safeAddress` is not a real wallet.** It is `"0x" + sha256("furlpay:safe:" + key).slice(0,40)` (`store.ts:49-51`) — a truncated hash, **not derived from any private key**. No one can sign for it. USDC sent to it would be permanently stuck.
3. **The device wallet (`lib/wallet.ts`, shipped 95c905b) is the first and only real key/address in the entire system** — and it is a *different* address from `safeAddress`, unlinked to the account.

Therefore "send from my FurlPay balance using the device key" is currently a **category error**: the balance is not at the device address, not at any real address, and not backed by movable USDC. This must be resolved by choosing a real custody model, not by wiring code to the existing one.

---

## Per-balance map

| Balance shown | Where the UI reads it | Ultimate source | Backed by real funds? | Who has custody | Which key can move it |
| --- | --- | --- | --- | --- | --- |
| **Stablecoin balances** (USDC/USDT/EURC/XSGD…) | `GET /api/overview`, `/api/wallets` → `account.tokenBalances` | Seeded static (`store.ts:163-171`); credited by deposit webhook (`deposit-detect:131-137`) | **No** — ledger number | Nobody (ledger only) | **None** — `safeAddress` has no key |
| **Fiat balances** (USD/EUR/INR virtual accounts) | `GET /api/banking` → `account.fiatBalances` | Seeded static (`store.ts:172-191`); `localDetails` generated on provision (`banking:12-17`) | **No** — no BaaS integration; account numbers are formatted fakes | Nobody | N/A (fiat rail not integrated) |
| **Equity holdings** (AAPL/NVDA/VOO…) | `GET /api/investing/portfolio` → `account.holdings` | Seeded shares (`store.ts:192-198`); prices re-marked from live quotes (`portfolio:10-19`) | **Partial** — prices real (market data), *shares/positions are ledger*, no Alpaca custody | Nobody (ledger) | N/A (no broker link) |
| **Earn positions** (Morpho vaults) | `GET /api/earn` → `earnSummary()` | Vault APY/TVL real (`fetchMorphoVaults`); *user position* is ledger | **No** for the position | Nobody | N/A |
| **Credit line** | `GET /api/credit` | Derived from ledger balances; `drawn` is a fixed demo constant (`credit:53`) | **No** | N/A | N/A |
| **Rewards / cashback** | `GET /api/rewards` | Deterministic demo constants (`rewards:22`) | **No** | N/A | N/A |
| **Card spend / limits** | `account.cards[].limits` | Seeded (`store.ts:199-224`); `dailySpent` bumped on 3DS approve | **No** — demo PAN, no issuer | Nobody | N/A |
| **Device wallet USDC** | `app/wallet.tsx` → on-chain address | **Real** secp256k1 address (`lib/wallet.ts`) | **Yes, if funded** — real Arbitrum USDC at a real address | **The user** (self-custodial) | **The device key** (Keystore, biometric-gated) |

### The one real on-chain read that exists

`getUsdcBalanceWithBudget` (`services/chainBalance.ts`) reads real USDC `balanceOf` on Arbitrum — but it is used **only** for the card-auth JIT-funding decision (`webhooks/card-auth`), reads against `safeAddress` (which no one controls), and falls back to the ledger on any failure. It never updates the displayed balances. So even the one real read is against a dead address.

### How "deposits" work today

`deposit-detect` watches **treasury addresses** (`TREASURY_ADDRESS_BSC` etc. via `isWatchedAddress`), not per-user deposit addresses, and on a confirmed deposit it credits **`db.tokenBalances`** — the shared demo singleton (`deposit-detect:131`), not the account that owns the address. There is **no per-user deposit-address model**. (This also predates the per-user isolation fix and still writes to `db`.)

---

## Custody taxonomy — where FurlPay actually sits

Against the standard models:

- ❌ **On-chain balance at the user's wallet address** — no; `safeAddress` is a fake and holds nothing.
- ⚠️ **Custodial ledger balance** — closest to reality (balances are ledger rows), **but there are no real reserves and no custody backing them**. It's a *demo* ledger, not a funded custodial ledger.
- ❌ **Smart-account (AA) balance** — no smart account is deployed.
- ⚠️ **EIP-3009 authorization model** — the *rail* exists (`/api/transfers/gasless`) and the device wallet can sign, but it moves funds at the *device* address, not the account balance.
- ❌ **Hybrid (ledger + on-chain reserves)** — no reconciliation exists because there are no reserves.

**Verdict: internal (demo) ledger, zero custody backing, plus one unlinked real self-custodial wallet.**

---

## The decision required before linking (do NOT skip)

"Where does the user's FurlPay balance live, who has custody, which key moves it, and what is the canonical source of truth?" must be answered as a product+architecture decision. The three coherent target models:

| Target model | What the balance becomes | Who custodies | Key that moves funds | Implication |
| --- | --- | --- | --- | --- |
| **A. Self-custodial (device wallet IS the account)** | On-chain USDC at the device address | The user | Device key (Keystore) | `safeAddress` retired; balance = on-chain read of the device address; deposits go to a per-user address = the device address; no FurlPay custody, KYC/travel funded from user's own on-chain USDC. Cleanest fit for what's built; requires a "link/adopt device address" step + balance reads switched to on-chain. |
| **B. Custodial (FurlPay holds funds)** | Ledger claim on FurlPay-held reserves | FurlPay (omnibus/treasury) | FurlPay's treasury key | Real custody, MSB/e-money licensing, reserve accounting, withdrawal architecture. The device key becomes an *auth* factor, not a *spend* key. Heaviest compliance. |
| **C. Hybrid** | Ledger + on-chain reserve reconciliation | Both | Both, with reconciliation | Most powerful, most dangerous — needs the strongest financial controls (ledger↔chain reconciliation is a P0 control, not a feature). |

**Recommendation given what exists:** Model **A (self-custodial)** is the honest, buildable target — it matches the real wallet already shipped and avoids becoming a licensed custodian. Under Model A, the correct sequence for the "#2 linking" task is:

1. **Adopt the device address as the account's on-chain identity** — a server endpoint that records the device wallet address on the account (replacing the fake `safeAddress`), signature-proven (the device signs a challenge to prove key possession).
2. **Switch displayed balances to on-chain reads** of that address (via `chainBalance`-style readers per chain), with the ledger relegated to a cache with a freshness stamp.
3. **Point deposits at the device address** (per-user watched address) instead of the shared treasury credit.
4. **Retire the demo seed** for real accounts (new users already start empty; the demo seed stays only for the demo/guest user).
5. Only then is "send from my balance" meaningful — it's a device-signed EIP-3009 transfer from the address that *is* the balance, which is exactly what `lib/wallet.ts` already does.

Until this decision is made and step 1 exists, **the balance↔wallet link must not be built** — it would imply movability that the model can't honor.

---

## Cross-checks the reviewer flagged (status against code)

| Item | Status |
| --- | --- |
| Correct USDC contract per chain | Arbitrum native USDC correct (`chainBalance.ts:17`, `NETWORK_USDC` in x402.ts). Per-chain map exists; **verify each** before multi-chain send. |
| Native vs bridged USDC | Arbitrum uses native (Circle) — correct. Other chains **unverified**. |
| chainId / EIP-712 domain | Correct + proven (`wallet.test.ts` vs viem oracle). |
| validAfter/validBefore | Set (0 / now+3600) in `sendGasless`; server enforces window. |
| Secure nonce | `Crypto.getRandomBytes(32)` — CSPRNG. Good. |
| Replay / used-authorization | Server burns nonce atomically (`transfers/gasless:105`). Good. |
| Confirmation thresholds / finality | Server scales depth by amount (`requiredConfirmations`). Good. |
| Atomic-unit arithmetic | Device wallet uses BigInt (`usdcToAtomic`). **Main Send tab + several ledger routes still use JS `Number`** — P1. |
| Recipient checksum / full-address inspection | Full address shown pre-sign (Send tab + wallet). Checksum on derived address (EIP-55); **inbound recipient not checksum-validated**, only regex. |
| Address poisoning | Full-address display only; **no known-payee allowlist** — P1. |
| Root/instrumented device | **No detection** — P2. |
| Backup / recovery / device migration | BIP-39 phrase backup/import shipped; **no cloud/social recovery**; device loss + lost phrase = funds lost (self-custody reality). |
| Account deletion with on-chain assets | **Undefined** — needs policy (deletion must warn if the device wallet holds funds). |
| DB reconciliation vs on-chain | **Does not exist** — the core Model-A/C control still to build. |

---

## Bottom line

The app is a polished client over a **demo ledger with no custody**. The real self-custodial wallet is the first genuine piece of on-chain custody in the system, and it is currently an island. Linking it to "the FurlPay balance" is not a coding task — it is the moment FurlPay picks its custody model. **Recommendation: commit to self-custodial (Model A), build the address-adoption + on-chain-balance-read + reconciliation sequence, and treat ledger↔chain reconciliation as a P0 financial control, not a feature.**
