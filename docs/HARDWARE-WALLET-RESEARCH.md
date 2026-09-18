# Hardware wallet integration — engineering research

**Date:** 2026-08-01 · **Target:** `native-app` (Expo SDK 53, RN 0.79.6, New Architecture enabled)
**Chains:** Ethereum (1), Arbitrum (42161), Base (8453), Polygon (137), Robinhood Chain (4663)

---

## How to read this report

Claims are tagged by how they were established:

| Tag | Meaning |
|---|---|
| **[VERIFIED]** | Checked directly this session — npm registry, live API call, or vendor docs fetched |
| **[SOURCED]** | From vendor documentation, not independently reproduced |
| **[ESTIMATE]** | Directional only. Treat as unreliable — see §1 |

The market-share section is the weakest part of any report like this, including this one. Hardware
wallet vendors are private and do not publish unit sales; every "45–50% market share" figure in
circulation traces back to vendor press releases or analyst estimates that cannot be audited. **Do
not make a roadmap decision on a percentage.** The SDK facts in §2 are checkable and should carry
the weight instead.

---

## 0. Findings that change the plan

Four things surfaced during verification that are not in the circulating research and that bear
directly on this codebase.

### 0.1 Ledger's Ethereum signer pulls in ethers 6 — [VERIFIED]

```
npm view @ledgerhq/device-signer-kit-ethereum@1.16.0 dependencies
→ { "ethers": "6.14.1", "xstate": "5.19.2", "inversify": "7.5.1",
    "purify-ts": "2.1.0", "reflect-metadata": "0.2.2", ... }
```

This app deliberately has no ethers and no viem — `lib/evm/` implements RLP and EIP-1559 on
`@noble/*`, verified against Yellow Paper vectors and a hand-derived spec envelope. **Adding the
Ledger signer kit puts ethers 6.14.1 in the bundle transitively**, along with `reflect-metadata`
and `inversify` (a decorator-based DI container, which needs `experimentalDecorators` and a
Babel plugin under Metro).

This is not a blocker, but it is a real cost that no version of the circulating report mentions,
and it should be a deliberate decision rather than a surprise discovered at bundle time. If the
"no ethers" constraint is load-bearing, the alternative is talking to the device over
`@ledgerhq/device-management-kit` + raw APDUs and skipping the Ethereum DSK — more work, and it
forfeits the ERC-7730 clear-signing plumbing the DSK provides.

### 0.2 The DSK accepts a serialized transaction — our serializer feeds it directly — [VERIFIED]

`signerEth.signTransaction(derivationPath, transaction, options)` takes `transaction` as a
**`Uint8Array`** — a serialized RLP transaction, not a structured object.

`lib/evm/transaction.ts` already produces exactly that. The Ledger plugin does not need its own
serializer; it needs the bytes we already build and test. That is a genuinely favourable surprise
and it means the signing-correctness work already done is reusable rather than duplicated.

### 0.3 DMK is observable-based, not promise-based — [VERIFIED]

Every DSK method returns `{ observable, cancel }`, and the DMK core depends on `xstate` 5. The code
sample circulating in the research report is wrong:

```ts
// ✗ From the circulating report — does not work
const signature = await ethSigner.signTransaction("44'/60'/0'/0/0", unsignedTxRlpHex);

// ✓ Actual shape: an observable of device-interaction states, plus a cancel handle
const { observable, cancel } = signerEth.signTransaction(path, txBytes, options);
```

That difference matters beyond syntax. The observable emits intermediate device states — *user must
confirm on device*, *wrong app open*, *device locked* — which is what makes a decent signing UI
possible. A promise-shaped wrapper would throw that away. Our `AccountSigner.signTransaction`
returns a promise, so the Ledger plugin must bridge observable → promise while forwarding those
states to `onProgress`. Worth designing for, not discovering.

### 0.4 Robinhood Chain will blind-sign unless we publish descriptors — [VERIFIED / SOURCED]

