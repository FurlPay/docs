# FurlPay — Self-Custodial USDC Wallet: Design & Migration Plan

> The design that must exist before code (per the custody decision, step 10). Company architecture = **Hybrid (Model C)**; the immediate USDC wallet = **Self-Custodial (Model A)**, built so regulated custodial products slot in later without rework. Prereq reading: [BALANCE-CUSTODY-MAP.md](./BALANCE-CUSTODY-MAP.md).

**Guiding invariant:** for a `SELF_CUSTODIAL_ONCHAIN` balance, **the blockchain is the source of truth.** Any stored number is an explicitly-stamped cache and must never override an on-chain read. FurlPay never holds the key.

---

## 1. Domain model

### 1.1 Custody types (new enum — the anti-ambiguity rule)

```ts
// apps/web/src/lib/custody.ts  (new)
export type CustodyType =
  | "SELF_CUSTODIAL_ONCHAIN" // user's device key controls it; chain is truth
  | "REGULATED_CUSTODIAL"    // held by FurlPay/partner under license (future)
  | "BANK_DEPOSIT"           // BaaS/partner fiat account (future)
  | "CARD_BALANCE"           // issuer-held spending balance (future)
  | "INVESTMENT_ASSET"       // brokerage-custodied equities (future)
  | "REWARD_CREDIT"          // promotional, NOT customer cash
  | "PENDING_SETTLEMENT";    // in-flight, not yet final

export interface Balance {
  custody: CustodyType;
  token: string;          // USDC, EURC, …
  chain?: string;         // for on-chain custody
  /** Atomic units as a STRING (bigint-safe). Never a JS number for money. */
  atomic: string;
  decimals: number;
  /** For on-chain: block/time the read reflects. Absent = not yet read. */
  asOf?: { source: "chain" | "cache"; at: string; block?: string };
  spendable: boolean;     // false for rewards, pending, investment
}
```

**UI rule:** these are never summed into one "balance" except in an explicitly-labelled **net-worth** view. Spendable actions read only `spendable === true` balances of a single custody type.

### 1.2 Account changes

```ts
// User (apps/web/src/lib/store.ts) — additive, non-breaking
interface User {
  // ... existing ...
  /** DEPRECATED placeholder (truncated SHA-256, controls nothing). Kept for
   *  back-compat reads; never used for on-chain ops once linkedWallet exists. */
  safeAddress: string;
  /** The user's verified self-custodial address (device key). Null until linked. */
  linkedWallet?: {
    address: string;          // EIP-55 checksummed
    linkedAt: string;
    proofNonce: string;       // the challenge that was signed (audit)
  };
}
```

`tokenBalances`/`fiatBalances`/`holdings` become **caches**, each row gaining a `custody` tag and `asOf`. Self-custodial USDC rows are derived from chain reads; other rows keep their current (demo/partner) semantics until those products are real.

---

## 2. API contract

| Endpoint | Method | Auth | Purpose |
| --- | --- | --- | --- |
| `/api/wallet/link/challenge` | POST | session | Mint a one-time nonce for the device to sign (kv, 5-min TTL, reuses `setChallenge` pattern). Returns `{ nonce, message }`. |
| `/api/wallet/link` | POST | session | Body `{ address, signature }`. Server `verifyMessage`-recovers the signer from the stored nonce message; if it equals `address`, records `user.linkedWallet`. Idempotent; rejects if a *different* address is already linked (require explicit unlink). |
| `/api/wallet/balances` | GET | session | Returns `Balance[]` for the linked address — **on-chain reads** per supported chain, each stamped `asOf.source="chain"`, with a `cache` fallback stamped honestly on RPC failure. |
| `/api/wallet/deposit-address` | GET | session | Returns the linked address as the user's deposit target (replaces treasury-watch crediting for self-custodial deposits). |
| `/api/transfers/gasless` | POST | session | **Unchanged** — already the real EIP-3009 relay; now `from` = linked address, which the balance reads also reflect. |

