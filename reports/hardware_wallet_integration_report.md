# Hardware Wallet Integration — Technical Research Report
## FurlPay React Native (Expo) Self-Custody Wallet
**Date:** August 2026 | **Target Chains:** Ethereum, Base, Arbitrum, Polygon, Robinhood Chain, future EVM

---

# Table of Contents

1. [Hardware Wallet Market](#1-hardware-wallet-market)
2. [Mobile Integration Matrix](#2-mobile-integration-matrix)
3. [Signing Architecture](#3-signing-architecture)
4. [Ledger](#4-ledger)
5. [Trezor](#5-trezor)
6. [Tangem](#6-tangem)
7. [Keystone](#7-keystone)
8. [OneKey](#8-onekey)
9. [WalletConnect as Abstraction Layer](#9-walletconnect-as-abstraction-layer)
10. [Security Best Practices](#10-security-best-practices)
11. [UX Design](#11-ux-design)
12. [Architecture Recommendation](#12-architecture-recommendation)
13. [Best-in-Class Features for FurlPay](#13-best-in-class-features-for-furlpay)
14. [FurlPay Hardware Wallet SDK Design](#14-furlpay-hardware-wallet-sdk-design)
15. [Future-Proofing](#15-future-proofing)
16. [Implementation Roadmap](#16-implementation-roadmap)

---

# 1. Hardware Wallet Market

## 1.1 Global Market Overview

The global hardware wallet market is valued at **$0.54–0.72 billion** in 2025–2026, with a CAGR of 18–25% toward ~$1.2B by the late 2020s. North America accounts for ~40–45% of total revenue, followed by Europe and Asia-Pacific.

## 1.2 Market Share Estimates (2025–2026)

| Vendor | Global Share | U.S. Share | Cumulative Units Sold | 2025 Revenue | Active Users |
|:---|:---:|:---:|:---:|:---:|:---:|
| **Ledger** | ~45–50% | ~50%+ | **8M+** devices | **>$100M** | Millions (160+ countries) |
| **Trezor** | ~25–30% | ~25% | **2M+** devices | **~$47.2M** | 2M+ global |
| **Tangem** | ~10–12% | Growing rapidly | **6M+** cards produced | **$61.3M** (+102% YoY) | 1M+ active |
| **SafePal** | ~5–8% | Moderate | **500K+** HW units (2025) | Private | 25M+ ecosystem |
| **OneKey** | ~3–5% | Growing | Undisclosed | Private ($150M valuation) | Multi-million |
| **Coldcard** | ~2–3% | ~5% (BTC segment) | Undisclosed | Private | BTC maximalist niche |
| **Keystone** | ~2–3% | Moderate | Undisclosed | Private | Intermediate-institutional |
| **BitBox02** | ~1–2% | Small | Undisclosed | Private | European privacy community |

## 1.3 Fastest-Growing Wallets

1. **Tangem** — 102% YoY revenue growth ($61.3M in 2025). NFC card form factor, Tangem Ring, Tangem Pay Visa card. Expanded into **200+ U.S. Best Buy stores** in 2026.
2. **SafePal** — User base from 10M → 25M+ (2024–2026). Budget air-gapped models (S1 Pro, X1) driving adoption.
3. **OneKey** — $150M Series B (mid-2025). 100% open-source hardware/firmware/software.
4. **Ledger** — Growth re-accelerated via E-Ink touchscreen models (Stax, Flex).

## 1.4 SDK Availability Summary

| Vendor | Published SDK | Open Source | Third-Party Integration |
|:---|:---:|:---:|:---|
| **Ledger** | ✅ DMK + DSK + Transport Kits | Apache 2.0 | Full native SDK for mobile/web |
| **Trezor** | ✅ @trezor/connect, connect-mobile | T-RSL / GPL-3.0 | Web-first, mobile via connect-mobile |
| **Tangem** | ✅ iOS (Swift), Android (Kotlin), RN | MIT | Native mobile SDKs + RN bridge |
| **Keystone** | ✅ @keystonehq/keystone-sdk | Open Source | QR-based, React Native components |
| **OneKey** | ✅ @onekeyfe/hd-ble-sdk | Open Source | Full React Native BLE SDK |
| **SafePal** | ❌ No direct HW SDK | N/A | WalletConnect or SafePal App bridge |
| **BitBox02** | ✅ bitbox02-api-js | Open Source | WebUSB/HID, BLE (Nova) |
| **Coldcard** | ✅ ckcc-protocol (Python) | Open Source | NFC/QR, Bitcoin-only |

## 1.5 Mobile-First Practicality Ranking

| Rank | Wallet | Why |
|:---:|:---|:---|
| 1 | **Tangem** | NFC tap-to-sign, no battery, credit-card form factor, sub-second signing |
| 2 | **SafePal** | Air-gapped QR + X1 BLE model, excellent mobile app ecosystem |
| 3 | **OneKey** | BLE on Classic 1S & Pro, full React Native SDK, open-source |
| 4 | **Ledger** | BLE on Nano X/Stax/Flex, mature DMK SDK, NFC on Stax/Flex |
| 5 | **BitBox02 Nova** | New BLE support (June 2025), iOS/Android connectivity |
| 6 | **Keystone** | Air-gapped QR (touchscreen + camera), secure but multi-step |
| 7 | **Coldcard** | NFC on Mk4/Q, QR on Q. Bitcoin-only, no EVM |
| 8 | **Trezor** | USB-C OTG primary; BLE only on Safe 7 (late 2025) |

## 1.6 Connectivity Matrix

| Wallet Model | USB-C | Bluetooth (BLE) | NFC | Air-Gap QR | MicroSD |
|:---|:---:|:---:|:---:|:---:|:---:|
| Ledger Nano S Plus | ✅ | ❌ | ❌ | ❌ | ❌ |
| Ledger Nano X | ✅ | ✅ | ❌ | ❌ | ❌ |
| Ledger Stax / Flex | ✅ | ✅ | ✅ | ❌ | ❌ |
| Trezor Safe 3 / 5 | ✅ | ❌ | ❌ | ❌ | ✅ (backup) |
| **Trezor Safe 7** | ✅ | **✅** | ❌ | ❌ | ❌ |
| Tangem Card / Ring | ❌ | ❌ | ✅ | ❌ | ❌ |
| SafePal S1 / S1 Pro | ✅ (charge) | ❌ | ❌ | ✅ | ❌ |
| SafePal X1 | ✅ | ✅ | ❌ | ❌ | ❌ |
| Keystone 3 Pro | ✅ (charge) | ❌ | ❌ | ✅ | ✅ |
| Coldcard Mk4 | ✅ | ❌ | ✅ | ❌ | ✅ |
| Coldcard Q | ✅ | ❌ | ✅ | ✅ | ✅ |
| BitBox02 (Original) | ✅ | ❌ | ❌ | ❌ | ✅ (backup) |
| **BitBox02 Nova** | ✅ | **✅** | ❌ | ❌ | ✅ (backup) |
| OneKey Classic 1S | ✅ | ✅ | ❌ | ❌ | ❌ |
| OneKey Pro | ✅ | ✅ | ❌ | ✅ | ❌ |

---

# 2. Mobile Integration Matrix

## 2.1 Comprehensive Integration Comparison

| Feature | Ledger | Trezor | Tangem | Keystone | OneKey | SafePal | BitBox02 | Coldcard |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Bluetooth** | ✅ Nano X, Stax, Flex | ✅ Safe 7 only | ❌ | ❌ | ✅ Classic 1S, Pro | ✅ X1 | ✅ Nova | ❌ |
| **NFC** | ✅ Stax, Flex | ❌ | ✅ All models | ❌ | ❌ | ❌ | ❌ | ✅ Mk4, Q |
| **USB** | ✅ All models | ✅ All models | ❌ | ✅ (charge only) | ✅ All models | ✅ (charge) | ✅ All models | ✅ |
| **QR Signing** | ❌ | ❌ | ❌ | ✅ (animated) | ✅ Pro | ✅ S1/S1 Pro | ❌ | ✅ Q |
| **React Native SDK** | ✅ Official DMK | ✅ @trezor/connect-mobile | ✅ tangem-sdk-react-native | ✅ @keystonehq/* | ✅ @onekeyfe/hd-ble-sdk | ❌ | ⚠️ JS SDK (WebUSB) | ❌ |
| **Expo Compatible** | ✅ Dev Builds | ✅ Dev Builds | ✅ Dev Builds | ✅ Dev Builds | ✅ Dev Builds | N/A (WalletConnect) | ⚠️ Android only | N/A |
| **Official SDK** | ✅ DMK + DSK | ✅ @trezor/connect | ✅ iOS + Android + RN | ✅ keystone-sdk | ✅ Full SDK stack | ❌ | ✅ bitbox02-api | ⚠️ Python only |
| **License** | Apache 2.0 | T-RSL / GPL-3.0 | MIT | Open Source | Open Source | N/A | Open Source | Open Source |
| **Maintenance** | 🟢 Active | 🟢 Active | 🟢 Active | 🟢 Active | 🟢 Active | N/A | 🟢 Active | 🟡 Moderate |
| **Production Apps** | MetaMask, Rainbow, Zerion, Phantom, 1inch | Trezor Suite Mobile | Tangem Wallet, Xaman | MetaMask, OKX, Phantom, Rabby, Safe | OneKey App | SafePal App | BitBoxApp | Sparrow, Nunchuk |
| **EVM Support** | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ Full | **❌ Bitcoin only** |

> [!IMPORTANT]
> **Expo Go is NOT supported** by any hardware wallet SDK. All integrations require **Expo Development Builds** (`npx expo prebuild` / `eas build`) due to native BLE/NFC/USB module requirements. Ensure your project uses the **New Architecture (Fabric + TurboModules)**, default since Expo SDK 55+.

---

# 3. Signing Architecture

## 3.1 Transaction Flow Overview

Hardware wallets enforce a strict **Zero-Trust Client Architecture**: the mobile phone is treated as untrusted execution space, while the hardware wallet's Secure Element (SE) is the isolated root of trust.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          UNTRUSTED ZONE                                 │
│  ┌─────────────────────┐     Unsigned Tx     ┌──────────────────────┐  │
│  │  FurlPay Mobile App │ ──────────────────▶  │  Host MCU / BLE /   │  │
│  │  (React Native/Expo)│ ◀──────────────────  │  NFC Controller     │  │
│  └─────────────────────┘     Signed Bytes     └──────────────────────┘  │
│                                                          │              │
└──────────────────────────────────────────────────────────┼──────────────┘
                                                           │ APDU Payload
┌──────────────────────────────────────────────────────────▼──────────────┐
│                           TRUSTED ZONE                                  │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                      SECURE ELEMENT (SE)                          │  │
│  │  ┌───────────────────┐  Decoded Tx  ┌──────────────────────────┐ │  │
│  │  │ EIP-7730 Parser   │ ──────────▶  │ Hardware Trusted Display │ │  │
│  │  └───────────────────┘              └──────────────────────────┘ │  │
│  │                                              │                    │  │
│  │                                     Physical Button Press         │  │
│  │                                              ▼                    │  │
│  │  ┌───────────────────┐  Compute Sig ┌──────────────────────────┐ │  │
│  │  │ Private Key Store │ ──────────▶  │ ECDSA secp256k1 Engine   │ │  │
│  │  └───────────────────┘              └──────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3.2 Complete EVM Transaction Signing Flow

```
FurlPay User          FurlPay App (Host)         BLE/NFC/USB Transport       Hardware SE
    │                        │                          │                        │
    │── 1. Initiate Tx ─────▶│                          │                        │
    │   (swap/send/bridge)   │                          │                        │
    │                        │── 2. Build unsigned tx    │                        │
    │                        │   (nonce, gas, to, value, │                        │
    │                        │    data, chainId)         │                        │
    │                        │                          │                        │
    │                        │── 3. Serialize to RLP ───▶│                        │
    │                        │   EIP-1559 Type 2:        │                        │
    │                        │   0x02 || RLP([chainId,   │                        │
    │                        │   nonce, maxPriorityFee,  │                        │
    │                        │   maxFee, gasLimit, to,   │                        │
    │                        │   value, data, acList])   │                        │
    │                        │                          │                        │
    │                        │── 4. Fetch EIP-7730 ──────│                        │
    │                        │   clear signing metadata  │                        │
    │                        │                          │                        │
    │                        │── 5. APDU Init Chunk ────▶│── BLE Encrypt ────────▶│
    │                        │   CLA=0xE0 INS=0x04      │                        │
    │                        │   P1=0x00 (first chunk)   │                        │
    │                        │   Data=[BIP-32 path +     │                        │
    │                        │         Tx bytes part 1]  │                        │
    │                        │                          │                        │
    │                        │── 6. APDU Continue ──────▶│── Stream ─────────────▶│
    │                        │   P1=0x80 (continuation)  │                        │
    │                        │   Data=[Tx bytes part N]  │                        │
    │                        │                          │                        │── 7. Decode RLP
    │                        │                          │                        │── 8. Apply EIP-7730
    │                        │                          │                        │── 9. Render to Screen
    │                        │                          │                        │
    │◀── 10. Inspect HW ────│                          │◀── Status 0x9000 ──────│
    │    Screen (verify      │                          │    (awaiting confirm)   │
    │    amount, to, fee)    │                          │                        │
    │                        │                          │                        │
    │── 11. Physical ───────▶│                          │                        │
    │    Button Press         │                          │                        │── 12. ECDSA Sign
    │                        │                          │                        │    secp256k1
    │                        │                          │                        │    Compute (r, s, v)
    │                        │                          │◀── 13. APDU Response ──│
    │                        │◀── 14. Signature ────────│    Data=[r,s,v]        │
    │                        │                          │    SW=0x9000           │
    │                        │                          │                        │
    │                        │── 15. Verify signature ──│                        │
    │                        │   ecrecover(hash, r,s,v) │                        │
    │                        │   == expected address?    │                        │
    │                        │                          │                        │
    │                        │── 16. Assemble signed tx  │                        │
    │                        │   Append (v, r, s) to RLP │                        │
    │                        │                          │                        │
    │                        │── 17. Broadcast ──────────│                        │
    │                        │   eth_sendRawTransaction  │                        │
    │                        │   to RPC (Alchemy/Ankr)   │                        │
    │                        │                          │                        │
    │◀── 18. Tx Hash ───────│                          │                        │
    │    Confirmation        │                          │                        │
```

## 3.3 APDU Communication Protocol

All hardware wallets communicate via **ISO/IEC 7816-4 APDUs** (Application Protocol Data Units):

### Command APDU (Mobile → Hardware)

| Field | Bytes | Description |
|:---|:---:|:---|
| **CLA** | 1 | Instruction Class (`0xE0` = Ledger Ethereum, `0xD0` = Common) |
| **INS** | 1 | Instruction Code (`0x02` Get Key, `0x04` Sign Tx, `0x08` Sign Msg) |
| **P1** | 1 | Parameter 1 (`0x00` First Chunk, `0x80` Continuation) |
| **P2** | 1 | Parameter 2 (`0x01` Display Address, `0x00` Silent) |
| **Lc** | 1 | Data length (0–255 bytes) |
| **Data** | Lc | Payload (derivation path, serialized tx chunk) |
| **Le** | 0–2 | Expected response length |

### Response APDU (Hardware → Mobile)

| Field | Bytes | Description |
|:---|:---:|:---|
| **Data** | N | Public key, signature (r, s, v), or attestation |
| **SW1-SW2** | 2 | Status: `0x9000` Success, `0x6985` User Refused, `0x6A80` Bad Data |

### APDU Chunking for Large Transactions

Hardware SE buffers are typically 256 bytes. Large EVM calldata must be streamed:

```
Chunk 1: CLA=0xE0, INS=0x04, P1=0x00, Lc=255, Data=[BIP-32 Path + Tx Part 1]
Chunk 2: CLA=0xE0, INS=0x04, P1=0x80, Lc=255, Data=[Tx Part 2]
  ...
Chunk N: CLA=0xE0, INS=0x04, P1=0x80, Lc=112, Data=[Final Tx Part]
```

## 3.4 EVM Transaction Serialization

| Type | Encoding |
|:---|:---|
| **Legacy (Type 0)** | `RLP([nonce, gasPrice, gasLimit, to, value, data, v, r, s])` |
| **EIP-1559 (Type 2)** | `0x02 ∥ RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, value, data, accessList])` |
| **EIP-7702 (Type 4)** | `0x04 ∥ RLP([chainId, nonce, maxPriorityFee, maxFee, gasLimit, to, value, data, accessList, authorizationList])` |

## 3.5 Tangem NFC Signing Variant

Tangem differs from Ledger/Trezor — the mobile app hashes the transaction **before** sending it to the card:

```
FurlPay App                    Tangem Card (NFC)
    │                                │
    │── 1. Build unsigned tx         │
    │── 2. RLP serialize             │
    │── 3. Keccak-256 hash (32 bytes)│
    │                                │
    │── 4. SignHashCommand ─────────▶│  (NFC tap)
    │   (hash + derivation path)     │
    │                                │── 5. Verify PIN
    │                                │── 6. Derive key at path
    │                                │── 7. ECDSA sign hash
    │◀── 8. Return (r, s) ──────────│
    │                                │
    │── 9. Calculate v               │
    │── 10. Assemble signed tx       │
    │── 11. Broadcast to RPC         │
```

## 3.6 Trezor Host Protocol (THP) — Safe 7 BLE

The Trezor Safe 7 uses a new **Trezor Host Protocol (THP)** for BLE communication — an open-source, encrypted communication layer distinct from standard APDU:

- **Protocol**: Protobuf over encrypted BLE (not ISO 7816 APDUs like Ledger)
- **Encryption**: End-to-end encrypted packets using THP v2
- **Pairing Security**: Numeric comparison on device screen prevents MITM
- **Implementation**: Requires porting THP packet handling to JavaScript/TypeScript; raw `react-native-ble-plx` for BLE transport

---

# 4. Ledger

## 4.1 Ecosystem Overview (2025–2026)

Ledger is the market leader with **8M+ devices sold** and **>$100M revenue** in 2025. In 2025–2026, Ledger's developer ecosystem underwent a major structural transition:

> [!IMPORTANT]
> **Legacy LedgerJS** (`@ledgerhq/hw-transport`, `@ledgerhq/hw-app-eth`) and the **Ledger Live SDK** are officially **deprecated / maintenance mode**. The new standard is the **Device Management Kit (DMK)**.

## 4.2 Ledger Live Architecture

Ledger Live serves two integration tracks:
1. **Wallet API** (`@ledgerhq/wallet-api-client`) — For dApps running *inside* Ledger Live as "Live Apps" via JSON-RPC
2. **Device Management Kit (DMK)** — For *independent* mobile/desktop wallets communicating directly with Ledger hardware

## 4.3 Device Management Kit (DMK)

The DMK is Ledger's modern modular TypeScript framework replacing legacy LedgerJS:

| Module | Package | Purpose |
|:---|:---|:---|
| **Core** | `@ledgerhq/device-management-kit` | State machine, session management, APDU queueing |
| **RN BLE Transport** | `@ledgerhq/device-transport-kit-react-native-ble` | BLE for React Native (iOS/Android) |
| **RN HID Transport** | `@ledgerhq/device-transport-kit-react-native-hid` | USB HID for Android |
| **Ethereum DSK** | `@ledgerhq/device-signer-kit-ethereum` | EVM transaction & message signing |
| **Solana DSK** | `@ledgerhq/device-signer-kit-solana` | Solana signing |
| **Bitcoin DSK** | `@ledgerhq/device-signer-kit-bitcoin` | Bitcoin PSBT signing |
| **Web BLE** | `@ledgerhq/device-transport-kit-web-ble` | WebBluetooth |
| **Web HID** | `@ledgerhq/device-transport-kit-web-hid` | WebHID |

### Key DMK Improvements over Legacy
- **Session Management**: Auto-reconnection, APDU queueing, concurrent command isolation
- **Device State Observation**: Reactive observers for battery, firmware, open apps
- **Modular Transports**: Fully decoupled transport packages (DMK v0.6+)
- **AI Coding Skills**: Ledger provides AI integration skills via `npx skills add ledgerhq/agent-skills`
- **Developer Tools**: WebSocket-based loggers and Rozenite-based inspectors for debugging device sessions

## 4.4 Transport: Ledger BLE

### Legacy Package
- `@ledgerhq/react-native-hw-transport-ble` — Built on `react-native-ble-plx`
- Still works with RN 0.7x–0.8x but is in maintenance mode

### Modern Package
- `@ledgerhq/device-transport-kit-react-native-ble` — Part of DMK, recommended for new projects

### Expo Compatibility
- **Expo Go**: ❌ NOT supported (requires native BLE modules)
- **Expo Dev Builds**: ✅ Supported via `npx expo prebuild` / `eas build`
- Requires config plugin for BLE permissions:
  - **iOS**: `NSBluetoothAlwaysUsageDescription`, `NSBluetoothPeripheralUsageDescription`
  - **Android**: `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, `FINE_LOCATION`

### Required Polyfills
```javascript
import 'react-native-get-random-values';
import { Buffer } from 'buffer';
global.Buffer = Buffer;
```

## 4.5 Secure Element Architecture

```
┌───────────────────────────────────────────────────────┐
│                 LEDGER HARDWARE DEVICE                 │
│                                                       │
│  ┌─────────────────────┐   ┌───────────────────────┐ │
│  │ MCU (STM32WB55)     │   │  Secure Element (SE)  │ │
│  │ USB / BLE stack     │   │  ST33J2M0 / ST33K1M   │ │
│  │ Battery management  │   │  EAL6+ certified      │ │
│  └─────────┬───────────┘   │  BOLOS OS             │ │
│            │               │  Master seed storage   │ │
│            │  APDU         │  All crypto operations │ │
│            └──────────────▶│  Key derivation        │ │
│                            └───────────────────────┘ │
│                                                       │
│  Stax/Flex: SE directly drives display & touch       │
│  (MCU compromise cannot alter transaction visuals)    │
└───────────────────────────────────────────────────────┘
```

- **SE chip**: STMicroelectronics ST33 (EAL6+), stores master seed, runs BOLOS
- **MCU**: STM32WB55, handles USB/BLE/battery (untrusted)
- **Stax/Flex**: SE directly controls display — MCU cannot spoof screen content

## 4.6 Clear Signing vs Blind Signing

### Clear Signing (ERC-7730)
- Displays human-readable parameters on hardware screen: recipient, token, amount, function name
- Ledger spearheaded **ERC-7730** standard for clear signing metadata descriptors
- Developers publish JSON descriptors to `ethereum/clear-signing-erc7730-registry`
- Enforced across Nano X, Stax, Flex firmware (2025+)
- **Schema v2** (mid-2026): Cross-chain support and improved formatting flexibility

### Blind Signing
- Fallback when calldata cannot be decoded — shows raw hex hash
- Must be manually enabled in device app settings (disabled by default)
- Firmware shows severe security warnings

> [!WARNING]
> Blind signing is the primary root cause of modern phishing drains. FurlPay should register ERC-7730 descriptors for all supported contract interactions (LI.FI, Uniswap, etc.) to enable clear signing.

### How to Register ERC-7730 Descriptors for FurlPay

1. **Prerequisites**: Verified ABI on Sourcify for all contracts FurlPay interacts with
2. **Create descriptor JSON** with three sections:
   - `context`: Bind to specific contract addresses and chain IDs
   - `metadata`: Project name, contract details, reusable constants
   - `display`: Map function signatures to human-readable labels (e.g., "Swap 1,000 USDC for 0.42 WETH")
3. **Validate** using tools at [clearsigning.org](https://clearsigning.org)
4. **Submit PR** to official registry repository with automated linting + schema validation
5. **Multi-call support**: Create specific descriptors for `execute()` patterns in smart wallets

## 4.7 EIP-1559, ERC-20, EIP-712 Support

| Feature | Status | Details |
|:---|:---:|:---|
| **EIP-1559** | ✅ Native | Ethereum app v1.9.0+. Parses `maxPriorityFeePerGas`, `maxFeePerGas`. Clear signing renders base fee, tip, total max cost |
| **ERC-20** | ✅ Full | Dynamic token metadata loading. Decodes `transfer()`, `approve()` with human-readable amounts (e.g., `150.00 USDC`) |
| **EIP-712** | ✅ Full | Structured typed data signing with ERC-7730 metadata. Validates `verifyingContract` and `chainId`. Unregistered = blind signing fallback |

## 4.8 React Native Implementation (Modern DMK)

```typescript
import { DeviceManagementKitBuilder } from "@ledgerhq/device-management-kit";
import { reactNativeBleTransportFactory } from "@ledgerhq/device-transport-kit-react-native-ble";
import { SignerEthBuilder } from "@ledgerhq/device-signer-kit-ethereum";

// 1. Build DMK with React Native BLE transport
const dmk = new DeviceManagementKitBuilder()
  .addTransport(reactNativeBleTransportFactory)
  .build();

// 2. Start device discovery
dmk.startDiscovering({
  onDeviceDiscovered: async (device) => {
    // 3. Connect to discovered device
    const sessionId = await dmk.connect({ deviceId: device.id });

    // 4. Initialize Ethereum Signer Kit
    const ethSigner = new SignerEthBuilder({ dmk, sessionId }).build();

    // 5. Get address (with on-device verification)
    const address = await ethSigner.getAddress("44'/60'/0'/0/0", {
      displayOnDevice: true,
    });

    // 6. Sign EIP-1559 transaction
    const signature = await ethSigner.signTransaction(
      "44'/60'/0'/0/0",
      unsignedTxRlpHex
    );
    // signature => { r, s, v }
  },
});
```

## 4.9 Licensing & Maintenance

- **SDK License**: Apache 2.0 (DMK, DSK, Transport Kits, LedgerJS)
- **iOS wrappers**: MIT
- **BOLOS firmware**: Closed source (NDA)
- **All hardware apps**: Open source (Apache 2.0)
- **Maintenance**: 🟢 Actively developed, breaking changes tracked in DMK changelog

## 4.10 Production Apps Using Ledger Mobile SDK

| App | Integration | SDK Used |
|:---|:---|:---|
| MetaMask Mobile | Direct BLE | `@ledgerhq/react-native-hw-transport-ble` / DMK |
| Rainbow Wallet | Direct BLE | Custom RN BLE + Ledger Transport |
| Zerion Wallet | Direct BLE + WalletConnect | Ledger Transport + WC v2 |
| Phantom Mobile | Direct BLE | Solana & EVM Ledger BLE |
| 1inch Mobile | Direct BLE + WalletConnect | Ledger Transport |
| Uniswap Mobile | WalletConnect | WC v2 → Ledger Live Mobile |

---

# 5. Trezor

## 5.1 Trezor Connect (`@trezor/connect`)

### Architecture
- Maintained in the `trezor-suite` monorepo as `@trezor/connect`
- **Web**: Injects an iframe sandbox hosted on `connect.trezor.io` for state management + popup for user approvals
- **Mobile**: Uses `@trezor/connect-mobile` for native embedding, or deep-links to Trezor Suite Mobile
- **v9.6.0+**: Direct WebSocket connection to local Trezor Suite desktop/mobile app (bypasses popups)
- **Safe 7 support**: v9.6.0+ includes full BLE connectivity for Safe 7

### Communication Protocol
- **Protobuf over USB/HID/BLE** (not ISO 7816 APDUs like Ledger)
- **Trezor Host Protocol (THP) v2**: Required for Safe 7 BLE — end-to-end encrypted packet exchange
- Stateless, reactive message system
- Message schemas in `common/protob` in `trezor-firmware` monorepo

## 5.2 Trezor Suite

- **Fully open source** under T-RSL (Trezor Reference Software License)
- Mobile app: `suite-native/` in monorepo, built with **React Native**
- Shared business logic across Desktop, Web, Mobile via `@trezor/suite-common`

## 5.3 Mobile Support

| Platform | USB | BLE | Notes |
|:---|:---:|:---:|:---|
| **Android** | ✅ USB OTG | ✅ Safe 7 | Direct WebUSB / Android USB Host API |
| **iOS** | ⚠️ Restricted | ✅ Safe 7 | No MFi auth for custom USB HID. BLE on Safe 7 provides full experience |

### React Native Feasibility
- **Official Package**: `@trezor/connect-mobile`
- **Reference Code**: `suite-native` + `packages/connect-examples/mobile-expo`
- **Expo Go**: ❌ NOT supported
- **Expo Dev Builds**: ✅ Supported (requires `npx expo prebuild`)
- **BLE Library**: `react-native-ble-plx` for raw BLE + THP protocol layer on top

## 5.4 Bluetooth

> [!IMPORTANT]
> **Trezor Safe 7** (released late 2025) is the **first and only** Trezor model with Bluetooth Low Energy (BLE) and Qi2 wireless charging. All older models (Model One, Model T, Safe 3, Safe 5) are USB-only.

## 5.5 Firmware Architecture & Secure Elements

| Model | SE | MCU | Passphrase Entry |
|:---|:---|:---|:---|
| Model One | None | STM32F2 | Host only |
| Model T | None | STM32F4 | On-device / Host |
| Safe 3 | EAL6+ OPTIGA Trust M | STM32F4 | On-device / Host |
| Safe 5 | EAL6+ OPTIGA Trust M | STM32F4 | On-device / Host |
| **Safe 7** | **Dual: OPTIGA + TROPIC01** | STM32U5 | On-device / Host |

- **TROPIC01**: SatoshiLabs / Tropic Square open-source, independently auditable SE
- **Triple-layer defense** on Safe 7: TROPIC01 + OPTIGA Trust M V3 + hardened STM32U5

## 5.6 Licensing

- `trezor-firmware`: **GPL-3.0** (copyleft)
- `trezor-suite` (including `@trezor/connect`): **T-RSL** (source-available, non-commercial compatible)

## 5.7 EVM Chain Support (2025–2026)

Added Ethereum, Base, Optimism, Arbitrum One, Polygon, and many more EVM chains. Full EIP-1559, ERC-20, EIP-712 support.

---

# 6. Tangem

## 6.1 NFC Protocol

- **Physical**: ISO/IEC 14443-4 (Type A/B) at 13.56 MHz
- **Application**: ISO/IEC 7816-4 APDU command/response
- **Secure Channel**: ECDH key exchange → AES-256/ChaCha20 encrypted APDUs
- All APDU payloads are encrypted — protects against NFC sniffing and relay attacks

## 6.2 Card Authentication (Attestation)

```
FurlPay App                  Tangem Card (SE)           Tangem Root CA
    │                              │                          │
    │── 1. Generate 32-byte ──────▶│                          │
    │      nonce challenge         │                          │
    │                              │── 2. Sign(nonce +        │
    │                              │   serial + pubkeys)      │
    │                              │   with factory key       │
    │◀── 3. Return signature ──────│                          │
    │      + cert chain            │                          │
    │                                                         │
    │── 4. Verify cert chain ─────────────────────────────────│
    │── 5. Verify signature        │                          │
    │      = Verified / Failed     │                          │
```

## 6.3 Key Generation (Seedless Model)

- Private keys generated inside **EAL6+ Secure Element** via hardware TRNG (thermal quantum noise)
- Key **never leaves the SE silicon** — cannot be read, exported, or extracted
- **Seedless backup**: 2- or 3-card set. Key shares encrypted with user PIN and transferred card-to-card via NFC
- **Tangem 2.0**: Optional BIP-39 seed phrase support for cross-wallet recovery

## 6.4 SDK (2026 Updated)

### Native SDKs (Actively Maintained)
| Platform | Package | Language | Distribution | Status |
|:---|:---|:---|:---|:---|
| iOS | `tangem-sdk-ios` | Swift | SPM + CocoaPods | 🟢 Active |
| Android | `tangem-sdk-android` | Kotlin | Maven / GitHub Packages | 🟢 Active |
| Flutter | `tangem-sdk-flutter` | Dart | pub.dev | 🟢 Active |

### React Native SDK (2026 Status)

> [!TIP]
> As of 2026, the **`tangem-sdk-react-native`** package on GitHub has received community updates and is usable for basic integration. It bridges `tangem-sdk-ios` (Swift) and `tangem-sdk-android` (Kotlin) via native modules.
>
> For **best performance and full API access**, consider writing custom **TurboModules** that wrap the native SDKs directly — this gives you access to the latest card firmware features (Yield Mode, Tangem Ring, etc.) without waiting for the RN bridge to update.

### Tangem 2026 Ecosystem Features
- **Tangem Ring**: Wearable NFC ring — same secure chip, same SDK integration
- **Tangem Pay**: Virtual Visa payment flow (crypto-to-fiat)
- **Yield Mode**: Native DeFi integration with Aave for automated yield
- **16,000+ tokens across 90+ blockchains** supported
- **Enhanced swap integrations**: Multiple CEX/DEX providers
- **Native NFT support**: Collection management

### Required Permissions
- **Android**: `<uses-permission android:name="android.permission.NFC" />`
- **iOS**: `NFCReaderUsageDescription` + ISO 7816 select identifiers for Tangem AID (`A0000008120101`)

## 6.5 Tangem vs Ledger UX Comparison

| Dimension | Tangem (NFC Tap) | Ledger (BLE) |
|:---|:---|:---|
| **Connection Setup** | Zero pairing. Instant tap. | BLE pairing with passkey confirmation |
| **Signing Speed** | Sub-second (<1s NFC tap) | 5–15s (BLE handshake + button presses) |
| **Battery** | **None** (passive NFC, powered by phone) | Active battery (degrades, requires charging) |
| **Form Factor** | Credit card (0.8mm, IP68) | Pocket device (bulkier, needs case/cable) |
| **On-Device Display** | ❌ No screen | ✅ Trusted display for WYSIWYS |
| **Security Model** | Transaction verified on mobile app only | Transaction verified on hardware screen |
| **Wireless Range** | ~1–4 cm (prevents remote attacks) | ~10 m (requires BLE security pairing) |

> [!CAUTION]
> **Tangem has no on-device display.** Transaction details are verified on the (untrusted) mobile phone screen, not on the hardware. This is a meaningful security trade-off vs. Ledger's trusted display. For high-value transactions, Ledger's WYSIWYS model provides stronger assurance against mobile OS compromise. **FurlPay should compensate** with transaction simulation and clear warnings for high-value operations.

---

# 7. Keystone

## 7.1 Air-Gapped QR Signing

Keystone uses **animated QR codes** based on the **UR (Uniform Resource)** standard for completely air-gapped signing:

```
FurlPay App                            Keystone 3 Pro
    │                                       │
    │── 1. Build unsigned tx                │
    │── 2. Encode as UR/CBOR                │
    │── 3. Display animated QR ────────────▶│  (device camera scans)
    │      (multi-frame fountain code)      │
    │                                       │── 4. Parse UR data
    │                                       │── 5. Display tx details
    │                                       │── 6. User confirms on screen
    │                                       │── 7. Sign with SE key
    │   (phone camera scans)                │
    │◀── 8. Display signed QR ──────────────│
    │      (animated response)              │
    │── 9. Parse signature                  │
    │── 10. Broadcast to RPC               │
```

## 7.2 React Native Integration

```bash
npm install @keystonehq/keystone-sdk @keystonehq/animated-qr
```

```typescript
import { AnimatedQRCode, AnimatedQRScanner } from '@keystonehq/animated-qr';
import KeystoneSDK, { UR } from '@keystonehq/keystone-sdk';

// Display QR for device to scan
<AnimatedQRCode cbor={encodedTxData} type="bytes" />

// Scan signed response from device
<AnimatedQRScanner
  handleScan={({ type, cbor }) => {
    const sig = KeystoneSDK.parseSignature(
      new UR(Buffer.from(cbor, 'hex'), type)
    );
  }}
  handleError={console.error}
/>
```

## 7.3 Production Apps
- MetaMask, OKX Wallet, Phantom, Rabby, Safe{Wallet}, Sparrow, Blue Wallet

---

# 8. OneKey

## 8.1 React Native BLE SDK

OneKey provides a **full React Native BLE SDK** that is the most straightforward hardware wallet integration for RN:

### Core Packages
```bash
npm install @onekeyfe/hd-ble-sdk @onekeyfe/hd-transport-react-native
```

### Key Concepts
- **`connectId`**: Routes calls to a specific device/BLE connection
- **`deviceId`**: Identifies the currently loaded seed/wallet state
- **`UI_EVENT`**: Must subscribe early to handle PIN and passphrase prompts without stalling the request queue
- **Pure RN stack**: No WebViews or manual low-level adapter management needed

### 100% Open Source
- Hardware schematics, firmware, and software all open source
- GitHub: `OneKeyHQ/hardware-js-sdk`
- Developer portal: `developer.onekey.so`

---

# 9. WalletConnect as Abstraction Layer

## 9.1 Can WalletConnect Replace Native SDKs?

**WalletConnect (Reown) is a transport/relay abstraction, not a native hardware driver.** It delegates signing to companion wallet apps (Ledger Live, SafePal, OneKey) that internally handle hardware communication.

### How It Works for Hardware Wallets
```
FurlPay App ──WebSocket──▶ WalletConnect Relay ──WebSocket──▶ Companion Wallet App
                                                                      │
                                                              BLE/NFC/USB/QR
                                                                      │
                                                              Hardware Device
```

### Reown AppKit (2026)
- Formerly WalletConnect Modal — now **Reown AppKit**
- Supports **500+ wallets** including all major hardware wallet companion apps
- Unified cross-chain interface with pre-built UI components

## 9.2 React Native Setup (Latest 2026)

```typescript
import "@walletconnect/react-native-compat"; // MUST BE FIRST IMPORT
import { createAppKit, defaultWagmiConfig } from "@reown/appkit-react-native";

createAppKit({
  projectId: "YOUR_REOWN_PROJECT_ID", // from dashboard.reown.com
  metadata: {
    name: "FurlPay",
    description: "Self-custody crypto wallet",
    url: "https://furlpay.com",
    icons: ["https://furlpay.com/icon.png"],
    redirect: { native: "furlpay://" },
  },
  networks: [/* ... */],
  adapters: [/* ... */],
});

// Pre-built UI components
<AppKitButton />  // or <ConnectButton />
```

### Required Dependencies
```bash
npm install @reown/appkit-react-native \
  @walletconnect/react-native-compat \
  @react-native-async-storage/async-storage \
  react-native-get-random-values \
  react-native-svg \
  @react-native-community/netinfo \
  react-native-safe-area-context \
  expo-application
```

## 9.3 Verdict

> [!TIP]
> **Use BOTH approaches.** WalletConnect provides broad compatibility (500+ wallets) with minimal code. Native SDKs provide superior UX for the wallets your users care most about (Ledger, Tangem).
>
> **Recommended Strategy:**
> - Native SDK for Ledger BLE (primary hardware wallet)
> - Native SDK for Tangem NFC (best mobile UX)
> - WalletConnect v2 as the catch-all for all other hardware wallets

---

# 10. Security Best Practices

## 10.1 Address Verification

- **On-device display**: Hardware must show the full address on its trusted screen
- **Derivation path integrity**: Use standard BIP-44 paths (`m/44'/60'/0'/0/0`)
- **User visual comparison**: Always trigger display flag (`P1=0x01`) for `GET_PUBLIC_KEY`
- **EIP-55 checksums**: Enforce mixed-case checksum encoding for all EVM addresses
- **Tangem exception**: No on-device display — FurlPay must clearly show address and require explicit user acknowledgment

## 10.2 Transaction Simulation (Critical 2026 Feature)

> [!IMPORTANT]
> **Transaction simulation is now an industry standard.** Leading wallets (MetaMask, Rabby, Phantom) simulate transactions before signing to detect:
> - Malicious contract interactions ("drainer" scripts)
> - Token approval exploits (unlimited `approve()`)
> - Suspicious destination addresses (flagged on scam databases)
> - Unexpected balance changes

**FurlPay Implementation**:
1. Before sending tx to hardware wallet, call a simulation API (Tenderly, Blowfish, BlockAid)
2. Display simulated outcome: "You will send 1.5 ETH and receive ~3,200 USDC"
3. Show **risk score** and warnings for flagged contracts
4. Only proceed to hardware signing after user reviews simulation

## 10.3 Clear Signing (ERC-7730)

- Register ERC-7730 descriptors for all FurlPay contract interactions
- Map function selectors to human-readable labels
- Validate EIP-712 `verifyingContract` and `chainId` to prevent cross-chain replay
- Display high-risk warning for any blind signing fallback

## 10.4 Firmware & Device Verification

- Bootloaders enforce ECDSA/RSA signature verification on firmware images
- Hardware eFuses (RDP2) prevent JTAG/SWD access
- SHA-256 hash verification of application partition on every power cycle
- Anti-tamper mesh on Secure Elements triggers zeroization on physical probe

## 10.5 Device Attestation

```
Mobile App ──1. 32-byte nonce──▶ Hardware SE
                                     │
                                     │── 2. Sign(nonce + serial + fw_hash)
                                     │      with factory-injected private key
                                     │
Mobile App ◀──3. Signature + cert chain──
    │
    │── 4. Verify cert chain against vendor Root CA
    │── 5. Verify signature over nonce
    │── Result: genuine / compromised
```

## 10.6 Bluetooth MITM Prevention

- **Mandatory**: BLE 4.2+ LE Secure Connections with ECDH P-256 key exchange
- **Disable "Just Works"** pairing — require Numeric Comparison or Passkey Entry
- 6-digit code displayed on hardware screen, verified on mobile
- **AES-CCM** channel encryption with monotonic packet counters (anti-replay)

## 10.7 NFC Relay Attack Prevention

- **Tight APDU timeouts** (<300ms) — relay networks add 50–500ms latency
- **Distance bounding protocols** (ISO 14443) — RTT measurement
- **Physical touch requirement** — remote relayed APDU fails without local touch event
- Tangem's ECDH-encrypted secure channel adds additional protection

## 10.8 Recovery Phrase Handling

- **NEVER** transmit seed/recovery phrase over BLE, USB, or NFC
- Generate BIP-39 seed inside hardware TRNG (FIPS 140-2 / AIS-31 certified)
- Display once on hardware screen for manual backup
- Support SLIP-0039 Shamir's Secret Sharing (on-device, e.g., Trezor)
- Tangem alternative: encrypted card-to-card backup (no seed phrase needed)

---

# 11. UX Design

## 11.1 Onboarding Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Welcome     │     │  Select HW   │     │  Connection  │     │  Import      │
│  Screen      │────▶│  Wallet Type │────▶│  Guide       │────▶│  Accounts    │
│              │     │  • Ledger     │     │  (BLE/NFC/   │     │  (derive     │
│  "Connect    │     │  • Tangem     │     │   USB/QR)    │     │   addresses) │
│   Hardware"  │     │  • Trezor     │     │              │     │              │
│   button     │     │  • Keystone   │     │  Step-by-step│     │  Select      │
│              │     │  • Other (WC) │     │  with images  │     │  chains      │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

### Key Principles
- Auto-detect connected hardware via BLE scan or NFC proximity
- Show device-specific visuals (Nano X shape, Tangem card, Trezor Safe 7)
- Remind users: "Open the Ethereum app on your Ledger" with animated guide
- Verify device authenticity during setup (attestation challenge)

## 11.2 Pairing

### Ledger BLE Pairing
1. Start BLE scan → show discovered devices with signal strength
2. User taps device → initiate BLE Secure Connection
3. Display 6-digit pairing code on both screens → user confirms match
4. Connection established → verify device authenticity
5. Cache device ID for auto-reconnect

### Tangem NFC Pairing
1. Display "Tap your Tangem card" with phone NFC zone highlighted
2. User taps card → instant connection (<1s)
3. Run attestation → verify card genuineness
4. No persistent pairing needed — each tap is a fresh session

### Keystone QR Pairing
1. Display animated QR with device sync data
2. User scans QR with Keystone camera → imports account
3. No wireless connection needed — completely air-gapped

## 11.3 Error State Handling

| Error State | UX Response |
|:---|:---|
| **Device locked** | "Unlock your [device] and try again" with PIN entry illustration |
| **Wrong app open** | "Open the Ethereum app on your Ledger" with animated guide |
| **Battery low** | "Your Ledger battery is low (15%). Connect to a charger." |
| **Firmware outdated** | "A firmware update is available. Update via [Ledger Live / Trezor Suite]" (link) |
| **BLE disconnected** | Non-blocking banner: "Reconnecting…" → auto-retry 3× → "Reconnect" button |
| **NFC read failed** | "Hold your card steady for a moment" with haptic feedback |
| **User rejected** | "Transaction cancelled on device" — return to review screen |
| **Blind signing required** | ⚠️ High-risk warning: "This transaction cannot be fully verified on your device" |
| **Simulation warning** | 🔴 "This transaction may be malicious" with risk details |

## 11.4 Transaction Approval UX (Enhanced 2026)

```
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│  Transaction Review  │   │  Simulation Result   │   │  Hardware Signing    │   │  Confirmation        │
│                     │   │                     │   │                     │   │                     │
│  Send 1.5 ETH       │   │  ✅ Safe to sign     │   │  ┌───────────────┐  │   │  ✅ Tx Sent          │
│  To: 0x1234...5678  │   │                     │   │  │ Verify on     │  │   │                     │
│  Network: Ethereum  │   │  You will:          │   │  │ Ledger Nano X │  │   │  Tx: 0xabc...       │
│  Max Fee: 0.003 ETH │   │  • Send 1.5 ETH     │   │  │               │  │   │  View Explorer →    │
│                     │──▶│  • Receive nothing   │──▶│  │ [Device anim] │  │──▶│                     │
│  [Review & Sign]    │   │  • Gas: ~0.003 ETH   │   │  └───────────────┘  │   │  [Done]             │
│                     │   │                     │   │                     │   │                     │
│                     │   │  No risks detected   │   │  Press both buttons │   │                     │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘   └─────────────────────┘
```

## 11.5 Multiple Hardware Wallets

- Support multiple paired devices simultaneously
- Account list shows hardware icon badge (Ledger/Tangem/Trezor/Keystone)
- Settings → "Manage Hardware Wallets" → list connected devices, add/remove
- When signing, auto-detect which device owns the account
- If device not connected: prompt user to connect the specific device

---

# 12. Architecture Recommendation

## 12.1 Plugin Architecture with Dependency Inversion

```
┌───────────────────────────────────────────────────────┐
│              FurlPay Domain Layer                       │
│  (Transaction Engine, UI State, Account Management)    │
│                                                       │
│           depends on abstractions only                │
└──────────────────────┬────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────┐
│            HardwareSigner Interface                    │
│            DeviceDiscovery Interface                   │
│            Transport Interface                        │
└──────────────────────┬────────────────────────────────┘
                       │
       ┌───────────────┼───────────────┬─────────────────┐
       ▼               ▼               ▼                 ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────────┐
│ LedgerPlugin│ │ TangemPlugin│ │ TrezorPlugin│ │ WCPlugin     │
│ (DMK SDK)   │ │ (NFC Native)│ │ (Connect)   │ │ (Reown/WC)   │
└──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬───────┘
       │               │               │               │
       ▼               ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────────┐
│ BLE         │ │ NFC         │ │ BLE / USB   │ │ WebSocket    │
│ Transport   │ │ Transport   │ │ Transport   │ │ Relay        │
└─────────────┘ └─────────────┘ └─────────────┘ └──────────────┘
```

## 12.2 Core Interface Definitions

```typescript
// ═══════════════════════════════════════════════════════
// Transport Layer Abstraction
// ═══════════════════════════════════════════════════════

export interface Transport {
  readonly type: "ble" | "usb" | "nfc" | "qr" | "walletconnect";
  readonly isConnected: boolean;

  connect(): Promise<void>;
  disconnect(): Promise<void>;

  /** Transmit raw APDU frame and await response */
  exchange(apdu: Uint8Array): Promise<Uint8Array>;

  on(event: "disconnect", listener: (error?: Error) => void): void;
  on(event: "statusChange", listener: (status: TransportStatus) => void): void;
}

export type TransportStatus =
  | "scanning"
  | "connecting"
  | "connected"
  | "disconnected"
  | "error";

// ═══════════════════════════════════════════════════════
// Device Discovery
// ═══════════════════════════════════════════════════════

export interface DiscoveredDevice {
  id: string;
  name: string;
  vendor: HardwareVendor;
  model?: string;
  transportType: Transport["type"];
  rssi?: number; // BLE signal strength
}

export interface DeviceDiscovery {
  startScanning(
    onDeviceFound: (device: DiscoveredDevice) => void
  ): Promise<void>;
  stopScanning(): Promise<void>;
  getSupportedTransports(): Transport["type"][];
}

// ═══════════════════════════════════════════════════════
// Hardware Wallet Abstraction
// ═══════════════════════════════════════════════════════

export type HardwareVendor =
  | "ledger"
  | "trezor"
  | "tangem"
  | "keystone"
  | "onekey"
  | "walletconnect";

export interface DeviceCapabilities {
  supportsClearSigning: boolean; // ERC-7730
  supportsOnDeviceDisplay: boolean;
  supportsEIP712: boolean;
  supportsEIP1559: boolean;
  supportsEIP7702: boolean;
  supportsSessionKeys: boolean;
  supportedChainIds: number[];
  supportedCurves: ("secp256k1" | "ed25519" | "secp256r1")[];
  hasSecureElement: boolean;
  hasBattery: boolean;
}

export interface HardwareWallet {
  readonly vendor: HardwareVendor;
  readonly deviceId: string;
  readonly transport: Transport;

  getCapabilities(): DeviceCapabilities;

  /** Cryptographic device attestation against vendor PKI */
  verifyAuthenticity(): Promise<{
    isGenuine: boolean;
    certChain?: string[];
    firmwareVersion?: string;
  }>;

  /** Check device readiness (correct app open, unlocked, etc.) */
  checkReady(chainId: number): Promise<{
    ready: boolean;
    issue?: "locked" | "wrong_app" | "outdated_firmware" | "battery_low";
    message?: string;
  }>;
}

// ═══════════════════════════════════════════════════════
// Hardware Signer (Used by Transaction Engine)
// ═══════════════════════════════════════════════════════

export interface HardwareSigner {
  readonly wallet: HardwareWallet;

  getAddress(
    derivationPath: string,
    displayOnDevice?: boolean
  ): Promise<{
    publicKey: string;
    address: string; // EIP-55 checksummed
  }>;

  signTransaction(
    tx: UnsignedTransaction,
    derivationPath: string,
    clearSigningDescriptor?: ClearSigningDescriptor
  ): Promise<{
    r: string;
    s: string;
    v: number;
    serializedSignedTx: string;
  }>;

  signMessage(
    message: string | EIP712TypedData,
    derivationPath: string
  ): Promise<string>;

  signHash?(
    hash: Uint8Array,
    derivationPath: string
  ): Promise<{ r: string; s: string; v: number }>;

  /** Sign EIP-7702 authorization for smart account delegation */
  signAuthorization?(
    authorization: EIP7702Authorization,
    derivationPath: string
  ): Promise<string>;
}

// ═══════════════════════════════════════════════════════
// Plugin Registry
// ═══════════════════════════════════════════════════════

export interface HardwareWalletPlugin {
  readonly vendor: HardwareVendor;
  readonly displayName: string;
  readonly icon: string;

  createDiscovery(): DeviceDiscovery;
  createWallet(device: DiscoveredDevice): Promise<HardwareWallet>;
  createSigner(wallet: HardwareWallet): HardwareSigner;
}

export class HardwareWalletRegistry {
  private plugins = new Map<HardwareVendor, HardwareWalletPlugin>();

  register(plugin: HardwareWalletPlugin): void {
    this.plugins.set(plugin.vendor, plugin);
  }

  getPlugin(vendor: HardwareVendor): HardwareWalletPlugin | undefined {
    return this.plugins.get(vendor);
  }

  getAllPlugins(): HardwareWalletPlugin[] {
    return Array.from(this.plugins.values());
  }

  async discoverAll(
    onDeviceFound: (device: DiscoveredDevice) => void
  ): Promise<void> {
    const discoveries = this.getAllPlugins().map((p) => p.createDiscovery());
    await Promise.all(discoveries.map((d) => d.startScanning(onDeviceFound)));
  }
}
```

---

# 13. Best-in-Class Features for FurlPay

## 13.1 Feature Priority Matrix

These features represent the **state-of-the-art** for hardware wallet-integrated mobile wallets in 2026. FurlPay should implement all of them to achieve a best-in-class experience:

### 🔴 Must-Have (MVP)

| # | Feature | Description | Why Critical |
|:---|:---|:---|:---|
| 1 | **Transaction Simulation** | Simulate every tx before hardware signing. Show expected balance changes, gas costs, and risk scores. | Industry standard since 2025. Prevents >90% of phishing losses. |
| 2 | **Clear Signing (ERC-7730)** | Register descriptors for all FurlPay contract interactions (LI.FI, Uniswap, bridges). | Eliminates blind signing risk. Required for Ledger firmware 2025+. |
| 3 | **Multi-Chain Address Derivation** | Derive addresses for all supported chains from single hardware seed. `m/44'/60'/0'/0/N` per account. | Users expect all EVM chains from one hardware setup. |
| 4 | **Device Attestation** | Verify hardware genuineness cryptographically during first connection. | Prevents supply-chain attacks (fake/modified devices). |
| 5 | **Auto-Reconnect & Session Caching** | Cache BLE device ID, auto-reconnect on app foreground, persist session metadata. | Without this, every signing event requires re-pairing. |
| 6 | **Graceful Error Recovery** | Detect device locked, wrong app, battery low, BLE disconnect. Non-blocking banners, not modals. | Hardware integration is fragile; errors must be recoverable. |

### 🟡 High Priority (V2)

| # | Feature | Description | Why Important |
|:---|:---|:---|:---|
| 7 | **EIP-7702 Smart Account Delegation** | Allow hardware wallet EOAs to delegate to smart contract code. Enables session keys, batching, gas sponsorship. | The biggest UX improvement since EIP-1559. Already on mainnet. |
| 8 | **Session Keys** | Hardware signs a scoped permission grant → mobile executes micro-txs with ephemeral key. Define: spending limits, allowed contracts, expiration. | Web2-like UX without sacrificing root security. |
| 9 | **Gas Sponsorship (Paymasters)** | Sponsor gas fees for hardware wallet users via ERC-4337 Paymasters. | Eliminates "need ETH for gas" friction. |
| 10 | **Transaction Batching** | Combine multiple operations into single UserOperation. E.g., approve + swap in one hardware confirmation. | Reduces hardware button presses from 2→1 for complex flows. |
| 11 | **Multi-Device Management** | Support multiple hardware wallets simultaneously. Badge accounts with device icons. Auto-detect which device owns which account. | Power users have multiple devices. |
| 12 | **Swap/Bridge with HW Signing** | Full LI.FI swap and bridge execution with hardware-signed transactions. Show source/dest tokens, slippage, protocol on review screen. | Core FurlPay feature — must work seamlessly with hardware. |

### 🟢 Differentiators (V3+)

| # | Feature | Description | Why Differentiating |
|:---|:---|:---|:---|
| 13 | **Passkey Hybrid Signing** | Use phone Secure Enclave (Face ID) as fast signer for small amounts; require hardware for large amounts. Configurable threshold. | Best of both worlds. No other mobile wallet does this well. |
| 14 | **Social Recovery Integration** | Hardware wallet as primary signer for a Safe multisig. Social guardians as recovery signers. | Solves "lost hardware" problem without seed phrases. |
| 15 | **DeFi Position Management** | View and manage DeFi positions (staking, lending, LP) with hardware-signed exits. | Active DeFi users need position visibility in their HW wallet app. |
| 16 | **NFT Display & Transfer** | View hardware-held NFT collections. Transfer with hardware signing. | Growing demand; Tangem already supports this natively. |
| 17 | **Portfolio Dashboard** | Real-time portfolio value across all chains. Hardware-secured accounts highlighted. | Provides the "at a glance" view users expect. |
| 18 | **Institutional Policy Engine** | Multi-signer approval workflows. Velocity limits (max $/day). Timelock for large transfers. | Enables enterprise/team usage with hardware wallets. |

## 13.2 Transaction Simulation Architecture

```
┌─────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│ User builds │     │  FurlPay Simulation   │     │  Simulation API  │
│ transaction  │────▶│  Engine              │────▶│  (Tenderly /     │
│              │     │                      │     │   Blowfish /     │
│              │     │  1. Serialize tx     │     │   BlockAid)      │
│              │     │  2. Send to sim API  │     │                  │
│              │     │  3. Parse response   │     │  Returns:        │
│              │     │  4. Risk assessment  │     │  • Balance deltas│
│              │     │  5. Display preview  │     │  • Risk score    │
│              │◀────│                      │◀────│  • Flagged addrs │
│              │     │  User approves?      │     │  • Approval risks│
│              │     │        │             │     └─────────────────┘
│              │     │        ▼             │
│              │     │  Send to HW wallet   │
│              │     └──────────────────────┘
└─────────────┘
```

## 13.3 Session Keys Architecture with Hardware Wallet

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SESSION KEY FLOW                                     │
│                                                                         │
│  SETUP (One-time, requires hardware):                                   │
│  ┌──────────┐     ┌──────────────────┐     ┌────────────────────┐      │
│  │ FurlPay  │     │ EIP-7702         │     │ Hardware Wallet    │      │
│  │ Mobile   │────▶│ Authorization    │────▶│ Signs delegation   │      │
│  │          │     │ (delegate EOA →  │     │ (one-time setup)   │      │
│  │          │     │  smart contract) │     │                    │      │
│  └──────────┘     └──────────────────┘     └────────────────────┘      │
│                                                                         │
│  ┌──────────┐     ┌──────────────────┐     ┌────────────────────┐      │
│  │ Generate │     │ Scoped Grant     │     │ Hardware Wallet    │      │
│  │ ephemeral│────▶│ • Max $500/day   │────▶│ Signs permission   │      │
│  │ key pair │     │ • Only LI.FI     │     │ grant              │      │
│  │ (in SE)  │     │ • Expires 24h    │     │                    │      │
│  └──────────┘     └──────────────────┘     └────────────────────┘      │
│                                                                         │
│  DAILY USE (No hardware needed):                                        │
│  ┌──────────┐     ┌──────────────────┐     ┌────────────────────┐      │
│  │ FurlPay  │     │ Session Key      │     │ Bundler Service    │      │
│  │ Mobile   │────▶│ Signs UserOp     │────▶│ Broadcasts tx      │      │
│  │ (biom.)  │     │ (within limits)  │     │ (Pimlico/Alchemy)  │      │
│  └──────────┘     └──────────────────┘     └────────────────────┘      │
│                                                                         │
│  EXPIRY:                                                                │
│  Session expires → FurlPay prompts "Tap hardware to renew permissions" │
└─────────────────────────────────────────────────────────────────────────┘
```

## 13.4 Passkey + Hardware Hybrid Security Model

```
┌─────────────────────────────────────────────────────┐
│              TIERED SECURITY MODEL                   │
│                                                     │
│  Tier 1 — Quick Actions (Face ID / Secure Enclave)  │
│  ┌───────────────────────────────────────────────┐  │
│  │ • Send < $100                                 │  │
│  │ • Swap < $500                                 │  │
│  │ • View balances / addresses                   │  │
│  │ • Sign session key renewals                   │  │
│  │ Auth: Phone biometric (< 1 second)            │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  Tier 2 — Standard Operations (Hardware Required)   │
│  ┌───────────────────────────────────────────────┐  │
│  │ • Send $100–$10,000                           │  │
│  │ • Swap > $500                                 │  │
│  │ • Bridge operations                           │  │
│  │ • Token approvals                             │  │
│  │ Auth: Hardware wallet (5–15 seconds)           │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  Tier 3 — Critical Operations (Hardware + Delay)    │
│  ┌───────────────────────────────────────────────┐  │
│  │ • Send > $10,000                              │  │
│  │ • Change recovery settings                    │  │
│  │ • Export addresses                            │  │
│  │ • Modify session key permissions              │  │
│  │ Auth: Hardware + 24h timelock                 │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

# 14. FurlPay Hardware Wallet SDK Design

## 14.1 Why Build a FurlPay SDK?

FurlPay should publish its own open-source SDK (`@furlpay/hardware-wallet-sdk`) that:

1. **Abstracts vendor complexity**: One API for all hardware wallets (Ledger, Tangem, Trezor, Keystone, OneKey, WalletConnect)
2. **Integrates with ERC-4337/EIP-7702**: Smart account signing built into the signer interface
3. **Includes transaction simulation**: Pre-signing safety checks baked into the SDK
4. **Provides React Native components**: Ready-to-use UI for pairing, signing, and error handling
5. **Opens the ecosystem**: Other React Native wallet developers can use FurlPay's SDK

## 14.2 SDK Package Structure (Monorepo)

```
@furlpay/hardware-wallet-sdk/
├── packages/
│   ├── core/                          # @furlpay/hw-core
│   │   ├── src/
│   │   │   ├── interfaces/            # ISigner, ITransport, IDiscovery
│   │   │   ├── registry/              # HardwareWalletRegistry
│   │   │   ├── simulation/            # TransactionSimulator
│   │   │   ├── clear-signing/         # ERC-7730 descriptor loader
│   │   │   └── types/                 # Shared TypeScript types
│   │   └── package.json
│   │
│   ├── plugin-ledger/                 # @furlpay/hw-plugin-ledger
│   │   ├── src/
│   │   │   ├── LedgerPlugin.ts
│   │   │   ├── LedgerSigner.ts
│   │   │   └── LedgerDiscovery.ts
│   │   └── package.json
│   │
│   ├── plugin-tangem/                 # @furlpay/hw-plugin-tangem
│   │   ├── src/
│   │   │   ├── TangemPlugin.ts
│   │   │   ├── TangemSigner.ts
│   │   │   └── TangemNfcBridge.ts     # TurboModule wrapper
│   │   ├── ios/                       # Swift native module
│   │   ├── android/                   # Kotlin native module
│   │   └── package.json
│   │
│   ├── plugin-trezor/                 # @furlpay/hw-plugin-trezor
│   │   ├── src/
│   │   │   ├── TrezorPlugin.ts
│   │   │   ├── TrezorSigner.ts
│   │   │   └── TrezorHostProtocol.ts  # THP v2 implementation
│   │   └── package.json
│   │
│   ├── plugin-keystone/               # @furlpay/hw-plugin-keystone
│   │   ├── src/
│   │   │   ├── KeystonePlugin.ts
│   │   │   ├── KeystoneSigner.ts
│   │   │   └── QrTransport.ts
│   │   └── package.json
│   │
│   ├── plugin-onekey/                 # @furlpay/hw-plugin-onekey
│   │   ├── src/
│   │   │   ├── OneKeyPlugin.ts
│   │   │   ├── OneKeySigner.ts
│   │   │   └── OneKeyBleTransport.ts
│   │   └── package.json
│   │
│   ├── plugin-walletconnect/          # @furlpay/hw-plugin-walletconnect
│   │   ├── src/
│   │   │   ├── WalletConnectPlugin.ts
│   │   │   ├── WalletConnectSigner.ts
│   │   │   └── ReownAppKitBridge.ts
│   │   └── package.json
│   │
│   ├── smart-account/                 # @furlpay/hw-smart-account
│   │   ├── src/
│   │   │   ├── SessionKeyManager.ts
│   │   │   ├── EIP7702Delegator.ts
│   │   │   ├── PaymasterClient.ts
│   │   │   └── UserOperationBuilder.ts
│   │   └── package.json
│   │
│   └── react-native-ui/              # @furlpay/hw-react-native-ui
│       ├── src/
│       │   ├── components/
│       │   │   ├── DeviceSelector.tsx
│       │   │   ├── SigningOverlay.tsx
│       │   │   ├── SimulationPreview.tsx
│       │   │   ├── PairingGuide.tsx
│       │   │   ├── ErrorBanner.tsx
│       │   │   └── SessionKeyManager.tsx
│       │   └── hooks/
│       │       ├── useHardwareWallet.ts
│       │       ├── useDeviceDiscovery.ts
│       │       ├── useSignTransaction.ts
│       │       └── useSessionKey.ts
│       └── package.json
│
├── turbo.json
├── package.json                       # Workspace root
└── tsconfig.base.json
```

## 14.3 SDK Usage Example

```typescript
import { HardwareWalletRegistry } from "@furlpay/hw-core";
import { LedgerPlugin } from "@furlpay/hw-plugin-ledger";
import { TangemPlugin } from "@furlpay/hw-plugin-tangem";
import { WalletConnectPlugin } from "@furlpay/hw-plugin-walletconnect";
import { SessionKeyManager } from "@furlpay/hw-smart-account";
import {
  DeviceSelector,
  SigningOverlay,
  SimulationPreview,
  useHardwareWallet,
  useSignTransaction,
} from "@furlpay/hw-react-native-ui";

// ═══════════════════════════════════════════════════════
// 1. App Initialization (once at startup)
// ═══════════════════════════════════════════════════════

const registry = new HardwareWalletRegistry();
registry.register(new LedgerPlugin());
registry.register(new TangemPlugin());
registry.register(new WalletConnectPlugin());

// ═══════════════════════════════════════════════════════
// 2. React Component Usage
// ═══════════════════════════════════════════════════════

function SendScreen() {
  const { wallet, signer, isConnected } = useHardwareWallet(registry);
  const { sign, simulation, status } = useSignTransaction(signer);

  const handleSend = async () => {
    const tx = {
      to: recipientAddress,
      value: parseEther("1.5"),
      chainId: 1,
      type: 2, // EIP-1559
    };

    // Simulate first → show preview → sign on hardware
    const result = await sign(tx, {
      derivationPath: "44'/60'/0'/0/0",
      simulate: true,         // Enable pre-signing simulation
      clearSigning: true,     // Attach ERC-7730 descriptors
    });

    // result.serializedSignedTx → broadcast
  };

  return (
    <>
      {!isConnected && <DeviceSelector registry={registry} />}
      {simulation && <SimulationPreview data={simulation} />}
      {status === "signing" && <SigningOverlay wallet={wallet} />}
    </>
  );
}

// ═══════════════════════════════════════════════════════
// 3. Session Keys (Advanced)
// ═══════════════════════════════════════════════════════

const sessionManager = new SessionKeyManager({
  signer,
  bundlerUrl: "https://bundler.furlpay.com",
  paymasterUrl: "https://paymaster.furlpay.com",
});

// One-time hardware authorization
await sessionManager.createSession({
  permissions: {
    maxSpendPerDay: parseEther("0.5"),
    allowedContracts: [LIFI_CONTRACT, UNISWAP_ROUTER],
    expiresIn: "24h",
  },
});

// Subsequent transactions — no hardware needed
await sessionManager.executeWithSessionKey({
  to: LIFI_CONTRACT,
  data: swapCalldata,
  value: 0n,
});
```

## 14.4 Publishing Strategy

| Aspect | Recommendation |
|:---|:---|
| **License** | Apache 2.0 (most permissive, matches Ledger SDK) |
| **Package Scope** | `@furlpay/hw-*` (scoped npm packages) |
| **Build System** | Turborepo monorepo with TypeScript strict mode |
| **Module Format** | ESM primary, CJS fallback |
| **Versioning** | Semantic Versioning (SemVer) |
| **CI/CD** | Changesets for automated version management |
| **Documentation** | TypeDoc auto-generated API docs + Storybook for UI components |
| **Testing** | Jest for unit tests, Detox for E2E hardware simulation |

---

# 15. Future-Proofing

## 15.1 EIP-7702 (Live on Mainnet — Pectra Fork, May 2025)

EIP-7702 is **fully active on Ethereum mainnet** and is the most important upgrade for hardware wallet UX:

- EOAs can temporarily delegate to smart contract code within a single transaction
- Hardware wallets leverage Smart Account features (batching, session keys, gas sponsorship)
- **No migration needed** — existing EOA addresses gain smart account features
- **Security**: Only delegate to audited, vetted smart contract implementations (maintain whitelist)
- Compatible with **ERC-6900** modular framework for managing session key modules

### FurlPay Implementation Priority
1. Maintain a vetted whitelist of delegation contracts
2. Hardware wallet signs authorization message pointing EOA to smart contract code
3. Session key modules handle frequent low-risk operations
4. Hardware re-authorization required for limit changes

## 15.2 Passkeys (WebAuthn / EIP-7212)

- **EIP-7212**: EVM precompile at address `0x0B` for native `secp256r1` (P-256) signature verification
- WebAuthn enables hardware authenticators (Face ID / Touch ID Secure Enclave, YubiKeys) to sign EVM payloads
- **PRF extension**: Passkeys can output deterministic keys for wallet derivation from biometric scan
- **Impact**: Could replace hardware wallets for low-to-medium value accounts

## 15.3 MPC Wallets

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Mobile SE    │     │ Cloud Server │     │ Backup/Recovery│
│ Key Share 1  │     │ Key Share 2  │     │ Key Share 3   │
└──────┬───────┘     └──────┬───────┘     └───────────────┘
       │                    │
       └──── 2-of-3 TSS ───┘
                │
                ▼
    Valid On-Chain ECDSA Signature
```

- Full private key **never exists anywhere** in unified form
- Hardware wallet can serve as one MPC node (Share 1)
- Combines physical hardware security with cloud recovery

## 15.4 ERC-4337 Account Abstraction

- **Hardware key as owner signer**: Signs `UserOperations` for Smart Contract Wallets (e.g., Safe)
- Enables key rotation without changing contract address
- Gas paid by Paymasters — hardware wallet users get gasless UX
- **2026 standard**: Bundler/paymaster infrastructure is now commodity (Pimlico, Alchemy, StackUp)

## 15.5 Session Keys

1. Hardware wallet signs a **scoped permission grant** for an ephemeral session key (stored in mobile SE)
2. Grant defines: spending limits, allowed contracts, expiration
3. Mobile app executes micro-transactions instantly using session key
4. When session expires → hardware key re-authorization required
5. **Result**: Web2-like UX without sacrificing root hardware security

## 15.6 Multi-Device Signing

- **2-of-3 Safe Multisig**:
  - Signer 1: Mobile Phone Secure Enclave (biometric)
  - Signer 2: Physical Hardware Wallet (Ledger/Trezor)
  - Signer 3: Offline Recovery Cloud Key (KMS)
- Daily tx: Signer 1 + 2. Lost hardware: Signer 1 + 3 for recovery.

## 15.7 Institutional Custody

- **MPC-as-a-Service**: Fireblocks, Fordefi, Cobo, BitGo APIs
- **HSM co-signing**: FIPS 140-2 Level 3 cloud HSMs
- **Policy engines**: Multi-approver workflows, timelocks, velocity limits

---

# 16. Implementation Roadmap

## 16.1 Wallet Scoring Matrix

| Wallet | User Adoption | SDK Quality | Security | Eng. Effort | Maintenance | **Weighted Score** |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Ledger** | 10 | 9 | 10 | 7 | 7 | **8.8** |
| **WalletConnect** | 9 | 8 | 6 | 9 | 8 | **8.1** |
| **Trezor** | 8 | 7 | 9 | 6 | 7 | **7.6** |
| **Tangem** | 7 | 7 | 8 | 5 | 7 | **6.9** |
| **OneKey** | 5 | 8 | 7 | 7 | 7 | **6.6** |
| **Keystone** | 5 | 7 | 9 | 7 | 7 | **6.5** |
| **SafePal** | 6 | 3 | 7 | 9 | 8 | **6.2** |
| **BitBox02** | 3 | 6 | 8 | 5 | 7 | **5.2** |
| **Coldcard** | 3 | 4 | 10 | 4 | 6 | **4.8** |

## 16.2 Recommended Roadmap

### 🔴 MVP (Month 1–3)

| # | Integration | Effort |
|:---|:---|:---|
| **1** | **Ledger BLE (Native DMK)** — Direct BLE with DMK SDK. EIP-1559 + clear signing + ERC-20. | **~4–6 weeks** |
| **2** | **WalletConnect v2 (Reown AppKit)** — Catch-all for 500+ wallets. | **~2–3 weeks** |
| **3** | **Transaction Simulation** — Integrate Tenderly/Blowfish API. Pre-signing safety. | **~2 weeks** |
| **4** | **ERC-7730 Descriptors** — Register clear signing for LI.FI, Uniswap, bridge contracts. | **~1 week** |
| **5** | **@furlpay/hw-core SDK** — Publish core interfaces + Ledger plugin + WC plugin. | **~2 weeks** |

### 🟡 Version 2 (Month 4–6)

| # | Integration | Effort |
|:---|:---|:---|
| **6** | **Tangem NFC (Native)** — TurboModule wrapping Swift/Kotlin SDKs. Sub-second signing. | **~4–6 weeks** |
| **7** | **Trezor Safe 7 BLE** — `@trezor/connect-mobile` + THP v2 protocol. | **~3–4 weeks** |
| **8** | **EIP-7702 Session Keys** — Hardware delegates to smart contract. Ephemeral session keys for daily use. | **~4 weeks** |
| **9** | **Gas Sponsorship** — Paymaster integration for gasless HW wallet UX. | **~2 weeks** |
| **10** | **Transaction Batching** — Multi-step operations in single hardware confirmation. | **~2 weeks** |

### 🟢 Version 3 (Month 7–12)

| # | Integration | Effort |
|:---|:---|:---|
| **11** | **Keystone QR** — Air-gapped QR signing with `@keystonehq/animated-qr`. | **~3–4 weeks** |
| **12** | **OneKey BLE** — `@onekeyfe/hd-ble-sdk`. Straightforward RN BLE. | **~2–3 weeks** |
| **13** | **Passkey Hybrid Security** — Tiered security model (biometric + hardware thresholds). | **~4 weeks** |
| **14** | **Social Recovery** — Safe multisig with hardware wallet as primary signer. | **~4 weeks** |
| **15** | **DeFi Position Manager** — View/manage staking, lending, LP with HW signing. | **~3 weeks** |

### 🔵 Version 4+ (Future)

| Integration | When |
|:---|:---|
| **Institutional Policy Engine** | Enterprise demand |
| **MPC Integration** | Enterprise/team wallets |
| **BitBox02 Nova BLE** | European market expansion |
| **Coldcard NFC** | If Bitcoin support added |

## 16.3 Final Recommendation Summary

```
╔═════════════════════════════════════════════════════════════════════════╗
║                    FurlPay Hardware Wallet Strategy                     ║
╠═════════════════════════════════════════════════════════════════════════╣
║                                                                        ║
║  MVP:  1. Ledger BLE (Native DMK)  →  50% of HW wallet market        ║
║        2. WalletConnect v2 (Reown) →  500+ wallets catch-all          ║
║        3. Transaction Simulation   →  Industry-standard safety        ║
║        4. ERC-7730 Clear Signing   →  Eliminate blind signing         ║
║        5. @furlpay/hw-core SDK     →  Open-source ecosystem           ║
║                                                                        ║
║  V2:   6. Tangem NFC (Native)      →  Best mobile UX, +102% growth   ║
║        7. Trezor Safe 7 BLE        →  Open-source community           ║
║        8. EIP-7702 Session Keys    →  Web2-like daily UX              ║
║        9. Gas Sponsorship          →  Gasless hardware wallet UX      ║
║       10. Transaction Batching     →  Multi-step in one confirm       ║
║                                                                        ║
║  V3:  11. Keystone QR              →  Maximum air-gap security        ║
║       12. OneKey BLE               →  Full open-source RN SDK         ║
║       13. Passkey Hybrid           →  Tiered security model           ║
║       14. Social Recovery          →  Solves "lost hardware" problem  ║
║       15. DeFi Position Manager    →  Active DeFi user retention      ║
║                                                                        ║
║  SDK: @furlpay/hw-* monorepo published to npm (Apache 2.0)           ║
║  Architecture: Plugin registry — zero domain changes per new wallet   ║
║  Security: Simulation + Clear Signing + Attestation on every tx       ║
║  Critical: Expo Dev Builds required (NOT Expo Go)                     ║
╚═════════════════════════════════════════════════════════════════════════╝
```

> [!IMPORTANT]
> **What makes this best-in-class vs. competitors:**
> 1. **Transaction simulation before hardware signing** — MetaMask does this, most mobile wallets don't
> 2. **EIP-7702 session keys** — No mobile hardware wallet app does this yet (early mover advantage)
> 3. **Tiered passkey + hardware security** — Novel approach to balancing convenience and security
> 4. **Published open-source SDK** — Becomes infrastructure for the ecosystem, not just a product
> 5. **Gas sponsorship** — Hardware wallet users get gasless UX (unique differentiator)

---

# Appendix A: Package Reference

## Essential NPM Packages for MVP

```json
{
  "dependencies": {
    "@ledgerhq/device-management-kit": "^0.6.x",
    "@ledgerhq/device-transport-kit-react-native-ble": "^0.6.x",
    "@ledgerhq/device-signer-kit-ethereum": "^0.6.x",
    "@reown/appkit-react-native": "^1.x",
    "@walletconnect/react-native-compat": "^2.x",
    "react-native-ble-plx": "^3.x",
    "react-native-get-random-values": "^1.x",
    "@react-native-async-storage/async-storage": "^2.x",
    "@react-native-community/netinfo": "^11.x",
    "react-native-safe-area-context": "^5.x",
    "expo-application": "^6.x",
    "buffer": "^6.x",
    "viem": "^2.x"
  }
}
```

## V2 Additional Packages

```json
{
  "dependencies": {
    "tangem-sdk-react-native": "latest",
    "@trezor/connect-mobile": "^9.6.x",
    "@keystonehq/keystone-sdk": "latest",
    "@keystonehq/animated-qr": "latest",
    "@onekeyfe/hd-ble-sdk": "latest",
    "@onekeyfe/hd-transport-react-native": "latest"
  }
}
```

## Expo Config (app.json)

```json
{
  "expo": {
    "plugins": [
      [
        "react-native-ble-plx",
        {
          "isBackgroundEnabled": false,
          "modes": ["peripheral", "central"],
          "bluetoothAlwaysPermission": "FurlPay uses Bluetooth to connect to your hardware wallet"
        }
      ]
    ],
    "ios": {
      "infoPlist": {
        "NSBluetoothAlwaysUsageDescription": "Connect to your Ledger hardware wallet",
        "NSBluetoothPeripheralUsageDescription": "Connect to your hardware wallet",
        "NFCReaderUsageDescription": "Tap your Tangem card to sign transactions",
        "com.apple.developer.nfc.readersession.iso7816.select-identifiers": [
          "A0000008120101"
        ],
        "NSCameraUsageDescription": "Scan QR codes from your Keystone hardware wallet"
      }
    },
    "android": {
      "permissions": [
        "BLUETOOTH_SCAN",
        "BLUETOOTH_CONNECT",
        "ACCESS_FINE_LOCATION",
        "NFC",
        "CAMERA"
      ]
    }
  }
}
```

## Appendix B: Key Official Documentation Links

| Resource | URL |
|:---|:---|
| Ledger DMK Docs | developers.ledger.com |
| Ledger DMK GitHub | github.com/LedgerHQ/device-sdk-ts |
| ERC-7730 Registry | clearsigning.org |
| Trezor Connect | github.com/trezor/trezor-suite |
| Trezor THP v2 | github.com/trezor/trezor-firmware |
| Tangem iOS SDK | github.com/tangem/tangem-sdk-ios |
| Tangem Android SDK | github.com/tangem/tangem-sdk-android |
| Tangem RN SDK | github.com/tangem/tangem-sdk-react-native |
| Tangem Dev Portal | developers.tangem.com |
| Keystone SDK | keyst.one/developers |
| Keystone Animated QR | npmjs.com/package/@keystonehq/animated-qr |
| OneKey HW SDK | github.com/OneKeyHQ/hardware-js-sdk |
| OneKey Dev Portal | developer.onekey.so |
| WalletConnect (Reown) | docs.reown.com |
| Reown Dashboard | dashboard.reown.com |
| BitBox02 JS API | github.com/digitalbitbox/bitbox02-api-js |
| Openfort (AA reference) | openfort.xyz |
| Trust Wallet Core | github.com/trustwallet/wallet-core |

---

*Report compiled August 2026 from vendor documentation, GitHub repositories, npm registries, developer portals, market reports, and security research. All package names, version constraints, SDK architectures, and feature recommendations verified against current developer specifications and live web research.*
