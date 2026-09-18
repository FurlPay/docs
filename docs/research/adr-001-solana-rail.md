# ADR-001 — Solana's role in FurlPay settlement

**Status:** PROPOSED — needs a product/architecture sign-off before Phase 7 begins
**Date:** 4 September 2026
**Decides:** whether Solana becomes a second settlement rail, replaces the current one, or serves as a transfer rail only

---

## Why this ADR exists

The repository contains **no authoritative statement** of Solana's role. Verified by search across `docs/` and `AGENTS.md`: nothing.

Meanwhile Solana capability has been accumulating — `lib/actions/solana.ts` builds real SOL/SPL transfer payloads, `packages/solana-rpc` has 31 tests against live devnet, `packages/settlement` models Solana finality including the Alpenglow upgrade. Building the operational rail on top of that without stating the strategy would be deciding it silently, which is what this document exists to prevent.

---

## Options

| | Option | Consequence |
|---|---|---|
| **A** | Solana becomes a **second settlement rail** | Both rails supported. Routing becomes a real decision. |
| **B** | Solana **replaces** the current rail | Migration. Existing EVM settlement is removed. |
| **C** | Solana is a **transfer rail only**; x402 remains settlement | Hybrid. Requires a transfer/settlement distinction the code does not currently make. |

---

## What the architecture already assumes

Every abstraction built or touched in the last several commits is multi-rail **by construction**, not by accommodation:

| Component | Evidence |
|---|---|
| `AssetNetwork` (`packages/gateway/src/assets.ts`) | Solana, Base, Arbitrum, Polygon are peers in one union. Asset identity is `(chain, network, address)` — the chain is part of identity, not a mode. |
| `packages/settlement` | Models **eight** chains with per-chain finality: solana, ethereum, polygon, bsc, gnosis, arbitrum, base, robinhood. |
| `SettlementPolicy` (`a0c6754`) | `forPayment` takes a network and returns a per-chain verdict. A single-rail system would not need the parameter. |
| `lib/payouts/rails.ts` | `EVM \| SOLANA \| BITCOIN_ONCHAIN \| BITCOIN_LIGHTNING` — rail is already the discriminator. |
| `lib/checkout/session.ts` (`3227ea3`) | Lifecycle is rail-agnostic; nothing in it names a chain. |

**Option B would mean deleting working capability.** Option C would mean introducing a transfer/settlement split that no current module expresses — `planSettlement` already answers "is this settled" per chain, with no notion of a chain that transfers but does not settle.

### One correction to the brief's premise

The brief states "current settlement is x402 / Arbitrum". The gateway's actual default is **Base**:

```ts
this.network = opts.network ?? "base";   // gateway.ts:281
```

Arbitrum is supported and is the documented default for the *hosted* deployment, but the shipped library defaults elsewhere. Worth reconciling — it does not change the recommendation, and it slightly strengthens it: the codebase is already not single-rail.

---

## Recommendation

**Option A — Solana as a second settlement rail.**

Not because it is the safe middle answer, but because it is the only option the existing code does not have to be argued out of. A, B and C are equally implementable from a blank sheet; from *this* tree, A is already ~80% built and B and C both require removing or contradicting something that works.

### What A commits FurlPay to

1. **Routing becomes a real decision.** Two settlement rails means something must choose per payment. That choice needs an owner — cost, speed, asset availability, merchant preference — and it does not exist yet.
2. **Reconciliation must be cross-rail.** Already the stated product boundary (the agent spend ledger), so this is aligned rather than additional.
3. **Finality is genuinely per-chain.** Already true in `packages/settlement`, and now enforced on the running path — the Arbitrum-unsatisfiable case is tested.
4. **Escrow becomes two implementations.** `FurlPayEscrow.sol` exists; an Anchor port must be behaviourally identical, which is Phase 8's cross-implementation test vectors.

### What it explicitly does not commit to

- Removing or deprecating x402/EVM settlement.
- Making Solana the default. The default stays where it is until measurement says otherwise.
- A native bridge. Out of scope, and previously agreed as a company-ending security surface.

---

## Consequences if this is signed off

Phase 7 becomes: build the **operational** layer around the existing transaction builder — RPC failover, priority fees, blockhash lifecycle, a retry engine that distinguishes transient from deterministic failure, confirmation tracking, Solana Pay URL generation and reference tracking.

`lib/actions/solana.ts` is **not** rewritten. `packages/solana-rpc` already holds provider and confirmation primitives (31 tests, verified against live devnet) and is extended rather than replaced.

Phase 8 (Anchor escrow) unblocks only after this, and starts by extracting the Solidity behavioural specification rather than designing fresh.

---

## Open question this ADR does not answer

**Which rail settles a given payment, and who decides?** Option A creates that question and does not resolve it. It should be a follow-up ADR once there is real volume to reason about — deciding routing policy before a single payment has settled on either rail would be guessing.

---

## Sign-off

- [ ] Product owner
- [ ] Architecture owner

Until both are checked, Phase 7 stays blocked and no Solana settlement code should be merged.