**Message format for the link proof** (EIP-191 personal_sign, human-readable):
```
FurlPay wallet link
address: <address>
nonce: <nonce>
issued: <iso8601>
```
Signed by the device via a new `lib/wallet.ts::signMessage(nonce)` (EIP-191: `keccak256("\x19Ethereum Signed Message:\n" + len + msg)`, then secp256k1 — same primitives already in `lib/wallet.ts`, add a 20-line `signMessage`).

---

## 3. Migration plan (incremental, reversible)

**Phase 1 — Types + link (no behavior change to balances).** Add `custody.ts`, `User.linkedWallet`, the challenge + link endpoints, and `lib/wallet.ts::signMessage`. App gains a "Link this device as my wallet" action in `app/wallet.tsx`. Balances still read from the ledger cache. *Nothing about displayed money changes yet — this only records the verified address.*

**Phase 2 — On-chain balance reads.** Add per-chain USDC readers (generalize `services/chainBalance.ts` beyond Arbitrum). `/api/wallet/balances` returns real on-chain USDC for the linked address, tagged `SELF_CUSTODIAL_ONCHAIN`, `asOf.source="chain"`. The app's wallet screen shows on-chain truth; the main dashboard keeps showing the ledger cache but **labelled by custody type** so the two are never confused.

**Phase 3 — Deposits to the user address.** `/api/wallet/deposit-address` returns the linked address; the deposit-detect webhook, for self-custodial users, stops crediting the shared `db` singleton and instead the balance is simply the on-chain read (no ledger write needed — chain is truth). Treasury-watch crediting remains only for the demo user.

**Phase 4 — Reconciliation + real Send.** Ledger↔chain reconciliation job (below) for any cached self-custodial number. Main Send tab spends from the linked address via `sendGasless` (device-signed EIP-3009, atomic integers). Retire the demo seed for real accounts (already empty; assert no seed leaks into non-demo accounts).

**Phase 5 — Separation guarantees.** Ensure `REGULATED_CUSTODIAL`/`BANK_DEPOSIT`/`CARD_BALANCE`/`INVESTMENT_ASSET` remain distinct rows with their own (future) custody backends; add the net-worth view as an explicit aggregation, never a spendable number.

Each phase ships behind the existing `INTEGRATION_MODE`/`NODE_ENV` gates; self-custodial reads are safe in prod (read-only), real settlement stays gated on the relayer.

---

## 4. Reconciliation (the P0 control, not a feature)

For every cached `SELF_CUSTODIAL_ONCHAIN` balance and every in-flight transfer:
- **Balance drift:** periodic (and on-open) compare cache vs a fresh chain read; on divergence the chain wins and the cache is corrected + a `recordSecurityEvent("balance_drift")` is logged.
- **Transfer lifecycle:** `PENDING_SETTLEMENT` rows poll the relay's `statusPath` / chain confirmations until final; a submitted-but-unconfirmed transfer never shows as settled, and a dropped/replaced tx transitions to failed (nonce already burned server-side → honest retry re-signs).
- **Never** does a UI success precede on-chain finality (already true in `sendGasless`).

---

## 5. Threat model

| Threat | Vector | Mitigation |
| --- | --- | --- |
| **Link spoofing** — attacker links a victim's address | POST `/api/wallet/link` with someone else's address | Requires a signature over a server-minted, single-use, short-TTL nonce, recovered to == claimed address. Can't sign without the key. |
| **Link replay** | Reuse a captured signature | Nonce is `takeChallenge`-consumed (one-shot); message embeds nonce+issued. |
| **Address swap after link** | Attacker relinks to their address to divert deposits | Relink to a *different* address requires explicit unlink (which itself should require step-up / device proof). |
| **RPC lies / MITM** | Malicious/compromised RPC returns fake balance | Balance is display-only; **spending** is gated by the device signature + the relay's own on-chain verification. A lying RPC can misreport a number, not move funds. Multi-RPC read for high-value flows (P2). |
| **Reconciliation drift → double-spend illusion** | Cache says funds exist that the chain doesn't | Chain-wins reconciliation; spendable amount derives from a fresh read at send time, not the cache. |
| **Deposit misattribution** | Crediting the wrong account | Removed — self-custodial deposits are the on-chain balance of the user's own address; no cross-user ledger credit. |
| **Key exfiltration** | Rooted device / backup extraction | Key in Keystore-backed SecureStore, biometric-gated, `allowBackup=false`, FLAG_SECURE on seed. Root detection is P2. Self-custody = user bears residual device risk (disclosed). |
| **Lost device / lost phrase** | User loses access | BIP-39 backup shipped; **no recovery beyond the phrase** (self-custody reality) — must be disclosed in UI + account-deletion policy must warn if funds remain. |