Robinhood Chain is an **Arbitrum Orbit L2, chainId 4663, ETH for gas, mainnet since 1 July 2026**.

Two consequences, and they point in opposite directions:

- **Signing works.** secp256k1 over an EIP-1559 digest is chain-agnostic; a Ledger will sign for
  4663 the same as for 42161. `chainId` is a plain `number` in the DSK options with no documented
  allowlist. No firmware update needed.
- **Clear signing does not.** ERC-7730 binds descriptors to specific chainIds via the deployments
  array. A contract on 4663 with no registry entry **falls back to blind signing** — the device
  shows a raw hash and the security story for that chain is materially weaker than for the other
  five.

So: of our five chains, four get clear signing from existing registry coverage. Robinhood Chain
gets it only if **we publish ERC-7730 descriptors for chainId 4663 ourselves**. That is a concrete,
ownable task and it is the single highest-leverage security item in this whole report — it is also
completely absent from the circulating version.

---

## 1. Market — and why it should not drive the decision

**[ESTIMATE — low confidence]** The commonly cited 2026 split is Ledger ~45–50%, Trezor ~25–30%,
Tangem ~10–12%, with SafePal/OneKey/Keystone/BitBox02/Coldcard sharing the remainder. Tangem is
the fastest grower on revenue.

I could not verify any of it. These vendors are private; the figures trace to press releases. What
*is* checkable, and what actually predicts integration cost, is SDK health — §2.

One directional signal that is defensible: Ledger is the only vendor whose SDK appears in the
dependency trees of multiple large third-party mobile wallets. That is evidence of a supported
integration path, which is the thing we care about.

---

## 2. SDK reality check — [VERIFIED against the npm registry, 2026-08-01]

| Package | Version | Last publish | Read |
|---|---|---|---|
| `@ledgerhq/device-management-kit` | **1.7.1** | 2026-08-01 | Healthy, shipping today |
| `@ledgerhq/device-transport-kit-react-native-ble` | **1.3.2** | 2026-08-01 | Healthy |
| `@ledgerhq/device-signer-kit-ethereum` | **1.16.0** | 2026-08-01 | Healthy |
| `@trezor/connect-mobile` | **9.7.3** | 2026-07-30 | Healthy |
| `@onekeyfe/hd-ble-sdk` | **1.1.32** | 2026-08-01 | Healthy |
| `@onekeyfe/hd-transport-react-native` | **1.1.32** | 2026-08-01 | Healthy |
| `@reown/appkit-react-native` | **2.0.6** | 2026-07-14 | Healthy |
| `@keystonehq/keystone-sdk` | **0.12.3** | 2026-06-05 | Maintained, pre-1.0 |
| `gridplus-sdk` | **4.0.0** | 2026-06-10 | Maintained but **node-only** — not RN viable |
| `tangem-sdk-react-native` | **3.1.0** | 2025-02-18 | **~18 months stale** |
| `bitbox02-api` | **0.15.1** | **2023-08-28** | **~3 years stale** |

**Two corrections to the circulating report.** It lists BitBox02 as "🟢 Active" — the npm package
has not been published since August 2023. And it contradicts itself on Tangem (§6.4 says the RN SDK
is deprecated; a later revision says it "received community updates"). The registry settles it:
last publish February 2025. Not dead, but not current either — a custom TurboModule over the
native Swift/Kotlin SDKs is the honest path if Tangem matters.

**Every version pin in the circulating report is roughly one major behind** (`^0.6.x` for DMK
against actual 1.7.1; `^1.x` for Reown against 2.0.6). Installing its Appendix A verbatim would
resolve to ancient or nonexistent versions.

### Compatibility with this app — [VERIFIED]

```
@ledgerhq/device-transport-kit-react-native-ble@1.3.2 peerDependencies:
  react-native: ">0.74.1"      → we are on 0.79.6      ✓
  react-native-ble-plx: "3.4.0" → exact pin, needs prebuild
  @ledgerhq/device-management-kit: "^1.0.0"
```

