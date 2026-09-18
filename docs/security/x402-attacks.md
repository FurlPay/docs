# x402 attack hardening — status & remaining work

Tracks our response to the two x402 security papers against the FurlPay stack.

- **arXiv:2605.30998** — "Free-Riding the Agentic Web" (F1–F5). Basis for `@furlpay/x402-guard`.
- **arXiv:2605.11781** — "Five Attacks on x402", Li et al. (I-A, I-B, II, III, IV). Assessed and fixed 2026-07-09 (commit `86096c2` on `extension-redesign`).

## Shipped (monorepo, committed — NOT yet deployed)

| Attack | Fix | Location |
|---|---|---|
| II — replay/idempotency | Atomic cross-instance nonce+quote claim via Redis `SET NX` | `apps/web/src/lib/x402.ts` (`verifyPayment`, now async), `apps/web/src/lib/x402Facilitator.ts` (`settle`), `lib/kv.ts::kvSetNx` |
| I-B — settlement preemption | `settle()` restricted to an owner-managed endorsed-facilitator allow-list | `packages/contracts/contracts/X402Facilitator.sol` (`onlyFacilitator`, `setFacilitator`, 3rd ctor arg) |
| I-A — revert-grant | `requiredConfirmations(amount)` (Corollary 10) gates settle; settler reports `{transaction, confirmations}`; shallow settlement fails closed, nonce kept burned | `apps/web/src/lib/x402.ts`, `apps/web/src/lib/x402Facilitator.ts` |
| III — cache leakage | `withNoStore`/`noStoreHeaders` on paid responses + 402 quotes | `apps/web/src/lib/x402.ts`, `apps/web/src/app/api/x402/fx/route.ts` |
| IV — discovery Sybil/metadata | `validateMetadata`, per-host registration cap, per-IP register rate limit, per-host-diversified shortlist | `apps/web/src/lib/x402Registry.ts`, `apps/web/src/app/api/x402/register/route.ts`, `.../discovery/resources/route.ts` |

Verified: `apps/web` `tsc` clean; 36 contract tests + 14 x402-guard tests pass.

## Remaining work

- [ ] **Provision Upstash Redis in prod** (`UPSTASH_REDIS_REST_URL/TOKEN`). Until then the Attack II replay claim silently degrades to single-instance — `replayProtectionDurable` (exported from `lib/x402.ts`) is `false`. Same KV the card-auth work needs.
- [ ] **Deploy.** All of the above is committed on `extension-redesign`; production still runs the pre-fix code. Ship on the next `vercel --prod`.
- [ ] **Real settler with confirmation depth.** `settle()` currently uses the mock settler (reports the required depth so dev passes). A live settler must actually wait for `requiredConfirmations(amount)` on-chain before reporting `confirmed`. Wire it when `INTEGRATION_MODE=live`.
- [ ] **Redeploy `X402Facilitator`** with the new 3-arg constructor and endorse the fee-payer EOA (`X402_FACILITATOR` env or `setFacilitator`). Existing deployments have the caller-unbound `settle`.
- [ ] **Per-route `no-store` audit.** Only `/api/x402/fx` is wired through `withNoStore` so far. Any other route that returns paid/gated content (travel/markets/compliance resources, once they enforce x402) must stamp `no-store` too.
- [ ] **Wire `@furlpay/x402-guard` into the deployed facilitator**, or keep the inline logic — right now `apps/web/src/lib/x402Facilitator.ts` reimplements nonce linearization rather than importing the guard. Decide on one source of truth.
- [ ] Fold the confirmation-depth + cache-control + facilitator-binding fixes back **upstream into the public libs** (tracked as issues on `FurlPay/x402-guard` #1–4 and `FurlPay/furlpay-x402` #4–7).

## Public tracking issues (filed 2026-07-09)

- `FurlPay/x402-guard` #1 (I-A), #2 (I-B), #3 (III), #4 (IV scope/doc)
- `FurlPay/furlpay-x402` #4 (III cache), #5 (I-A grant), #6 (II trust/verify), #7 (facilitator client status bug)
- `FurlPay/furlpay-extension` #8 (forged x402 postMessage spoofing), #9 (unvalidated Backend URL phishing), #10 (challengeId encoding)
