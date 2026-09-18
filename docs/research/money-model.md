# Money model

**Date:** 4 September 2026
**Status:** proposed — no code changed yet
**Companion:** [`tangem-redotpay-furlpay-forensic-audit.md`](./tangem-redotpay-furlpay-forensic-audit.md)

The canonical representation of money and of asset identity in FurlPay, and the audit that motivated it.

---

## 1. Current state — five representations

Every one of these is live in the tracked tree today.

| Location | Type | Unit | Evidence |
|---|---|---|---|
| `packages/payment-intent` | `string` | integer atomic | `isAtomicAmount()` rejects anything but `/^(0\|[1-9]\d*)$/`; `sumAtomic` uses `BigInt` |
| `packages/agent-trust` | `number` (`Micros`) | micros | `toMicros(amount: string)` |
| `packages/ledger` | `number` | integer minor units | `Posting.amount`, validated by `isPositiveInt` |
| `apps/web/lib/agentPolicy.ts` | `number` | cents | `Math.round(amountUsd * 100)` |
| `packages/settlement` | `number` | whole USD | `tierFor(amountUsd: number)` |

### Is anything actually float-corrupt today?

Mostly no, and the distinction matters — this is a **conversion** problem, not a rounding-error problem:

- `ledger` enforces `Number.isInteger`, so a fractional posting throws rather than drifting.
- `payment-intent` never leaves integer arithmetic; `sumAtomic` reduces through `BigInt`.
- `agentPolicy` rounds once at the boundary.

**But `agentPolicy` does perform float arithmetic on the way in:**

```ts
const cents = Math.round(amountUsd * 100);
const budgetCents = Math.round(record.policy.dailyBudgetUsd * 100);
```

`amountUsd` arrives as a JSON `number`. `0.1 + 0.2 !== 0.3` is upstream of this line, not in it — by the time the multiplication happens the error may already be present, and `Math.round` will faithfully preserve it. The defect is not that this line is wrong; it is that **a float ever represented the amount at all.**

### The real cost: 2^53

`number` holds exact integers only to 9,007,199,254,740,991. At 6 decimals that is **9.007 billion USDC**. A ledger that silently stops being exact past a threshold is not a ledger, and the threshold is inside the range a treasury moves in a year.

---

## 2. What Tangem does

Traced from `reference/tangem-app-android`.

**On the wire and on-chain: integer atomic units.**

```kotlin
// libs/visa/src/main/kotlin/com/tangem/lib/visa/utils/BigIntegerExt.kt
internal fun BigInteger.toBigDecimal(decimals: Int): BigDecimal {
    return this.toBigDecimal().movePointLeft(decimals)
}
```

`VisaContractInfo` reads `BigInteger` from the contract and converts to `BigDecimal` **for presentation only**, using the asset's own `decimals`.

**In the domain: `BigDecimal`, never `Double`.**

```kotlin
data class Amount(
    val currencySymbol: String,
    val value: BigDecimal? = null,   // nullable — "unknown" is a real state
    val decimals: Int,               // carried WITH the value
    val type: AmountType,
)
```

Two properties worth stealing regardless of the type chosen:

1. **An amount never travels without its `decimals`.** There is no global constant to consult and get wrong.
2. **`value` is nullable.** A balance that has not been fetched is `null`, not `0`. Zero and unknown are different facts, and conflating them is how a UI tells someone they have no money when it simply has not asked yet.

---

## 3. Decision

> **Integer atomic units, encoded as decimal strings, at every persistence and API boundary. `BigInt` for arithmetic. A formatter for display. Never `number`, never floating point, for any monetary decision.**

### Why strings and not `bigint`

`bigint` is the natural arithmetic type and **it is not a serialisable one**:

```js
JSON.stringify({ amount: 1000n })   // TypeError: Do not know how to serialize a BigInt
```

Money in FurlPay crosses JSON on every hop — HTTP bodies, KV values, CSV exports, audit records. A type that throws on `JSON.stringify` cannot be the boundary type. `payment-intent` already settled on decimal strings, and it is the module the whole reconciliation chain joins against; disagreeing with it would corrupt the join.

So: **strings at rest and in transit, `BigInt` in expressions, converted at the edges.**

### Why not `BigDecimal`-style fixed point

TypeScript has no stdlib `BigDecimal`. A library would add a dependency to the money path — the one place a supply-chain risk is least acceptable. Integer atomic units make the decimal point a *presentation* concern, which removes the need for the type entirely.

