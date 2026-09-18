# FurlPay — verified claims register

**Last verified 2026-09-13.** Every row was checked against the repository or
Solana mainnet on that date. This file exists so that anything FurlPay says to a
grant committee, an investor, or a user can be traced to evidence before it is
sent.

Use it as the source for external copy. If a claim is not in the GREEN table,
it does not go in an email.

---

## Why this file exists

Eight partnership and grant emails were sent on 2026-09-13. Reviewing them
against the codebase afterwards found several claims that are wrong or
materially incomplete. They are listed in §4 with corrections.

A grant application is a quasi-formal document. Overstating capability is the
kind of thing that disqualifies an applicant when diligence finds it — and the
diligence here is easy, because the repository is open source and the chain is
public. A correction sent within days reads as rigor. The same correction
extracted during review reads as something else.

---

## 1. GREEN — verifiable claims

Safe to state externally. Evidence column says how to check.

| Claim | Evidence |
| --- | --- |
| FURL is live on Solana mainnet, Token-2022 | `getAccountInfo` — owner is `TokenzQd…PxuEb` |
| Mint `FYatJq71zqvaQNLLtbBy4hQcsV25dMATYfCSVBzV3gY1` | Public on any explorer |
| 1,000,000,000 total supply, 6 decimals | `getMint` — supply 1e15 base units |
| **Freeze authority is null** — no account can ever be frozen | `getMint` — `freezeAuthority: null` |
| No transfer hook, no transfer fee, no permanent delegate, no default-frozen state | TLV parse of the mint account; `scripts/audit-mainnet.ts` asserts each absence |
| Extensions are exactly MetadataPointer + TokenMetadata | Same audit, offset 166 |
| Metadata is on-chain and Arweave-backed | `getTokenMetadata`; URI returns 200 `application/json` |
| `CanonicalFURL.sol` — ERC-20 spoke with 6-decimal parity, `MAX_SUPPLY` cap, `ERC20Permit`, `AccessControlDefaultAdminRules` | `packages/contracts`, **20 tests** |
| Cross-chain supply invariant reconciler — BREACH on unbacked spoke supply, WARN on in-flight | `packages/furl-token/src/supplyInvariant.ts`, **14 tests** |
| NTT deployment configuration prepared (Solana locking, EVM burning, 5M FURL/24h rate limits) | `packages/contracts/deployment.yaml` — **not deployed** |
| Hand-written SLIP-0010 ed25519 derivation, no SDK on the signing path | `native-app/lib/solana/slip10.ts`, verified against official spec vectors |
| Solana + EVM accounts from one mnemonic, multi-account | `native-app/lib/accounts/manager.ts`, **18 tests** |
| SPL/Token-2022 **instruction construction** and ATA derivation | `native-app/lib/solana/spl.ts`, **23 tests**, vectors cross-checked against `@solana/spl-token` |
| x402 HTTP-402 agentic payment protocol | `apps/web/src/lib/x402.ts`, `x402Facilitator.ts` |
| Solana Actions / Blinks payment endpoint | `apps/web/src/app/api/actions/pay/[orderId]/route.ts` |
| Public token + supply APIs with chain-derived circulating supply | `/api/token/furl`, `/supply`, `/supply/cross-chain` |
| **5,231 tests passing** across four workspaces | contracts 56 · furl-token 75 · native-app 1,557 · web 3,543 |
| TypeScript clean in every workspace | `tsc --noEmit` |
| Native app: **69 route files**, **10 locales** | `find app -name "*.tsx"`; `ls lib/i18n/locales` |
| iOS project exists and is prebuilt | `native-app/ios/FurlPay.xcodeproj` |

---

## 2. AMBER — true but requires a qualifier

Stating these without the qualifier is misleading.

| Claim | Required qualifier |
| --- | --- |
| "On-device SPL token transfers" | Instruction construction, ATA derivation, balance checks and confirmation are built and tested. **Transaction message serialization is not implemented, so the app cannot yet broadcast a transfer.** |
| "1B fixed supply" | Fixed by published intent. **The mint authority is ACTIVE**, so the cap is not yet runtime-enforced. |
| "Transaction simulation" | `lib/txscan/` exists; describe what it actually does, not as a Blockaid-equivalent unless it is one. |
| "Staking" | `app/staking.tsx` exists. Confirm whether it settles against a live protocol before calling it a shipped feature. |
| "Merchant checkout gateway" | The session and payment-construction layer is real. Settlement depends on rails below. |
| "69 screens" | 69 route files; some are sub-routes rather than distinct screens. |

---

## 3. RED — do not claim

Each is gated by a feature flag carrying an `unimplemented` reason, or is
blocked outright.

