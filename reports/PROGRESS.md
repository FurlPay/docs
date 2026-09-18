# FurlPay — Engineering Progress

**Updated:** 6 September 2026 · **Branch:** `extension-redesign` · **Base:** `63f76e5` (uncommitted work on top)

**Principle:** *We don't compete by pretending we're bigger. We compete by being more correct.*

**Invariant:** *Every rupee, dollar and token entering FurlPay must be accounted for, traceable, reconciled and explainable.*

---

## Completed

### The FurlPay Money Standard — enforced in code, not documentation

> FurlPay considers money settled only when independently verifiable settlement evidence exists.

| Mechanism | Where | What it refuses |
|---|---|---|
| `REQUIRED_EVIDENCE` + `assertTransition` | `lib/money/lifecycle.ts` | A transition into `SETTLED`/`COMPLETED`/`CRYPTO_PENDING`/`BANK_PENDING` without the reference that proves it. An HTTP 200 cannot become `COMPLETED`. |
| Off-ramp table | same | `CRYPTO_CONFIRMED → COMPLETED`. The customer asked for fiat; the chain leg only moved crypto to a provider. |
| On-ramp table | same | `PAYMENT_CONFIRMED → CRYPTO_PENDING`. Confirmation is not settlement, and inbound credits can be recalled. |
| Crypto-deposit table | same | `CRYPTO_CONFIRMED → COMPLETED`. A confirmed deposit from a sanctioned source is confirmed and must not be credited. |
| `ftx_completed_without_evidence` | `0014` + `lib/reconciliation.ts` | Anything that reached `COMPLETED` by writing the state directly. Pages as critical. |

### Financial correctness

- **Exact money** — `lib/money/money.ts`. `bigint` minor units + asset carrying its settlement domain. Refuses to add `USDC.arbitrum` to `USDC.base`, refuses precision it cannot hold, rational fee scaling with an explicit rounding mode, remainder-preserving `allocate`.
- **Chart of accounts** — `lib/ledger.ts`. 21 accounts: customer fiat/stablecoin liability, escrow, InCA/OCA (cross-border segregation per the PA Directions 2025), treasury, provider receivable, in-transit fiat/chain, gas float, operational cash, merchant payable, refund liability, chargeback liability, TDS, GST, fee **revenue**, gas and operational expense.
- **Durable-or-refuse ledger** — production with no Postgres throws `LedgerDurabilityError` rather than posting to a per-lambda map.
- **Transaction envelope** — migration `0014`. Immutable id, idempotency key, state, provider/bank/chain/ledger refs, compliance decision, risk score, reconciliation status, actor, timestamps. Append-only event history by trigger. `ftx_transition()` is row-locked.

### Provider independence

- Canonical ports for on-ramp, off-ramp, banking — `lib/providers/types.ts`. No `if (provider === …)` in application code.
- Capability matrix gated on `contractStatus` + `vdaMerchantAccepted` + evidence URL — `lib/providers/capability.ts`. **Empty**, because zero corridors are contracted.
- Registry, health, explainable selection, failover policy — `lib/providers/router.ts`. Unprobed providers are `unknown`, not healthy. Operator pause survives a health probe.
- **Failover cannot double-pay** — `assertFailoverAllowed` refuses on an indeterminate outcome and on any state where value is committed.

### Asset independence *(new — driven by the 2026 stablecoin landscape)*

- `lib/assets/stablecoins.ts`. Issuer, reserve model, freeze/pause capability, canonical vs bridged, per-address issuer-published source + verification date.
- **Par is never the fallback.** An absent or stale peg quote is `unknown` and refuses settlement. Replaces `swap.ts`'s hardcoded `USDC: 1.0`.
- Bridged representations refused for settlement — a claim on a bridge is not a claim on the issuer.
- Issuer-level concentration reporting: same issuer across chains is one counterparty.

### Treasury

- `lib/treasury/register.ts`. Wallet id → chain → address → purpose → environment → control policy → owner → expected assets → automated-movement cap.
- **Customer/corporate segregation is structural** — `holdsFor()` is exhaustive over purpose, and `canMove` refuses a cross-boundary movement. Gas is corporate.
- `coverage()` counts only declared customer-backing wallets; an undeclared balance is a finding, never coverage.
- **Empty**, because no wallet is provisioned under a documented control policy.

### Custody and signing

- Real `KmsProvider` (enclave signing, no key in-process) + viem account adapter — `lib/kms.ts`, `lib/kmsAccount.ts`.
- Settler resolves KMS first, **refuses a raw env key in production**, gated on `ONCHAIN_SETTLEMENT_LIVE` + kill switch.
- Unimplemented backends refused **by name** rather than falling back to the env signer.

### Trust Layer *(new — fraud, identity and risk)*

`lib/trust/` replaces `packages/security/risk.ts`, which added fixed points per boolean signal and clamped at 100. See `docs/TRUST_LAYER.md`.