### Why not `number` with a documented cap

Because the cap is invisible at the call site. Nothing in `Posting.amount = 9007199254740993` looks wrong, and nothing throws.

---

## 4. Canonical types

```ts
/**
 * An amount of a specific asset.
 *
 * `atomic` is a decimal string of integer base units — "1000000" is one USDC
 * at 6 decimals. The asset travels WITH the amount because an atomic value
 * without its decimals is not a quantity of money, it is an integer.
 */
interface Money {
  readonly atomic: string;
  readonly asset: AssetId;
}

/**
 * Canonical asset identity.
 *
 * Equality is (chain, network, address). NEVER symbol: "USDC" names at least
 * seven different tokens across the networks this repo already references, and
 * two of those are testnet forgeries of the others.
 */
interface AssetId {
  readonly chain: string;      // "solana" | "base" | "arbitrum" | "polygon"
  readonly network: string;    // "mainnet-beta" | "devnet" | "sepolia"
  readonly address: string;    // mint or contract address
  readonly decimals: number;   // per asset — there is no global default
  readonly symbol: string;     // DISPLAY ONLY. Never compared, never keyed on.
}
```

### Rules

1. **Construction validates.** An `AssetId` whose `address` is not registered for its `(chain, network)` throws at construction, as Tangem's `CryptoCurrency.init { checkProperties() }` does.
2. **Equality never uses `symbol`.** Comparing symbols is what makes mainnet and devnet USDC interchangeable.
3. **Arithmetic requires identical `AssetId`.** `add(a, b)` throws when the assets differ. No implicit conversion, ever.
4. **Unknown is `null`, not `"0"`.** A settled amount that has not been observed is `null`.
5. **Display is a separate function.** `format(money)` is the only place a decimal point appears.

---

## 5. The defect this closes

From the forensic audit, `packages/gateway/src/protocol.ts`:

```ts
asset: string;                                    // ×3 declarations
export const USDC_DECIMALS = 6;                   // global
export const NETWORK_USDC: Record<string,string> = {
  arbitrum: "0xaf88…", "arbitrum-sepolia": "0x75fa…",
  base: "0x8335…",     "base-sepolia": "0x036C…",
  polygon: "0x3c49…",  solana: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
};
```

Three consequences:

1. **`solana-devnet` is missing** while both EVM testnets are present — so `examples/agent-paid-api/src/config.ts` hardcodes `DEVNET_USDC_MINT = "4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU"` separately. Two sources of truth for one fact.
2. **`{ asset: <mainnet mint>, network: "solana-devnet" }` typechecks.** Nothing binds them.
3. **`USDC_DECIMALS = 6` is global** — the system cannot correctly represent an 18-decimal asset.

Under the model above, all three become unrepresentable rather than merely discouraged.

---

## 6. Migration

Ordered so that no step depends on a later one.

| Step | Change | Risk |
|---|---|---|
| 1 | Add `AssetId` + registry. Nothing consumes it yet. | none — additive |
| 2 | `NETWORK_USDC` + `USDC_DECIMALS` + `DEVNET_USDC_MINT` → the registry; keep the old exports as deprecated shims that resolve through it | low — one source of truth, old call sites still compile |
| 3 | `Money` in new code only (`UsageEvent`, `SettlementEvidence`, reconciliation) | none — new surface |
| 4 | `agentPolicy` accepts atomic strings; the USD entry point becomes a thin converter at the HTTP edge | **medium** — 7 concurrency tests must stay green |
| 5 | `ledger` `Posting.amount` → string | **medium** — 6 tests construct numbers |
| 6 | `agent-trust` `Micros` → atomic string | **medium** — 22 tests |
| 7 | Delete the deprecated shims | low |

Steps 4–6 each carry a **required regression**: the existing concurrency proof (10 × \$20 against \$50 → exactly 2 reservations) must hold unchanged before and after.

---

## 7. Tests this model demands

Each must fail if the corresponding rule is removed.

1. Constructing an `AssetId` with a mainnet address on a devnet network **throws**.
2. `add()` across two different `AssetId`s **throws**.
3. Two assets sharing `symbol` but differing in `address` are **not equal**.
4. An amount exceeding 2^53 atomic units round-trips exactly through JSON and back.
5. `"0"` and `null` are distinguishable in every settled-amount field.
6. No monetary code path performs `*`, `/` or `+` on a `number`. (Enforceable as a lint rule over the money modules.)
7. `format()` is the only function in the money path that emits a `.`.