| Do not claim | Reality |
| --- | --- |
| **Visa card settlement rails** | `CARD_ISSUING_LIVE` is off with an `unimplemented` reason. No card issuer is contracted. |
| Banking / ACH / SEPA / UPI rails | `BANKING_LIVE` off — no banking partner. |
| On-ramp or off-ramp | `ONRAMP_LIVE` / `OFFRAMP_LIVE` off — no provider contracted. |
| Tokenized stocks / brokerage | `TOKENIZED_STOCKS_LIVE`, `BROKERAGE_LIVE`, `CROSS_CHAIN_STOCKS_LIVE` off. |
| On-chain settlement or balances as live | `ONCHAIN_SETTLEMENT_LIVE`, `ONCHAIN_BALANCES_LIVE` off. |
| Hardware wallet support | `HARDWARE_WALLET_LIVE` off. |
| DeFi collateral | `DEFI_COLLATERAL_LIVE` off — no risk engine. |
| Any FURL price, market cap, FDV or TVL | No pool, no route (`TOKEN_NOT_TRADABLE`), no market price exists. |
| Cross-chain FURL / multi-chain availability | No bridge deployed. FURL exists only on Solana. |
| Bridge volume, TPV, or user numbers | Circulating supply is 0 and there is one holder. |
| A third-party security audit | None exists. |

---

## 4. Corrections to the emails sent 2026-09-13

Send these proactively. Suggested subject: *"Correction to our application —
FurlPay"*.

### 4.1 Material — correct these

**a) Mint authority.** Emails 1, 2, 4 and 8 say "1B fixed supply, freeze
authority permanently revoked" and never mention the mint authority. A reader
reasonably concludes the supply is immutable. It is not: **mint authority is
ACTIVE** at `FgeSXvrWcqE93byDoF3Vhwou7XTUhY42cYhs7vXkJPUL`. The revocation
script is written and deliberately unrun, pending the treasury migration.

This is the single most important correction, because it is the fact a technical
reviewer will check first and the one whose omission looks worst.

**b) Visa card settlement.** Emails 1 and 3 list "Visa card settlement rails"
among what is built. No issuer is contracted and the feature flag is off with an
`unimplemented` reason. Correct to: "card issuing is designed and flag-gated; no
issuer is contracted yet."

**c) On-device SPL transfers.** Email 4 lists "On-device SPL token transfer
construction" under what was built. True as written, but in context it implies
the app can send. It cannot — message serialization is unimplemented and the
code throws rather than returning a fake signature. Worth stating plainly; the
deliberate refusal to stub it is a point in your favour.

**d) Grant tiers and amounts.** The tier table ("Community/Tooling $5–15K,
Standard Builder $15–50K, Strategic Infrastructure $50–250K") does not match
Wormhole's published xGrant structure, which uses **Contribution / Incubation /
Moonshot** tiers, advertises decisions in **48–72 hours**, and describes entry
grants around **$1–$10,000**. A $50–75K ask against the wrong tier names signals
the application was not read from source.

**e) Application channel.** Wormhole's published route is an **application
form** (`wormhole.com/wormhole-xgrant-application`), not `grants@wormhole.foundation`.
Emails to that address may not enter the pipeline at all. Submit the form.

### 4.2 Minor

| Said | Actual |
| --- | --- |
| "i18n (9 locales)" | 10 locales |
| "5,200+ tests" | 5,231 — accurate, keep it |
| "69 screens" | 69 route files |
| Outlier Ventures Base Camp | Wormhole Foundation runs its own Base Camp accelerator |

### 4.3 Unverified — do not repeat until checked

The contact directory (individual names, emails, LinkedIn URLs) was not
verifiable from here; Wormhole's site blocks automated fetches. Confirm any
individual's role and contact details from their own profile before addressing
them by name. Addressing a named person in the wrong role is a small error that
reads as carelessness.

---

## 5. What is genuinely strong here

Worth saying, because the corrections above are not the whole picture. These are
unusual and defensible:

1. **Every derivation is verified against an outside oracle**, not against
   itself — SLIP-0010 against the official spec vectors, Solana addresses and
   ATAs against `@solana/spl-token` (6/6 matched), and the deployer's derived ATA
   equals the account holding 100% of supply on mainnet.
2. **The supply invariant was written before the bridge exists**, so it cannot
   later be shaped to match whatever the bridge does.
3. **`MAX_SUPPLY` on the spoke contract** bounds a compromised bridge manager,
   and `burnFrom` is allowance-checked rather than role-based — declining a
   permanent-delegate power that FURL's Solana mint also omits.
4. **Honesty controls are CI tests, not policy.** The build fails if circulating
   supply is hard-coded, if the tokenomics drift, if a second mint address
   appears, or if custody claims reappear.
5. **The 100% single-key concentration is published on `/token`**, not hidden.
6. **`/bridge` says no bridge exists** rather than showing a disabled widget.

A funder who checks will find the disclosures already there. That is the
strongest possible position — provided the emails match it.

---

## 6. Rule going forward

Before any external claim about FurlPay capability:

1. Find it in §1. If it is not there, it does not ship.
2. If it is in §2, include the qualifier.
3. If it is in §3, do not say it in any form.
4. If it is new, verify it and add a row here first.

Cheaper than a correction, and far cheaper than being corrected.
