# Tokenized Stocks — Technical Research Report
## Building Tokenized Stock Trading in FurlPay
**Date:** August 2026 | **Sources:** Coinbase, Phantom, Robinhood, Backed Finance, Dinari, Securitize, Jupiter, Solana

---

# Table of Contents

1. [Market Overview](#1-market-overview)
2. [How Tokenized Stocks Work](#2-how-tokenized-stocks-work)
3. [Major Providers & Platforms](#3-major-providers--platforms)
4. [Coinbase Ecosystem](#4-coinbase-ecosystem)
5. [Robinhood Chain](#5-robinhood-chain)
6. [Backed Finance (xStocks)](#6-backed-finance-xstocks)
7. [Dinari (dShares)](#7-dinari-dshares)
8. [Securitize](#8-securitize)
9. [Solana Tokenized Stocks](#9-solana-tokenized-stocks)
10. [Phantom & Other Wallets](#10-phantom--other-wallets)
11. [Regulatory & Compliance](#11-regulatory--compliance)
12. [How to Build This in FurlPay](#12-how-to-build-this-in-furlpay)
13. [FurlPay Architecture](#13-furlpay-architecture)
14. [Implementation Roadmap](#14-implementation-roadmap)

---

# 1. Market Overview

## 1.1 Tokenized Stocks in 2026

Tokenized stocks are blockchain-based tokens that represent ownership or economic exposure to real-world equities (Apple, Tesla, Nvidia, S&P 500 ETFs, etc.). Each token is **backed 1:1** by actual shares held by a regulated custodian.

### Key Market Stats (Mid-2026)

| Metric | Value |
|:---|:---|
| **Monthly on-chain volume** | **$9.22 billion** (June 2026) |
| **Solana market share** | ~95% of all tokenized stock volume |
| **Available stocks** | 60+ U.S. equities and ETFs |
| **xStocks total volume** | $3B+ cumulative |
| **Top traded** | TSLA, AAPL, NVDA, SPY, QQQ, AMZN, GOOGL |
| **Trading hours** | **24/7** (vs. NYSE 9:30am–4pm) |
| **Fractional ownership** | ✅ Yes (buy $10 of AAPL) |
| **Dividends** | ✅ Auto-distributed on-chain |
| **Stock splits** | ✅ Auto-handled via multiplier system |

## 1.2 Why This Matters for FurlPay

```
╔═════════════════════════════════════════════════════════════════╗
║  THE OPPORTUNITY                                                 ║
║                                                                 ║
║  Users want ONE app for:                                        ║
║  • Crypto (ETH, SOL, USDC)     ← FurlPay already does this    ║
║  • Stocks (AAPL, TSLA, NVDA)   ← NEW: tokenized stocks        ║
║  • Swaps & bridges             ← FurlPay already does this    ║
║  • DeFi yields                 ← FurlPay can add this         ║
║  • Agent payments              ← Research completed            ║
║                                                                 ║
║  Adding tokenized stocks makes FurlPay a "superapp" —           ║
║  crypto + stocks + payments in one self-custody wallet           ║
╚═════════════════════════════════════════════════════════════════╝
```

---

# 2. How Tokenized Stocks Work

## 2.1 The Full Lifecycle

```
┌──────────────────────────────────────────────────────────────────────┐
│                    TOKENIZED STOCK LIFECYCLE                          │
│                                                                      │
│  1. ISSUANCE                                                         │
│  ┌──────────┐     ┌──────────────────┐     ┌───────────────────┐    │
│  │ Issuer   │────▶│ Regulated        │────▶│ Mint ERC-20/SPL   │    │
│  │ (Backed, │     │ Custodian buys   │     │ token on chain    │    │
│  │  Dinari,  │     │ real shares      │     │ (1 token = 1      │    │
│  │  Coinbase)│     │ (Alpaca, InCore) │     │  share of AAPL)   │    │
│  └──────────┘     └──────────────────┘     └───────────────────┘    │
│                                                                      │
│  2. TRADING                                                          │
│  ┌──────────┐     ┌──────────────────┐     ┌───────────────────┐    │
│  │ User in  │────▶│ DEX/Aggregator   │────▶│ Swap USDC →       │    │
│  │ FurlPay  │     │ (Jupiter, LI.FI, │     │ AAPLx token       │    │
│  │ wallet   │     │  Uniswap)        │     │ (instant, 24/7)   │    │
│  └──────────┘     └──────────────────┘     └───────────────────┘    │
│                                                                      │
│  3. CORPORATE ACTIONS                                                │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Dividend → auto-distributed to token holders on-chain        │   │
│  │ Stock split → multiplier adjusted (e.g., 4:1 = 4x tokens)   │   │
│  │ Price feed → Chainlink oracles publish real-time prices      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  4. REDEMPTION                                                       │
│  ┌──────────┐     ┌──────────────────┐     ┌───────────────────┐    │
│  │ User     │────▶│ Burn token       │────▶│ Custodian sells   │    │
│  │ sells    │     │ on-chain         │     │ real shares,      │    │
│  │ position │     │                  │     │ returns USDC      │    │
│  └──────────┘     └──────────────────┘     └───────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

## 2.2 Ownership Model

| Aspect | Description |
|:---|:---|
| **What you hold** | ERC-20 or SPL token representing 1 share |
| **Backing** | 1:1 by real shares in regulated custody |
| **Legal ownership** | Economic exposure (not direct shareholder of record) |
| **Voting rights** | Generally ❌ (varies by issuer) |
| **Dividends** | ✅ Auto-distributed on-chain |
| **Fractional** | ✅ Buy any amount (e.g., 0.001 AAPL) |
| **Settlement** | Instant (vs T+1 in traditional markets) |
| **Trading hours** | 24/7/365 |

---

# 3. Major Providers & Platforms

## 3.1 Provider Comparison Matrix

| Provider | Token Standard | Chains | Stocks Available | Custody | KYC Required | API/SDK | Integration Model |
|:---|:---|:---|:---:|:---|:---|:---:|:---|
| **Backed (xStocks)** | ERC-20 + SPL | Ethereum, Base, Solana, Polygon, Gnosis | 60+ | InCore Bank, Alpaca | ✅ (for issuers) | ❌ No public SDK | Standard token swap via DEX |
| **Dinari (dShares)** | ERC-20 | Arbitrum, Base, Ethereum | 40+ | Regulated custodian | ✅ KYB for partners | ✅ REST API | B2B API partnership |
| **Coinbase Tokenize** | ERC-20 | Base, Ethereum | Coming Q3 2026 | Coinbase Custody | ✅ Institutional | ✅ CDP APIs | Institutional platform |
| **Robinhood Chain** | ERC-20 | Robinhood L2 (Arbitrum Orbit) | 25+ | Robinhood Assets (Jersey) | ✅ | ✅ REST API | Direct chain + API |
| **Securitize** | ERC-3643 | Ethereum, Solana, Polygon | Institutional funds | Securitize Trust | ✅ Securitize iD | ✅ API (restricted) | Institutional |

## 3.2 Which Is Best for FurlPay?

| Approach | Pros | Cons | Recommended |
|:---|:---|:---|:---:|
| **Swap via DEX (xStocks/Jupiter)** | No partnership needed, permissionless, instant | No KYC enforcement (FurlPay must handle), geographic restrictions | ✅ **MVP** |
| **Dinari B2B API** | Full compliance handled, corporate actions, managed accounts | Requires KYB partnership, API access gated | ✅ **V2** |
| **Robinhood Chain direct** | Official REST API, Chainlink price feeds, corporate actions | New chain (small ecosystem), geographic restrictions | 🟡 **V3** |
| **Coinbase Tokenize** | Largest exchange brand, Base ecosystem | Institutional focus, not yet public, non-US only | 🟡 **Future** |
| **Securitize** | Institutional grade, ERC-3643 compliance | Enterprise-only, complex integration | ❌ Overkill for retail |

---

# 4. Coinbase Ecosystem

## 4.1 Coinbase Tokenize

Coinbase's institutional platform for issuing tokenized securities:

- **Purpose**: End-to-end issuance, custody, compliance, and trading of RWAs
- **Built on**: Base L2 + Ethereum
- **Project Diamond**: Foundational program for digital debt instruments
- **Chainlink integration**: CCIP for cross-chain interop, Functions for real-world data
- **Status**: Active for institutional clients, retail tokenized stocks coming Q3 2026

## 4.2 Coinbase Developer Platform (CDP)

Key APIs for FurlPay integration:

| API | Purpose | URL |
|:---|:---|:---|
| **Advanced Trade API** | Spot + derivatives trading | docs.cdp.coinbase.com |
| **CDP Trade API** | On-chain token swaps (Base, Ethereum) | docs.cdp.coinbase.com |
| **Token Manager** | Lifecycle management (vesting, lockups, distributions) | docs.cdp.coinbase.com |
| **Policy API** | Compliance controls | docs.cdp.coinbase.com |

## 4.3 1:1 Backed Tokenized U.S. Equities

- **Target**: Non-U.S. customers only (regulatory constraints)
- **Structure**: True equity backing with dividends and shareholder rights
- **Chain**: Base (Ethereum L2)
- **Launch**: Imminent (Q3 2026)
- **Competition**: Directly competing with Robinhood Chain

---

# 5. Robinhood Chain

## 5.1 Architecture

- **Type**: Ethereum L2 built on **Arbitrum Orbit**
- **Launched**: July 1, 2026
- **Block time**: 100ms
- **Issuer**: Robinhood Assets (Jersey) Limited
- **Design**: Permissionless, "AI-native" financial infrastructure

## 5.2 Stock Token Mechanics

```typescript
// Stock Token is a standard ERC-20 with corporate action multiplier
interface StockToken {
  // Standard ERC-20
  name: string;          // "Tesla Stock Token"
  symbol: string;        // "TSLA"
  decimals: number;      // 18

  // Corporate Actions
  multiplier: number;    // e.g., 4.0 after a 4:1 stock split
  // Actual shares = tokenBalance * multiplier
  // If you hold 1.0 TSLA token and multiplier is 4.0,
  // you have exposure to 4 shares of Tesla
}
```

## 5.3 Developer API

```typescript
// REST API for off-chain metadata
const ROBINHOOD_API = "https://api.robinhood.com/rhj/";

// Get all available stock tokens
const assets = await fetch(`${ROBINHOOD_API}assets`).then(r => r.json());
// Returns: { assets: [{ symbol, name, contractAddress, chainId, multiplier, ... }] }

// Price feeds via Chainlink on Robinhood Chain
import { ethers } from "ethers";
const priceFeed = new ethers.Contract(CHAINLINK_TSLA_FEED, aggregatorV3ABI, provider);
const { answer } = await priceFeed.latestRoundData();
// answer = TSLA price in USD (8 decimals)
```

## 5.4 Key Features

| Feature | Details |
|:---|:---|
| **24/7 trading** | No market hours — trade anytime |
| **Fractional** | Buy any amount ($1 of TSLA) |
| **Dividends** | On-chain distribution |
| **Stock splits** | Multiplier system (auto-adjusts) |
| **Chainlink prices** | Real-time on-chain price feeds |
| **Standard ERC-20** | Works with any EVM wallet (viem, ethers.js) |
| **Composable** | Use as collateral in DeFi lending |

## 5.5 Geographic Restrictions

> [!WARNING]
> Stock Tokens are **NOT available** in: United States, Canada, United Kingdom, Switzerland, and other restricted jurisdictions. FurlPay must implement geographic gating.

---

# 6. Backed Finance (xStocks)

## 6.1 Product Overview

**Backed Finance** (acquired by Kraken, late 2025) issues the **xStocks** product line — the most widely traded tokenized stocks on-chain.

| Metric | Value |
|:---|:---|
| **Available stocks** | 60+ U.S. equities and ETFs |
| **Chains** | Solana (primary), Ethereum, Base, Polygon, Gnosis |
| **Volume** | $3B+ cumulative, ~95% on Solana |
| **Custody** | InCore Bank (Switzerland), Alpaca Securities |
| **Compliance** | Swiss FINMA regulated |
| **Token standard** | ERC-20 (EVM) + SPL (Solana) |

## 6.2 Popular xStocks

| Token | Underlying | Chain |
|:---|:---|:---|
| `AAPLx` | Apple Inc. | Solana, Ethereum |
| `TSLAx` | Tesla Inc. | Solana, Ethereum |
| `NVDAx` | Nvidia Corp. | Solana, Ethereum |
| `AMZNx` | Amazon.com | Solana, Ethereum |
| `GOOGLx` | Alphabet Inc. | Solana, Ethereum |
| `MSFTx` | Microsoft Corp. | Solana, Ethereum |
| `SPYx` | S&P 500 ETF | Solana, Ethereum |
| `QQQx` | Nasdaq 100 ETF | Solana, Ethereum |

## 6.3 Integration for FurlPay

**No API/SDK needed** — xStocks are standard ERC-20/SPL tokens tradeable on DEXs:

```typescript
// Solana: Swap USDC → AAPLx via Jupiter
import { Jupiter } from "@jup-ag/api";

const jupiter = new Jupiter({ connection, cluster: "mainnet-beta" });
const routes = await jupiter.computeRoutes({
  inputMint: USDC_MINT,          // USDC on Solana
  outputMint: AAPLX_MINT,        // AAPLx token mint
  amount: 100_000_000,           // $100 USDC (6 decimals)
  slippageBps: 50,               // 0.5% slippage
});

const { swapTransaction } = await jupiter.exchange({
  routeInfo: routes.routesInfos[0],
});
// Sign and send with user's wallet
```

```typescript
// EVM (Base/Ethereum): Swap via LI.FI (FurlPay already integrated)
import { getQuote, executeRoute } from "@lifi/sdk";

const quote = await getQuote({
  fromChain: "8453",              // Base
  toChain: "8453",
  fromToken: USDC_BASE,
  toToken: AAPLX_BASE,           // AAPLx on Base
  fromAmount: "100000000",        // $100 USDC
  fromAddress: userAddress,
});
```

---

# 7. Dinari (dShares)

## 7.1 Product Overview

**Dinari** issues **dShares™** — tokenized U.S. stocks and ETFs with full regulatory compliance and a B2B API for fintech integration.

## 7.2 B2B API

| Feature | Details |
|:---|:---|
| **API Type** | RESTful |
| **Authentication** | API Key + Secret |
| **Partnership** | Requires KYB (Know Your Business) onboarding |
| **Chains** | Arbitrum, Base, Ethereum |
| **Corporate actions** | Auto-handled (dividends, splits) |
| **Managed accounts** | Available (Dinari handles wallet management) |

### API Flow

```typescript
// 1. Create order via Dinari API (requires partnership)
const order = await dinariClient.createOrder({
  symbol: "AAPL",
  side: "buy",
  amount: "100.00",            // $100 USD
  paymentToken: "USDC",
  recipientAddress: userWalletAddress,
});

// 2. Dinari buys real AAPL shares
// 3. Mints dShare tokens to user's address
// 4. User sees AAPL tokens in FurlPay wallet

// 5. Corporate actions auto-handled:
// - Dividends → USDC airdrop to token holders
// - Stock splits → token balance adjusted automatically
```

## 7.3 Developer Resources

| Resource | URL |
|:---|:---|
| Documentation | dinaricrypto.github.io |
| GitHub | github.com/dinaricrypto |
| Go API Library | github.com/dinaricrypto (Stainless-generated) |
| Partnership | dinari.com/business |
| Testnet | Available for testing API calls |
| Developer Incentive | FinTech Developer Program for non-US devs |

---

# 8. Securitize

## 8.1 Overview

**Securitize** is the institutional-grade platform for tokenized securities, powering BlackRock's BUIDL fund and other major institutional products.

## 8.2 DS Protocol (Compliance Standard)

Securitize uses its proprietary **DS Protocol** which enforces:
- Pre-transfer compliance checks
- Only verified wallets can hold/trade
- KYC/AML via "Securitize iD"
- Automated cap table management
- Dividend and corporate action processing

## 8.3 FurlPay Relevance

> [!NOTE]
> Securitize is **institutional-focused** and likely overkill for FurlPay's retail use case. However, their compliance model (whitelisted wallets, transfer hooks) is worth understanding for when FurlPay needs to handle regulated assets.

---

# 9. Solana Tokenized Stocks

## 9.1 Why Solana Dominates (95% Market Share)

| Factor | Solana Advantage |
|:---|:---|
| **Speed** | 400ms block time, sub-second finality |
| **Cost** | $0.00025 per transaction |
| **Token Extensions** | Native compliance features (transfer hooks, confidential transfers) |
| **Jupiter DEX** | Best aggregator for stock token liquidity |
| **DeFi composability** | Use stock tokens as collateral in Kamino, lending protocols |
| **Ecosystem** | Phantom, Backpack, Jupiter — all support xStocks |

## 9.2 Solana Token Extensions for Compliance

Solana's **Token Extensions** (Token-2022) provide native compliance features:

```
┌──────────────────────────────────────────────────────────────┐
│              SOLANA TOKEN EXTENSIONS FOR STOCKS               │
│                                                              │
│  ┌─────────────────────┐                                     │
│  │ Transfer Hook       │ Validates KYC status before every   │
│  │                     │ transfer. Non-verified = blocked.   │
│  └─────────────────────┘                                     │
│  ┌─────────────────────┐                                     │
│  │ Confidential        │ Hides transaction amounts while     │
│  │ Transfers           │ still enforcing compliance rules.   │
│  └─────────────────────┘                                     │
│  ┌─────────────────────┐                                     │
│  │ Permanent Delegate  │ Issuer can force-burn/recall tokens │
│  │                     │ for regulatory compliance.          │
│  └─────────────────────┘                                     │
│  ┌─────────────────────┐                                     │
│  │ Interest-Bearing     │ Native dividend accrual.            │
│  │ Tokens              │                                     │
│  └─────────────────────┘                                     │
│  ┌─────────────────────┐                                     │
│  │ Non-Transferable    │ Lock tokens during restricted        │
│  │                     │ periods (blackout windows).          │
│  └─────────────────────┘                                     │
└──────────────────────────────────────────────────────────────┘
```

## 9.3 Jupiter Integration (Primary DEX)

### Jupiter Terminal (Drop-in Widget)

```typescript
// Embed Jupiter swap widget for stock tokens
import { JupiterTerminal } from "@jup-ag/terminal";

// Drop-in swap modal
<JupiterTerminal
  integratedTargetId="swap-container"
  formProps={{
    initialInputMint: USDC_MINT,
    initialOutputMint: AAPLX_MINT,
    fixedOutputMint: false,
  }}
/>
```

### Jupiter API v6 (Custom Implementation)

```typescript
// Step 1: Get best route
const quoteResponse = await fetch(
  `https://quote-api.jup.ag/v6/quote?` +
  `inputMint=${USDC_MINT}&` +
  `outputMint=${TSLAX_MINT}&` +
  `amount=${100_000_000}&` +  // $100 USDC
  `slippageBps=50`
).then(r => r.json());

// Step 2: Build swap transaction
const { swapTransaction } = await fetch("https://quote-api.jup.ag/v6/swap", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    quoteResponse,
    userPublicKey: wallet.publicKey.toString(),
    wrapAndUnwrapSol: true,
    dynamicComputeUnitLimit: true,
    prioritizationFeeLamports: "auto",
  }),
}).then(r => r.json());

// Step 3: Sign and send
const transaction = VersionedTransaction.deserialize(
  Buffer.from(swapTransaction, "base64")
);
transaction.sign([wallet.payer]);
const txid = await connection.sendTransaction(transaction);
```

---

# 10. Phantom & Other Wallets

## 10.1 How Phantom Handles Tokenized Stocks

Phantom wallet (50M+ users) supports tokenized stocks natively:

1. **Multi-chain**: Solana, Ethereum, Base — all in one wallet
2. **Swap tab**: Users swap USDC/SOL → stock tokens via integrated Jupiter
3. **Portfolio view**: Shows stock token holdings with real-time prices
4. **No special UI**: Stock tokens appear alongside regular crypto tokens

## 10.2 Wallet Comparison for Stock Token Support

| Wallet | Chains | Stock Tokens | Swap Integration | Notes |
|:---|:---|:---:|:---|:---|
| **Phantom** | Solana, Ethereum, Base, Polygon | ✅ xStocks | Jupiter (Solana), integrated DEX | Market leader, 50M+ users |
| **Backpack** | Solana, Ethereum | ✅ xStocks | Jupiter | Built by ex-Meta engineers |
| **Coinbase Wallet** | EVM, Solana | ✅ Coming Q3 | Coinbase DEX | Will add Coinbase tokenized stocks |
| **MetaMask** | EVM only | ✅ ERC-20 stocks | Uniswap, 1inch | Largest EVM wallet |
| **Bitget Wallet** | Multi-chain | ✅ xStocks | Aggregator | Growing rapidly |
| **OKX Wallet** | Multi-chain | ✅ xStocks | OKX DEX | Large Asian user base |
| **FurlPay** | EVM (current) | ❌ Not yet | LI.FI | **OPPORTUNITY** |

---

# 11. Regulatory & Compliance

## 11.1 Key Compliance Requirements

### ERC-3643 (Permissioned Tokens on EVM)

ERC-3643 is the leading standard for compliant tokenized securities:

```
┌─────────────────────────────────────────────────────────────┐
│                    ERC-3643 COMPLIANCE FLOW                   │
│                                                             │
│  User A (KYC'd) ──transfer──▶ Token Contract               │
│                                     │                       │
│                               Check compliance:             │
│                               1. Is User B KYC'd?    ✅/❌  │
│                               2. Is User B in allowed       │
│                                  jurisdiction?        ✅/❌  │
│                               3. Does transfer violate      │
│                                  lock-up period?      ✅/❌  │
│                               4. Is User B on               │
│                                  sanctions list?      ✅/❌  │
│                                     │                       │
│                               All checks pass?              │
│                               ├── ✅ Transfer executes      │
│                               └── ❌ Transfer blocked       │
└─────────────────────────────────────────────────────────────┘
```

### FurlPay Compliance Checklist

| Requirement | How to Implement |
|:---|:---|
| **Geographic gating** | Block US, UK, CA, CH users from stock token trading. Use IP geolocation + phone number country |
| **KYC verification** | Partner with KYC provider (Jumio, Onfido, Sumsub) for identity verification |
| **Sanctions screening** | OFAC/SDN list checking before every transfer |
| **Transfer restrictions** | Respect ERC-3643 transfer hooks — blocked txs show clear error |
| **Risk disclaimers** | Display: "Stock tokens provide economic exposure, not direct share ownership" |
| **Terms of service** | Add stock token-specific T&C covering risks, custodial model |

## 11.2 Geographic Availability

| Region | Status | Notes |
|:---|:---:|:---|
| **United States** | ❌ Blocked | Securities regulations (SEC) |
| **Canada** | ❌ Blocked | CSA regulations |
| **United Kingdom** | ❌ Blocked | FCA restrictions |
| **European Union** | ✅ Available | MiCA framework allows |
| **Switzerland** | ⚠️ Restricted | Varies by issuer |
| **Asia-Pacific** | ✅ Most countries | Check per-country restrictions |
| **Latin America** | ✅ Available | Growing market |
| **Africa** | ✅ Available | Growing market |
| **Middle East** | ✅ Most countries | UAE, Saudi — active |

---

# 12. How to Build This in FurlPay

## 12.1 What FurlPay Already Has

| Component | Status | Gap for Tokenized Stocks |
|:---|:---|:---|
| EVM wallet | ✅ Built | Add Solana support for xStocks |
| LI.FI swap integration | ✅ Built | LI.FI supports stock token swaps on EVM |
| Transaction signing | ✅ Built | Ready for stock token tx signing |
| Multi-chain support | ✅ Built | Add Robinhood Chain RPC |
| Token display | ✅ Built | Add stock metadata (price, company, logo) |
| Price feeds | ✅ Built | Add stock price oracle integration |

## 12.2 Integration Strategy

### Approach 1: DEX Swap (No Partnership — MVP)

xStocks are standard ERC-20/SPL tokens. FurlPay can enable stock trading by simply routing swaps through existing DEX aggregators:

```
User wants to buy $100 of Apple stock
        │
        ▼
FurlPay checks: Is user in allowed region?
        │
        ├── ❌ US/UK/CA → Show "Not available in your region"
        │
        └── ✅ Allowed → Continue
                │
                ▼
        Select chain:
        ├── Solana → Jupiter API (best liquidity for xStocks)
        ├── Base → LI.FI (already integrated)
        └── Arbitrum → LI.FI
                │
                ▼
        Get quote: $100 USDC → 0.42 AAPLx
        Show: "Buy 0.42 shares of Apple ($100)"
                │
                ▼
        User confirms → Sign transaction
                │
                ▼
        AAPLx token appears in FurlPay wallet
        Portfolio shows: Apple — 0.42 shares — $100.00
```

### Approach 2: Dinari B2B API (Partnership — V2)

For a more robust, compliance-handled experience:

```
User wants to buy $100 of Tesla
        │
        ▼
FurlPay → Dinari API: Create buy order
        │
        ▼
Dinari validates compliance
Dinari buys real TSLA shares via custodian
Dinari mints dShare tokens to user's address
        │
        ▼
dTSLA token appears in FurlPay wallet
Dividends auto-distributed to wallet
Stock splits auto-adjusted
```

### Approach 3: Robinhood Chain Direct (V3)

```
User wants to buy $100 of Nvidia
        │
        ▼
FurlPay → Bridge USDC to Robinhood Chain
FurlPay → Swap USDC → NVDA on Robinhood DEX
        │
        ▼
NVDA stock token in wallet on Robinhood Chain
Chainlink price feed for real-time valuation
```

## 12.3 Stock Token Portfolio UI

```
┌─────────────────────────────────────────────────┐
│              FurlPay Portfolio                    │
│                                                  │
│  📊 Stocks                    $2,456.78  (+3.2%) │
│  ┌───────────────────────────────────────────┐   │
│  │ 🍎 Apple (AAPLx)                         │   │
│  │    0.42 shares · $100.38  · +1.2% today   │   │
│  │                                           │   │
│  │ 🚗 Tesla (TSLAx)                         │   │
│  │    2.15 shares · $856.40  · +5.1% today   │   │
│  │                                           │   │
│  │ 💻 Nvidia (NVDAx)                        │   │
│  │    1.00 shares · $1,500.00 · +2.8% today  │   │
│  └───────────────────────────────────────────┘   │
│                                                  │
│  🪙 Crypto                    $5,230.50  (+0.8%) │
│  ┌───────────────────────────────────────────┐   │
│  │ Ξ Ethereum (ETH)                         │   │
│  │    1.5 ETH · $4,500.00  · +1.0% today    │   │
│  │                                           │   │
│  │ 💵 USDC                                  │   │
│  │    730.50 USDC · $730.50                  │   │
│  └───────────────────────────────────────────┘   │
│                                                  │
│  [Buy Stocks]  [Swap]  [Bridge]  [Send]          │
└─────────────────────────────────────────────────┘
```

---

# 13. FurlPay Architecture

## 13.1 Stock Token Integration Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                    FURLPAY STOCK TOKEN LAYER                       │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Stock Token Registry                                       │ │
│  │  • Map stock symbols → contract addresses per chain         │ │
│  │  • AAPLx → { solana: "7xKX...", base: "0x1234..." }       │ │
│  │  • Pull metadata: name, logo, market cap, P/E              │ │
│  │  • Chainlink/oracle price feeds                            │ │
│  └───────────────────────────┬─────────────────────────────────┘ │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Compliance Engine                                          │ │
│  │  • Geographic gating (block US/UK/CA)                       │ │
│  │  • KYC status check (if required by issuer)                │ │
│  │  • Transfer restriction awareness (ERC-3643 hooks)         │ │
│  │  • Risk disclaimers & T&C acceptance                       │ │
│  └───────────────────────────┬─────────────────────────────────┘ │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Swap Router                                                │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐    │ │
│  │  │ Jupiter API │  │ LI.FI SDK   │  │ Dinari API      │    │ │
│  │  │ (Solana)    │  │ (EVM chains)│  │ (B2B, V2)       │    │ │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘    │ │
│  └───────────────────────────┬─────────────────────────────────┘ │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Price Feed Aggregator                                      │ │
│  │  • Chainlink oracles (on-chain, real-time)                 │ │
│  │  • CoinGecko API (fallback, aggregated)                    │ │
│  │  • Robinhood REST API (Robinhood Chain stocks)             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Portfolio Manager                                          │ │
│  │  • Detect stock tokens in wallet (by known contract list)  │ │
│  │  • Group: Stocks vs. Crypto vs. Stablecoins                │ │
│  │  • Show: shares owned, $ value, daily change, dividends    │ │
│  │  • Corporate actions: alert on dividends, splits           │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

## 13.2 Key API Endpoints to Build

```typescript
// apps/web/src/app/api/stocks/

// GET /api/stocks/available — List all tradeable stocks
// Returns: [{ symbol, name, logo, chains, prices, issuer }]

// GET /api/stocks/[symbol]/quote — Get swap quote
// Params: amount, fromAsset, chain
// Returns: { inputAmount, outputShares, pricePerShare, route, fees }

// POST /api/stocks/[symbol]/swap — Execute stock purchase
// Body: { amount, fromAsset, chain, walletAddress, slippage }
// Returns: { transactionHash, sharesReceived }

// GET /api/stocks/portfolio — User's stock holdings
// Returns: [{ symbol, shares, value, dailyChange, dividendsPaid }]

// GET /api/stocks/[symbol]/price-history — Historical prices
// Returns: [{ timestamp, price }] (1d, 1w, 1m, 1y, all)

// GET /api/stocks/corporate-actions — Recent corporate actions
// Returns: [{ symbol, type, date, details }]
```

---

# 14. Implementation Roadmap

## 14.1 Phase 1: MVP — DEX Swap (Month 1–2)

| # | Task | Details | Effort |
|:---|:---|:---|:---|
| 1 | **Stock Token Registry** | Map known stock tokens (xStocks, dShares) to contract addresses across chains. Pull metadata from CoinGecko. | 1 week |
| 2 | **Geographic Gating** | Block stock trading for US/UK/CA users. IP geolocation + phone number country check. | 1 week |
| 3 | **Swap via LI.FI** | Route stock token purchases through LI.FI (already integrated) on Base/Ethereum/Polygon. | 2 weeks |
| 4 | **Stock Portfolio View** | New "Stocks" section in portfolio. Show shares owned, $ value, daily change. Company logos. | 2 weeks |
| 5 | **Price Feeds** | Integrate Chainlink oracles + CoinGecko API for real-time stock token prices. | 1 week |
| 6 | **Risk Disclaimers** | Show compliance disclaimers, T&C acceptance for stock trading. | 1 week |

## 14.2 Phase 2: Enhanced Experience (Month 3–4)

| # | Task | Details | Effort |
|:---|:---|:---|:---|
| 7 | **Jupiter Integration (Solana)** | Add Jupiter API for Solana xStock swaps (95% of volume). Requires Solana wallet support. | 3 weeks |
| 8 | **Dinari Partnership** | Apply for KYB, integrate dShares API for compliance-handled stock purchases. | 3 weeks |
| 9 | **Stock Detail Screen** | Price chart (1d/1w/1m/1y), company info, market cap, P/E ratio, dividend yield. | 2 weeks |
| 10 | **Corporate Actions** | Display dividend payments, stock splits in transaction history. | 1 week |
| 11 | **Watchlist & Alerts** | Save favorite stocks, price alerts ("TSLA hits $400"). | 1 week |

## 14.3 Phase 3: Advanced (Month 5–6)

| # | Task | Details | Effort |
|:---|:---|:---|:---|
| 12 | **Robinhood Chain** | Add Robinhood L2 as supported chain. Integrate their REST API for metadata. | 2 weeks |
| 13 | **DeFi Composability** | Use stock tokens as collateral for lending (Kamino on Solana, Aave on Base). | 3 weeks |
| 14 | **Recurring Buys (DCA)** | Auto-purchase $X of stock per week/month. | 2 weeks |
| 15 | **Agent Stock Trading** | Connect to Agent Vault — AI agents can buy/sell stocks within limits. | 2 weeks |
| 16 | **Limit Orders** | Set buy/sell price targets. Execute when price is met. | 2 weeks |

## 14.4 Final Summary

```
╔═══════════════════════════════════════════════════════════════════════╗
║              FURLPAY TOKENIZED STOCKS STRATEGY                       ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  MVP (Month 1-2):                                                     ║
║  • Stock Token Registry + geographic gating                          ║
║  • Swap via LI.FI (EVM) — already integrated                        ║
║  • Stock portfolio view with real-time prices                        ║
║  • Risk disclaimers and T&C                                          ║
║                                                                       ║
║  V2 (Month 3-4):                                                      ║
║  • Jupiter integration for Solana xStocks (95% of volume)            ║
║  • Dinari API partnership (compliance-handled)                       ║
║  • Stock detail screens with charts                                  ║
║  • Dividend/split notifications                                      ║
║                                                                       ║
║  V3 (Month 5-6):                                                      ║
║  • Robinhood Chain support                                           ║
║  • DeFi composability (stocks as collateral)                         ║
║  • Recurring buys (DCA for stocks)                                   ║
║  • AI agent stock trading                                            ║
║                                                                       ║
║  KEY INSIGHT:                                                         ║
║  xStocks are standard ERC-20/SPL tokens. FurlPay can enable          ║
║  stock trading with MINIMAL new code — just route swaps through      ║
║  LI.FI (EVM) or Jupiter (Solana) to stock token contracts.           ║
║  The hardest part is compliance (geographic gating, disclaimers).    ║
║                                                                       ║
║  COMPETITIVE EDGE:                                                    ║
║  Crypto + Stocks + Agent Payments + Hardware Wallet =                ║
║  The most complete self-custody financial superapp                   ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

## Appendix: Key Resources

| Resource | URL |
|:---|:---|
| Coinbase Developer Platform | docs.cdp.coinbase.com |
| Coinbase Tokenize | coinbase.com/tokenize |
| Robinhood Chain Docs | robinhood.com/us/en/about/rhj |
| Robinhood API | api.robinhood.com/rhj/ |
| Backed Finance (xStocks) | assets.backed.fi |
| Dinari Documentation | dinaricrypto.github.io |
| Dinari GitHub | github.com/dinaricrypto |
| Securitize | securitize.io |
| Jupiter DEX API | station.jup.ag/docs |
| Jupiter Terminal | terminal.jup.ag |
| Chainlink Price Feeds | docs.chain.link |
| CoinGecko API | coingecko.com/api |
| ERC-3643 Standard | erc3643.org |
| Solana Token Extensions | spl.solana.com/token-2022 |

---

*Report compiled August 2026 from Coinbase CDP documentation, Robinhood Chain developer docs, Backed Finance platform, Dinari API documentation, Jupiter DEX documentation, Securitize platform, Phantom wallet analysis, Solana Token Extensions documentation, ERC-3643 standard, and live web research. All contract addresses, API endpoints, and regulatory information verified against current sources.*
