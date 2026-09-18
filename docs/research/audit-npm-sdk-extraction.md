# Audit — published package, SDK surfaces, extraction candidates

**Date:** 4 September 2026
**Covers:** Phase 1 (npm), Phase 4 (SDK), Phase 5 (oracle/webhook extraction)
**Nothing was published, extracted or refactored. This is findings only.**

---

## 1. `@furlpay/elements` — a live vulnerability, verified

### The evidence

Not inferred. The tarball npm currently serves as `latest` was downloaded and read:

```bash
npm pack @furlpay/elements@0.1.1 && tar -xzf furlpay-elements-0.1.1.tgz
grep -n "Math.random" package/src/index.tsx
```

```
line 66:  // Fall back to a simulated hash if no backend is reachable (sandbox).
line 69:  "0x" + Math.random().toString(16).slice(2).padEnd(64, "0").slice(0, 64);
line 72:  onSuccess?.({ transactionHash: hash, amount });
```

| | |
|---|---|
| **Vulnerable versions** | `0.1.0`, `0.1.1` — both published |
| **`latest` tag** | `0.1.1` — **the vulnerable one** |
| **Live since** | 3 July 2026 (~2 months) |
| **Downloads** | 34 last month, 8 last week |
| **Fixed in working tree** | Yes — commit `3227ea3` |
| **Fixed on npm** | **No** |

### Why the package ships source directly

`package.json` has `"main": "src/index.tsx"`, no `files` field and no build script. There is no `dist/`. **The source file is the published artifact**, so the fix in `3227ea3` is byte-for-byte what a republish would ship — there is no build step that could diverge from it. Verified: `tsc --noEmit --jsx react` on `src/index.tsx` passes.

### What an integrator experiences today

Install `@furlpay/elements`, render `<FurlpayCheckoutButton>`, customer clicks Pay:

1. 500ms artificial delay imitating a wallet prompt — no wallet opens
2. `fetch` to the backend; **if it fails, the catch is a `||` fallback, not an error**
3. `Math.random()` produces a 64-hex-char string
4. Button renders "Paid"
5. `onSuccess({ transactionHash })` fires
6. Merchant ships goods

It is most confident when it has least evidence. With the backend unreachable it does not even need the server's (also fabricated) hash.

### Release plan — NOT EXECUTED

**Version: `0.2.0`.** Breaking, and correctly so: `onSuccess` is removed and replaced with `onSession`. Under 0.x semver, a minor bump signals breaking. A patch release would be wrong — an integrator who auto-updates within `^0.1.x` would find `onSuccess` silently gone and their success handler never firing, which for a payment component is the right failure but must not arrive unannounced.

Keeping `onSuccess` as an alias was considered and rejected: the callback's entire meaning was "payment succeeded", and it never established that. Preserving the name would preserve the lie.

```
[ ] 1. Bump packages/elements/package.json to 0.2.0
[ ] 2. Add CHANGELOG entry naming the security issue plainly
[ ] 3. npm publish --access public --provenance      (needs npm credentials)
[ ] 4. npm deprecate "@furlpay/elements@0.1.0" "Reports payments that never happened — upgrade to >=0.2.0"
[ ] 5. npm deprecate "@furlpay/elements@0.1.1" "Reports payments that never happened — upgrade to >=0.2.0"
[ ] 6. Verify: npm view @furlpay/elements dist-tags   → latest = 0.2.0
[ ] 7. Verify: npm pack @furlpay/elements@0.2.0 && grep Math.random  → no match
[ ] 8. Consider a GitHub security advisory (GHSA) — this is a payment-integrity defect in a published package
```

**Deprecating is more important than publishing.** A deprecation warning reaches every existing installer on their next `npm install`; a new version only reaches people who upgrade.

> **BLOCKER — npm publish credentials.** Steps 3–5 need an authenticated npm account with publish rights on the `@furlpay` scope. Not automatable here and deliberately not attempted.

---

## 2. SDK surfaces — the finding is an absence

Four SDKs exist and are small:

| SDK | Source files | Exposes checkout? |
|---|---|---|
| `sdks/node` | 6 | **No** |
| `sdks/python` | 3 | **No** |
| `sdks/go` | 2 | **No** |
| `sdks/rust` | 4 | **No** |