---

## 6. Rollback strategy

- Phases are additive; each is independently revertible by feature-flag (`FURLPAY_SELF_CUSTODY_STAGE = 0|1|2|3|4`).
- `linkedWallet` is additive to `User`; reverting a phase never deletes it (it's user-owned truth), it only stops *reading* from it.
- Balance reads: if the on-chain path misbehaves, flip the stage flag down — the ledger-cache path (current behavior) still exists and serves.
- No destructive migration: the demo seed and `safeAddress` are left in place (deprecated), never rewritten, so any rollback lands on today's working state.

---

## 7. Affected files

**New:** `apps/web/src/lib/custody.ts`, `apps/web/src/app/api/wallet/link/challenge/route.ts`, `apps/web/src/app/api/wallet/link/route.ts`, `apps/web/src/app/api/wallet/balances/route.ts`, `apps/web/src/app/api/wallet/deposit-address/route.ts`, `native-app/lib/wallet.ts::signMessage`, tests.

**Modified:** `apps/web/src/lib/store.ts` (User.linkedWallet, custody tags), `apps/web/src/lib/services/chainBalance.ts` (multi-chain reader), `apps/web/src/app/api/webhooks/deposit-detect/route.ts` (skip singleton credit for self-custodial), `native-app/app/wallet.tsx` (link action + on-chain balance), `native-app/app/(tabs)/send.tsx` (Phase 4: spend from linked address, atomic ints).

**Explicitly untouched:** the custodial/fiat/card/investment rows (kept distinct for future regulated products); `/api/transfers/gasless` (already correct).

---

## 8. Test plan (per phase, before merge)

- **P1:** `signMessage` EIP-191 vector vs viem `verifyMessage` (oracle test, same pattern as `wallet.test.ts`); link endpoint accepts a valid proof, rejects wrong-address / reused-nonce / expired-nonce.
- **P2:** balance reader returns atomic-string + `asOf.source`; RPC-failure path returns cache stamped `source:"cache"` (never silently "chain").
- **P3:** deposit to linked address reflects in `/api/wallet/balances` on next read; no `db` singleton mutation for self-custodial users.
- **P4:** reconciliation corrects a drifted cache (chain wins) + logs the event; send spends atomic integers, no float anywhere in the money path (grep-gate `Number(` in wallet/send money lines); pending never shows settled.
- **Regression:** two distinct accounts see distinct linked wallets/balances (BOLA guard holds).

---

## 9. Regulatory perimeter (gate before real settlement at scale)

Self-custody ≠ unregulated. FurlPay operates a relayer, may charge fees, and intermediates swaps/bridges/travel/fiat — each can trigger obligations. **Before enabling real USDC settlement at scale, produce a jurisdiction-by-jurisdiction perimeter map (US, EU, UK, UAE, HK, SG, IN)** classifying each feature as: launchable under self-custody / needs a licensed partner / needs a FurlPay-held license. That map — not this doc — gates flipping `INTEGRATION_MODE=live` for the public. Evolution: self-custodial now → licensed partners for custodial/fiat/card → FurlPay's own licenses in chosen jurisdictions later.

---

## Implementation note

Build strictly in phase order, each behind `FURLPAY_SELF_CUSTODY_STAGE`, each with its tests green before the next. Phase 1 changes **no displayed money** — it only records the verified address — so it is the safe first increment.