- **Reason codes** — 31 codes, 9 categories, stable identities. Analyst explanation and customer message authored **separately**; the customer one is `null` wherever it would disclose a threshold. Only 2 codes may decline alone, both compliance, enforced by test.
- **Trust score** — 0–1000, noisy-OR within category, weighted sum across. Decay per category (behaviour 3 days, compliance 10 years). An unparseable timestamp decays fully rather than counting as fresh.
- **The confidence guard** — a high score on thin evidence routes to a human, never to a decline. False positives otherwise concentrate on the customers we know least about, who cannot argue because nobody looked.
- **Entity graph** — per-type sharing thresholds. IP and ASN can never produce a reason at any count; 400 accounts on one IP yields zero. Ring density calibrated against fixtures (normal 0.5, ring 2.4, threshold 2.0). `auto_declined` does **not** propagate as fraud evidence — only human-confirmed does.
- **Adaptive verification** — prefers methods collecting no new personal data. Document+liveness only when identity itself is in doubt, and always together.

**Two calibration defects the tests caught:** a detected card-testing run scored 204 and auto-approved (fixed with a critical floor); a 12-account fraud ring scored 212 because no single category could reach medium (fixed by recalibrating weights).

### Safety fixes shipped this cycle

Fabricated bank details removed (`/api/banking` returns 503 in every environment) · three impersonating provider webhooks deleted · outbound/inbound webhook secrets separated · both parties sanctions-screened on transfer · `safeAddress` removed from every receive surface · replaced/dropped transactions handled, abandoned settlements terminal · webhook ordering + KYC-demotion guard · reconciliation alerting · deploy gated on a green CI run for the exact SHA.

---

## Current state

| | |
|---|---|
| Tests | **3,289 passing**, 18 skipped, **0 failing** |
| Typecheck | `tsc --noEmit` exit 0 |
| Lint | 0 warnings, 0 errors |
| Build | `next build` exit 0 |
| Route-security invariants | pass (302 routes classified) |
| Migrations written | 14 · **applied: 0006 only** |

---

## Blocked — engineering (no external dependency)

Ordered by financial risk.

1. **Trust Layer has no data.** The graph analysis, score and verification logic are built and tested; nothing populates the graph, nothing persists an assessment, and `/api/wallets/transfer` still calls the old 4-signal `assessRisk`. Highest-value next step.
2. **Chain adapter abstraction.** EVM logic is spread across `x402Settler`, `chainRegistry`, `onchain/balances`. No single port for balance / gas / nonce / broadcast / confirm / reorg.
2. **Chain ⇄ ledger reconciliation.** Needs an indexer. `lib/reconciliation.ts` names this gap explicitly.
3. **Reorg-aware deposit crediting.** Settlement has a confirmation-depth gate; deposits do not.
4. **Transaction simulation before signing.** `/api/mpc/sign` accepts an opaque digest with advisory context.
5. **Nonce and gas management** for the fee-payer.
6. **Account freeze / hold state.** No `frozen` on an account; no LEA response path.
7. **Solana.** Absent from `CHAIN_REGISTRY`. Safe is EVM-only and cannot cover it.
8. **Testnet USDC addresses** (Base Sepolia, Ethereum Sepolia, Solana devnet) missing from the registry.
9. **DR:** no backup/PITR documentation, no RTO/RPO, no restore drill.
10. **Public-testnet x402 run.** The 2026-07-10 evidence is a local hardhat fork.

## Blocked — external

Legal classification opinion *(do first — it determines everything below)* · FIU-IND registration · payment-aggregator contract with VDA-merchant approval · bank-sponsored payout partner · escrow at a scheduled commercial bank · sponsor bank's written VDA acceptance · blockchain analytics · Travel Rule vendor · card issuer + BIN sponsor · smart-contract audit · penetration test · AML officer and compliance operations · insurance.

---

## Decisions

| Decision | Rationale |
|---|---|
| Capability matrix, treasury register and India adapter ship **empty** | A populated registry would make selection return a provider that cannot be paid through, and reconciliation sum balances nobody controls. Returning nothing is the honest state; the caller stops. |
| Peg price at 6 decimals, not the peg currency's 2 | USD cents cannot express `0.9985` — exactly the band where drift lives. Caught by a test. |
| Fees are **revenue**, not expense | The old chart booked them as an expense and its own comment conceded the contradiction. It made the P&L wrong by twice the fee. |
| Turnkey labelled **custodial** | It returns a complete signature; the device holds no share. "2-of-2 MPC" overstated it, and custody classification changes regulatory scope. |
| No new money-moving endpoint | Building `/api/onramp/order` against no provider is a route that can only fail. |
| The asset is a parameter | The 2026 stablecoin landscape is contested. Hardcoding one token bets the company on which one wins. |

## Risks

| Risk | Mitigation |
|---|---|
| Migrations unapplied → production money routes 503 | Correct fail-closed behaviour. Apply `0009`/`0014` with evidence before any beta. |
| Sanctions screening is OFAC crypto-addresses only, from a community mirror | Supply-chain dependency on the control itself. Needs an authoritative feed + UNSC + PEP. |
| Turnkey compromise = total loss of those wallets | Documented and labelled. Mitigate with multisig control policy in the register. |
| Provider concentration | Capability matrix supports multiple rows per corridor; policy not yet set. |
| Single-issuer stablecoin concentration | `assetConcentrationBps` reports it; ceiling is a treasury decision, deliberately not hardcoded. |