`newArchEnabled: true` is already set in `app.json` and `android/` is already prebuilt, so the dev-build
requirement costs nothing new. Expo Go will not work with any hardware wallet SDK — that part of
the circulating report is correct.

---

## 3. How signing actually works

The mobile phone is untrusted execution space. The device's Secure Element is the root of trust and
the only place a private key exists. Three protocol families:

**APDU (ISO 7816-4)** — Ledger, Tangem. Command/response frames, chunked because SE buffers are
~256 bytes. `CLA=0xE0 INS=0x04 P1=0x00` opens a transaction signature; `P1=0x80` continues it.
Status `0x9000` success, `0x6985` user refused.

**Protobuf** — Trezor. Not APDU. Safe 7 BLE adds Trezor Host Protocol v2, an encrypted framing
layer that must be implemented on top of raw BLE.

**UR/CBOR over animated QR** — Keystone, Coldcard Q, SafePal S1. Multi-frame fountain codes,
camera in both directions. Slowest, and the only genuinely air-gapped option.

### Where our existing code fits

```
lib/evm/transaction.ts   →  serialize (0x02 ‖ RLP[...])     ← already built and spec-verified
        ↓ Uint8Array
AccountSigner.signTransaction(tx)                            ← already abstracted (lib/signing/)
        ↓
  device-key  │  ledger (BLE→APDU)  │  tangem (NFC→hash)  │  keystone (QR→UR)
        ↓
lib/evm/rpc.ts           →  eth_sendRawTransaction          ← already built
```

The abstraction landed last session, so a vendor plugin is now a leaf, not a refactor.

### The Tangem exception, which is a security difference and not a detail

Ledger and Trezor receive the **whole transaction** and decode it on-device, so the SE knows what it
is approving. Tangem receives a **32-byte Keccak hash** and signs it blind — the card physically
cannot know what it signed, and it has no screen to show anyone. The only thing the user ever reads
is the phone: the exact surface a hardware wallet exists to remove from the trust path.

Our `SignerCapabilities` already encodes this (`prehashedOnly`, `trustedDisplay`) and
`requireOwnConfirmation()` derives behaviour from it. A policy of *refuse Tangem above $N* is
implementable today.

---

## 4. Integration approach — native SDK vs WalletConnect

| | Native SDK | WalletConnect / Reown | Air-gapped QR |
|---|---|---|---|
| Transport | Direct BLE/NFC | WebSocket relay → companion app | Camera, both ways |
| Latency | 1–2 s | app-switch + relay | 5–15 s |
| Failure modes | BLE drop, permissions | relay outage, app not installed | glare, focus |
| Control over UX | Full | None — the companion app owns it | Moderate |
| Devices covered | One vendor per plugin | Hundreds | Vendors that support UR |
| Engineering | 3–6 weeks/vendor | ~2–3 weeks total | ~3–4 weeks |

**Recommendation: WalletConnect first, then Ledger native.** This inverts the circulating report's
order, deliberately.

The reasoning is coverage per unit of risk. WalletConnect reaches Ledger *and* Trezor *and* Tangem
*and* SafePal *and* OneKey through their companion apps for one integration, adds no native module,
no ethers, no BLE permissions, and nothing that can only be tested with hardware in hand. It
establishes whether users actually want hardware support before we spend six weeks and a
transitive ethers dependency finding out.

Ledger native then earns its cost on UX for the largest single vendor — no app-switching, direct
BLE, in-app error states. But it is an optimisation of a flow WalletConnect already provides, and
optimisations should follow demand.

---

## 5. Security — what matters most here

Ranked by what would actually bite this app:

1. **Blind signing on Robinhood Chain.** See §0.4. Publish ERC-7730 descriptors for chainId 4663
   covering the LI.FI diamond, or accept that one of five chains signs blind.
2. **Address verification on displayless devices.** Tangem has no screen. Our `trustedDisplay: false`
   → `requireOwnConfirmation()` path is the only defence and must stay enforced in the UI.