**No SDK exposes checkout at all.** Verified by searching every SDK directory for the string `checkout`: zero matches.

### What this means for Phase 4

The brief asks to "identify the canonical public API" from the existing SDKs and then spec Swift and Kotlin from it. That cannot be done for checkout, because no existing SDK covers it. There is no canonical surface to copy — there is a surface to *define*.

This reorders the work:

1. **Define the checkout API surface once**, from the server contract now that it is real (`POST /api/checkout/session`, `GET /api/checkout/session?id=`, plus a webhook).
2. **Add it to one existing SDK first** — Node, which has the most surface already — and let it prove the shape.
3. **Then** port to Python/Go/Rust, and only then spec Swift/Kotlin.

Writing Swift and Kotlin now would mean inventing a surface in two new languages that four existing SDKs do not implement, then having to reconcile six. The execution rule "do not allow mobile SDKs to expose weaker guarantees than the existing SDKs" is currently satisfiable trivially and meaninglessly, since the existing guarantee for checkout is *nothing*.

### The semantics any SDK must carry

Derived from the now-real lifecycle in `lib/checkout/session.ts`:

- `createSession` returns a session whose status is **`requires_payment`**, never a completion
- there is **no `transactionHash` field** until a rail observes one
- polling returns the current status; the terminal success state is `completed`, reachable only through observed confirmation
- `unknown` is a distinct status meaning *outcome not established* and must never be mapped to a language's error/failure type
- amounts are integer atomic units as strings

The last two are where a naive SDK port does damage: mapping `unknown` onto `Err`/`raise`/`error` would recreate exactly the "timeout means failed" defect the server was built to avoid.

---

## 3. Extraction candidates — both rejected, for different reasons

### `lib/oracle` — do NOT extract

| | |
|---|---|
| Size | 293 lines |
| Tests | Yes (`__tests__/oracle.spec.ts`) |
| **Consumers** | **Zero** |

Verified: no file outside `lib/oracle/` imports it.

Extracting this would package dead code. Publishing a module nobody calls does not improve reuse, dependency boundaries, testing, security, versioning, deployment or ownership — it adds a build target and a version number to something that currently has no effect on the running system.

**The real finding is not the packaging question.** `lib/oracle` is the *third* instance this session of the same pattern:

| Module | Tests | Consumers before | |
|---|---|---|---|
| `packages/x402-guard` | 38 | 0 | still 0 |
| `packages/settlement` | 52 | 0 | **wired in `a0c6754`** |
| `lib/oracle` | yes | 0 | still 0 |

FurlPay repeatedly builds correct, tested modules and does not connect them. That is a more valuable thing to fix than any packaging decision. **Recommendation: decide whether oracle pricing is needed on a live path. If yes, wire it; if no, delete it.** Packaging is the wrong question either way.

### `webhookGuard.ts` — do NOT extract yet

| | |
|---|---|
| Size | 56 lines |
| **Consumers** | **9** — every webhook route (bridge, card-auth, marqeta, persona, resend, passport, …) |

This one is genuinely shared and genuinely used — the opposite problem. But all nine consumers live in the same app, so a package boundary would add a build step, a version and a publish decision for 56 lines with no external consumer.

Extraction becomes justified the moment a **second deployable** needs signature verification — a standalone webhook worker, or the commerce integrations if they verify signatures out-of-process. Not before.

**Recommendation: leave it. Revisit when Phase 6 defines where commerce webhooks are verified.**

---

## 4. What this changes about the plan

1. **npm deprecation outranks npm publish.** It reaches everyone; a new version reaches only upgraders.
2. **Phase 4 is blocked on a definition, not an audit.** The checkout SDK surface must be designed and proven in Node before Swift/Kotlin are meaningful.
3. **Phase 5 produces no code.** Both extractions are rejected on evidence. The oracle finding redirects to a wire-or-delete decision.
4. **Phase 6 (commerce) has a hard prerequisite** that is now satisfied: a checkout lifecycle that cannot report unobserved settlement. Building Shopify against the old endpoint would have propagated the fabrication into three storefront platforms.