3. **BLE pairing.** Require LE Secure Connections with numeric comparison; refuse "Just Works",
   which is trivially MITM-able.
4. **Device attestation on first pair.** Every major vendor exposes a challenge/response against
   their root CA. Skipping it accepts supply-chain-tampered devices silently.
5. **NFC relay.** Tight APDU timeouts (<300 ms) — relay networks add 50–500 ms.
6. **Never transmit the seed.** Nothing in any of these flows ever moves a recovery phrase over the
   wire. Any SDK that appears to want one is a red flag.

Our existing pre-sign validation (`lib/swap/validatePlan.ts`) already covers chain-mismatch,
address shape with EIP-55, calldata presence and quote freshness — those apply unchanged to a
hardware signer, since they run before the signer is ever called.

---

## 6. Recommended roadmap

Effort estimates assume one engineer with the relevant hardware physically present. **None of this
is testable without the devices** — that is not a caveat, it is a staffing requirement.

### Phase 1 — WalletConnect (≈2–3 weeks)

`@reown/appkit-react-native@2.0.6` as an `AccountSigner` plugin. Covers Ledger, Trezor, Tangem,
SafePal, OneKey and hundreds of software wallets through one integration. No native module beyond
what AppKit needs, no ethers, no hardware required to build it.

**Ship first because it answers the demand question cheaply.**

### Phase 2 — Ledger native BLE (≈4–6 weeks) + ERC-7730 descriptors (≈1 week)

DMK 1.7.1 + transport 1.3.2 + Ethereum DSK 1.16.0, bridging observable → promise behind our
existing `AccountSigner`. Accept the transitive ethers 6, or budget more to go raw-APDU.

Publish ERC-7730 descriptors for our LI.FI interactions **including chainId 4663** — this is
cheap, independent of everything else, and improves every Ledger user's security immediately.
It can start now.

### Phase 3 — driven by user data, not by this document

Tangem (custom TurboModule; the RN package is stale) if NFC UX demand shows up. Keystone QR if
air-gap demand shows up. **OneKey is the cheapest remaining native integration** — a genuinely
maintained RN BLE SDK — if its user base appears.

### Not recommended

- **GridPlus** — SDK is node-only. Not viable on React Native. [VERIFIED]
- **BitBox02** — npm package unpublished since 2023. [VERIFIED]
- **Coldcard** — Bitcoin-only firmware, no EVM. Irrelevant unless we add Bitcoin.

---

## 7. The mistake most wallet apps make

They add hardware wallet support as a *wallet type* — a parallel universe of screens, state and
transaction building that drifts from the software path until the two behave differently and only
one is well tested.

The correct shape, which this codebase now has, is hardware as **another signer behind one
interface**. Transaction building, validation, gas, nonce, broadcast, receipt polling, bridge
tracking and telemetry are all identical regardless of what holds the key. A vendor plugin
implements `getAddress` and `signTransaction`, declares its capabilities honestly, and changes
nothing else.

Everything in §6 is a leaf on that interface. That was the point of building it first.

---

## Sources

- [Ledger Ethereum Signer Kit](https://developers.ledger.com/docs/device-interaction/references/signers/eth) — method signatures, `Uint8Array` transaction input
- [Ledger Clear Signing overview](https://developers.ledger.com/docs/clear-signing/overview) and [for wallets](https://developers.ledger.com/docs/clear-signing/for-wallets) — blind-signing fallback conditions
- [ERC-7730 registry](https://github.com/ethereum/clear-signing-erc7730-registry) — descriptor format, chainId binding
- [Robinhood Chain docs](https://docs.robinhood.com/chain/add-network-to-wallet) and [support](https://robinhood.com/us/en/support/articles/robinhood-chain-mainnet/) — Orbit L2, chainId 4663, ETH gas
- [GridPlus SDK](https://github.com/GridPlus/gridplus-sdk) — node-only scope
- npm registry, queried 2026-08-01, for every version and publish date in §2
