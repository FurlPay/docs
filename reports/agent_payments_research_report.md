# Agent Payments & Paybox Architecture — Technical Research Report
## Building FurlPay's Agent Commerce Infrastructure
**Date:** August 2026 | **Reference:** MoonPay Paybox, x402 Protocol, MCP, Visa TAP

---

# Table of Contents

1. [What is Paybox & Why It Matters](#1-what-is-paybox--why-it-matters)
2. [The Missing Last Step: Trust Infrastructure](#2-the-missing-last-step-trust-infrastructure)
3. [Core Architecture Breakdown](#3-core-architecture-breakdown)
4. [x402 Protocol Deep Dive](#4-x402-protocol-deep-dive)
5. [Model Context Protocol (MCP)](#5-model-context-protocol-mcp)
6. [MPC Key Security (Sodot)](#6-mpc-key-security-sodot)
7. [Agent Payments Protocol (AP2)](#7-agent-payments-protocol-ap2)
8. [Visa Agentic Commerce](#8-visa-agentic-commerce)
9. [How to Build This in FurlPay](#9-how-to-build-this-in-furlpay)
10. [FurlPay Agent Vault Architecture](#10-furlpay-agent-vault-architecture)
11. [Implementation Roadmap](#11-implementation-roadmap)

---

# 1. What is Paybox & Why It Matters

## 1.1 The Problem

AI agents (Claude, ChatGPT, custom agents) can think, plan, and research — but they **cannot act financially**. The moment an agent needs to buy, pay, trade, or move money, it stops. The missing piece was never intelligence — it was **trust infrastructure**: a secure place where money sits while agents work, and moves only when you authorize it.

## 1.2 What Paybox Does

**MoonPay Paybox** (launched July 29, 2026) is a **non-custodial payments vault** for AI agents:

```
┌────────────────────────────────────────────────────────────────────┐
│                    PAYBOX FLOW                                      │
│                                                                    │
│  User ──"Book dinner at Gramercy"──▶ Claude/ChatGPT                │
│                                          │                         │
│                                    Agent researches,               │
│                                    finds restaurant,               │
│                                    fills booking,                  │
│                                    assembles payment               │
│                                          │                         │
│                                    PAUSES ◄── Agent cannot pay     │
│                                          │       without approval  │
│                                          ▼                         │
│  User ◄──"Approve $85 for Gramercy"── Push notification            │
│    │                                                               │
│    │── One tap (passkey/biometric) ──▶ Paybox Vault                │
│    │                                      │                        │
│    │                               MPC threshold sign              │
│    │                               (no single party has key)       │
│    │                                      │                        │
│    │                               Payment executes                │
│    │                               (x402 / card rails)             │
│    │                                      │                        │
│    │◀── "Table booked, paid" ────── Agent confirms                 │
└────────────────────────────────────────────────────────────────────┘
```

## 1.3 What Makes Paybox Different from Existing Agent Wallets

| Dimension | Existing Agent Wallets | Paybox | **FurlPay Opportunity** |
|:---|:---|:---|:---|
| **Platform** | CLI, SDK, file system | Mobile app (Claude/ChatGPT) | Mobile-first (React Native) |
| **Requires** | Computer + browser use | Phone only, in-chat | Same — phone-native |
| **Key Security** | Full key held by agent or user | MPC split (Sodot) — no one has full key | MPC or TEE-backed |
| **Auth** | API keys, seed phrases | Passkey (biometric, no password) | Passkey + hardware wallet |
| **Modes** | Usually autonomous only | "Always Ask" + "Autonomous" modes | Same dual-mode |
| **Rails** | Crypto only | Crypto + Visa card + fiat on-ramp | Crypto + fiat ramp |
| **Compliance** | None | MoonPay regulated (licensed) | Build on licensed rails |
| **Protocol** | Proprietary | x402 + MCP + Visa TAP (open standards) | Same open standards |

---

# 2. The Missing Last Step: Trust Infrastructure

## 2.1 The Three Pieces That Must Come Together

From the Paybox demo, the "last step" requires three components:

### Piece 1: The Key (No Single Party Holds It)
- Wallet key is **split into pieces** via MPC (Multi-Party Computation)
- Held by separate parties in hardware-isolated enclaves (TEEs)
- **Not Claude, not Paybox, not MoonPay** can independently reconstruct the key
- Built by **Sodot** — cryptography team specializing in threshold signatures

### Piece 2: The Rails (Regulated Money Movement)
- Moving real money — buying crypto, spending on card, staying compliant — requires a **licensed, regulated payments company**
- On-ramp (fiat → crypto), off-ramp (crypto → fiat), card issuance, KYC/AML
- A startup can build a wallet, but can't build the rails → partner with licensed entities
- Paybox uses MoonPay's existing regulated infrastructure

### Piece 3: The Open Layer (Agent-Agnostic Interface)
- Must work with **any AI agent** (Claude, ChatGPT, custom) — not locked to one model
- Must work with **any payment method** — crypto, card, bank transfer
- Built on open protocols: **x402**, **MCP**, **Visa TAP**
- Compatible with the ecosystem: Uniswap, D-Flow, thousands of dApps

## 2.2 Why This Is a Moat

```
╔═══════════════════════════════════════════════════════════════╗
║  THE AGENT PAYMENTS MOAT                                       ║
║                                                               ║
║  Key + Rails + Open Layer = Agent Commerce Infrastructure     ║
║                                                               ║
║  Most startups can build ONE of these:                        ║
║  • Key management (many MPC providers)                        ║
║  • Open protocol (x402 is open source)                        ║
║  • Nice mobile UX (React Native)                              ║
║                                                               ║
║  The MOAT is having all three + regulatory licenses:          ║
║  • MPC/TEE key security ← Sodot/Fireblocks/Lit Protocol      ║
║  • Regulated fiat rails ← MoonPay/Transak/Ramp partnership   ║
║  • Open protocols      ← x402 + MCP + AP2 (free)             ║
║  • Mobile UX           ← FurlPay already builds this         ║
║                                                               ║
║  FurlPay's advantage: Already has wallet + signing + chains   ║
║  Just needs: MPC layer + agent interface + x402 + rails       ║
╚═══════════════════════════════════════════════════════════════╝
```

---

# 3. Core Architecture Breakdown

## 3.1 Full System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         AI AGENT LAYER                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    │
│  │  Claude   │  │ ChatGPT  │  │ Custom   │  │ Autonomous       │    │
│  │  (MCP)    │  │ (Plugin) │  │ Agent    │  │ Agent (headless) │    │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────────┬────────┘    │
│        │              │              │                  │             │
│        └──────────────┴──────────────┴──────────────────┘             │
│                                │                                      │
│                        MCP / REST API                                 │
└────────────────────────────────┼──────────────────────────────────────┘
                                 │
┌────────────────────────────────▼──────────────────────────────────────┐
│                      FURLPAY AGENT VAULT                              │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Agent Client Manager                                         │   │
│  │  • Create agent clients (scoped API keys)                     │   │
│  │  • Define grants (permissions, limits, whitelists)            │   │
│  │  • Issue client keys for agent authentication                 │   │
│  └───────────────────┬───────────────────────────────────────────┘   │
│                      │                                               │
│  ┌───────────────────▼───────────────────────────────────────────┐   │
│  │  Policy Engine                                                │   │
│  │  • Per-tx spending caps ($100 max per transaction)            │   │
│  │  • Daily velocity limits ($500/day)                           │   │
│  │  • Recipient whitelists (only approved contracts/addresses)   │   │
│  │  • Time-based restrictions (only during business hours)       │   │
│  │  • Asset restrictions (only USDC, ETH)                        │   │
│  │  • Human approval required above threshold                    │   │
│  └───────────────────┬───────────────────────────────────────────┘   │
│                      │                                               │
│  ┌───────────────────▼───────────────────────────────────────────┐   │
│  │  Operating Modes                                              │   │
│  │                                                               │   │
│  │  ┌─────────────────────┐  ┌─────────────────────────────┐    │   │
│  │  │  "ALWAYS ASK"       │  │  "AUTONOMOUS"               │    │   │
│  │  │                     │  │                             │    │   │
│  │  │  Every transaction  │  │  Agent executes within      │    │   │
│  │  │  requires passkey   │  │  pre-authorized limits      │    │   │
│  │  │  approval on phone  │  │  (e.g., <$50, only USDC,   │    │   │
│  │  │                     │  │   only whitelisted addrs)   │    │   │
│  │  │  Best for: High     │  │                             │    │   │
│  │  │  value, sensitive   │  │  Exceeding limits triggers  │    │   │
│  │  │                     │  │  passkey approval           │    │   │
│  │  └─────────────────────┘  └─────────────────────────────┘    │   │
│  └───────────────────┬───────────────────────────────────────────┘   │
│                      │                                               │
│  ┌───────────────────▼───────────────────────────────────────────┐   │
│  │  MPC Signing Layer                                            │   │
│  │                                                               │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │   │
│  │  │ User Share   │  │ Server Share │  │ Recovery     │       │   │
│  │  │ (Phone TEE)  │  │ (Cloud TEE)  │  │ Share (KMS)  │       │   │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────────┘       │   │
│  │         │                 │                                   │   │
│  │         └── 2-of-3 TSS ──┘                                   │   │
│  │                  │                                            │   │
│  │         Valid ECDSA signature                                 │   │
│  └───────────────────┬───────────────────────────────────────────┘   │
│                      │                                               │
└──────────────────────┼───────────────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────────────────┐
│                      PAYMENT RAILS                                    │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────┐ │
│  │ x402 Crypto  │  │ Visa Agentic │  │ Fiat On/Off  │  │ DeFi    │ │
│  │ Payments     │  │ Commerce     │  │ Ramp         │  │ Swaps   │ │
│  │ (stablecoins)│  │ (card tokens)│  │ (MoonPay/    │  │ (LI.FI) │ │
│  │              │  │              │  │  Transak)    │  │         │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

# 4. x402 Protocol Deep Dive

## 4.1 What is x402?

**x402** is an open, internet-native payment protocol that repurposes the dormant **HTTP 402 "Payment Required"** status code. It enables machine-to-machine stablecoin payments within the standard HTTP request-response cycle.

- **Origin**: Created by Coinbase (May 2025)
- **Governance**: Donated to **Linux Foundation** (April 2026) — vendor-neutral
- **Backers**: Coinbase, Cloudflare, AWS, Stripe, Google
- **Status**: **v2** specification live (July 2026)
- **Volume**: Millions of transactions processed by mid-2026

## 4.2 How x402 Works

```
AI Agent (Client)                    Paid API / Service (Server)
      │                                        │
      │── 1. GET /premium-data ───────────────▶│
      │                                        │
      │◀── 2. HTTP 402 Payment Required ───────│
      │    Headers:                             │
      │    PAYMENT-REQUIRED: {                  │
      │      price: "0.01",                     │
      │      asset: "USDC",                     │
      │      network: "base",                   │
      │      recipient: "0xABC...",             │
      │      facilitator: "https://..."         │
      │    }                                    │
      │                                        │
      │── 3. Sign payment (EIP-3009) ──────────│
      │    (MPC wallet signs authorized transfer)
      │                                        │
      │── 4. Retry with PAYMENT-SIGNATURE ────▶│
      │    Headers:                             │
      │    PAYMENT-SIGNATURE: {                 │
      │      payload: "signed_transfer_data",   │
      │      network: "eip155:8453"             │  (CAIP-2 format)
      │    }                                    │
      │                                        │
      │                              Facilitator│
      │                              verifies + │
      │                              settles    │
      │                              on-chain   │
      │                                        │
      │◀── 5. HTTP 200 + Data ─────────────────│
      │    Headers:                             │
      │    PAYMENT-RESPONSE: {                  │
      │      txHash: "0x123...",                │
      │      success: true                      │
      │    }                                    │
```

## 4.3 x402 NPM Packages

| Package | Purpose |
|:---|:---|
| `@x402/core` | Core SDK — client, server, facilitator components |
| `x402-next` | Next.js middleware for paid API routes |
| `x402-express` | Express.js middleware |
| `x402-hono` | Hono middleware |
| `x402-fetch` | Client-side Fetch API wrapper (auto-handles 402 responses) |
| `x402-axios` | Axios interceptor |

## 4.4 x402 Server Implementation (Next.js)

```typescript
// apps/web/src/app/api/premium/route.ts
import { x402Middleware } from "x402-next";

export const GET = x402Middleware(
  async (req) => {
    // This code only runs after payment is verified
    return Response.json({ data: "premium content" });
  },
  {
    price: "0.01",           // 0.01 USDC per request
    asset: "USDC",
    network: "base",         // Base L2
    recipient: process.env.FURLPAY_TREASURY_ADDRESS!,
    facilitator: "https://facilitator.x402.org",
  }
);
```

## 4.5 x402 Client Implementation (Agent Wallet)

```typescript
// Agent's wallet auto-pays for x402 resources
import { x402Fetch } from "x402-fetch";

const fetcher = x402Fetch({
  walletSigner: furlpayMpcSigner,  // MPC signer from FurlPay vault
  maxPayment: "1.00",              // Max $1 per request
  allowedNetworks: ["base", "ethereum"],
});

// Agent makes request — 402 is handled automatically
const response = await fetcher("https://api.example.com/premium-data");
const data = await response.json();
```

---

# 5. Model Context Protocol (MCP)

## 5.1 MCP as the Agent Interface

MCP is the **"USB-C port"** for AI agents — a universal adapter that lets any AI model securely call tools exposed by external services.

### FurlPay MCP Server

FurlPay should expose an **MCP server** that AI agents can connect to:

```typescript
// FurlPay MCP Server — tools available to AI agents
const furlpayMcpServer = {
  tools: [
    {
      name: "get_balance",
      description: "Get wallet balance across all chains",
      parameters: { chainId: "optional" },
    },
    {
      name: "get_portfolio",
      description: "Get full portfolio with positions and values",
      parameters: {},
    },
    {
      name: "prepare_payment",
      description: "Prepare a payment (requires user approval)",
      parameters: {
        to: "recipient address or service",
        amount: "amount in USD",
        asset: "USDC, ETH, etc.",
        reason: "why this payment is being made",
      },
    },
    {
      name: "execute_swap",
      description: "Prepare a token swap (requires approval above limit)",
      parameters: {
        fromToken: "source token",
        toToken: "destination token",
        amount: "amount to swap",
        maxSlippage: "slippage tolerance",
      },
    },
    {
      name: "check_approval_status",
      description: "Check if a pending payment has been approved",
      parameters: { paymentId: "string" },
    },
  ],
};
```

## 5.2 MCP vs AP2 — Separation of Concerns

| Protocol | What It Handles | Security Level |
|:---|:---|:---|
| **MCP** | Tool discovery, balance queries, tx preparation | Read + prepare (non-sensitive) |
| **AP2** | Payment authorization, compliance, audit trails | Execute (sensitive, OAuth 2.0) |
| **x402** | Per-request micropayments for APIs/services | Auto-execute (within limits) |

> [!IMPORTANT]
> **MCP handles discovery and preparation. AP2 handles execution and compliance. x402 handles micropayments.** FurlPay's agent vault sits at the intersection of all three.

---

# 6. MPC Key Security (Sodot)

## 6.1 How MPC Eliminates Single Points of Failure

```
┌─────────────────────────────────────────────────────────────────┐
│                    MPC KEY ARCHITECTURE                           │
│                                                                 │
│  The private key is NEVER assembled in any single location       │
│                                                                 │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐       │
│  │ SHARE 1       │  │ SHARE 2       │  │ SHARE 3       │       │
│  │ User's Phone  │  │ FurlPay Cloud │  │ Recovery KMS  │       │
│  │ (Secure       │  │ (TEE/Enclave) │  │ (Cold backup) │       │
│  │  Enclave)     │  │               │  │               │       │
│  └───────┬───────┘  └───────┬───────┘  └───────────────┘       │
│          │                  │                                    │
│          └── 2-of-3 ────────┘                                    │
│               Threshold                                          │
│               Signature                                          │
│                  │                                               │
│                  ▼                                               │
│          Valid ECDSA/EdDSA Signature                             │
│          (indistinguishable from normal wallet signature)         │
│                                                                 │
│  Attack Scenarios Prevented:                                     │
│  ✅ Phone stolen → attacker has 1 of 3 shares (useless)        │
│  ✅ Cloud hacked → attacker has 1 of 3 shares (useless)        │
│  ✅ Agent compromised → can't sign beyond policy limits         │
│  ✅ MoonPay/FurlPay compromised → can't sign alone              │
│  ✅ Malicious agent prompt → policy engine blocks                │
└─────────────────────────────────────────────────────────────────┘
```

## 6.2 MPC Provider Options for FurlPay

| Provider | Key Features | Pricing | Best For |
|:---|:---|:---|:---|
| **Sodot** | Self-hosted TEE signers ("Vertex"), dynamic quorum reconfiguration | Enterprise | MoonPay's choice, max control |
| **Lit Protocol** | Decentralized MPC network, programmable signing | Per-signature | Decentralized, no vendor lock |
| **Fireblocks** | Enterprise MPC + policy engine, SOC 2 Type II | $$$$ Enterprise | Institutional grade |
| **Privy** | Embedded wallets, passkey-first, React Native SDK | Freemium | Fastest to integrate |
| **Turnkey** | Sub-organization wallets, TEE-backed, policy API | Per-wallet | API-first, programmable |
| **Crossmint** | Agentic wallet SDK, x402 support, pre-built | Freemium | Agent-specific wallets |

## 6.3 Dynamic MPC Features

- **Key Rotation**: Refresh key shares without changing the public key/address
- **Quorum Changes**: Change from 2-of-3 to 3-of-5 without migration
- **Policy-Based Signing**: Reject transactions that violate spending rules at the signing layer
- **Kill Switch**: User retains master share that can revoke agent's session at any time

---

# 7. Agent Payments Protocol (AP2)

## 7.1 What is AP2?

**AP2** (Agent Payments Protocol) is Google's open protocol specifically for agent-led commerce. It complements MCP by handling the security and compliance aspects of payments.

### AP2 Core Properties
- **Authorization**: Proves user explicitly granted the agent authority for a specific transaction
- **Authenticity**: Merchant can verify agent's request reflects user's true intent
- **Accountability**: Clear audit trails for fraud prevention and dispute resolution
- **Based on**: OAuth 2.0 / OpenID Connect

## 7.2 Protocol Comparison

| Protocol | Creator | Purpose | Model |
|:---|:---|:---|:---|
| **x402** | Coinbase → Linux Foundation | Per-request HTTP micropayments | Stablecoin per-call |
| **AP2** | Google | Agent commerce authorization/compliance | OAuth-based spending mandates |
| **MPP** | Stripe | Session-based billing for agents | Session tokens |
| **Visa TAP** | Visa | Trusted agent identity + card tokens | Card network tokens |
| **MCP** | Anthropic | Agent ↔ tool communication | Tool discovery + invocation |

---

# 8. Visa Agentic Commerce

## 8.1 Visa Intelligent Commerce (2026)

Visa's agentic commerce infrastructure provides:

- **Agentic Tokens**: Scoped card credentials bound to specific agents, merchant categories, and spending limits
- **Trusted Agent Protocol (TAP)**: Identity verification for AI agents — merchants can verify agent is legitimate
- **Agent Score**: Reliability rating for AI agents (helps merchants assess trust)
- **Agent Directory**: Registry of verified agents and merchants
- **Consumer Controls**: Spending limits, merchant whitelists, approval thresholds

## 8.2 Why FurlPay Should Care

If FurlPay implements Visa agentic tokens, agents could:
1. Buy physical goods (Amazon, restaurants, flights)
2. Pay for subscriptions
3. Handle recurring payments
4. All without ever exposing the raw card number to the AI

---

# 9. How to Build This in FurlPay

## 9.1 What FurlPay Already Has

| Component | Status | Gap to Agent Commerce |
|:---|:---|:---|
| Self-custody wallet | ✅ Built | Needs MPC for agent-safe signing |
| EIP-1559 signing | ✅ Built | Needs policy engine for agent limits |
| Multi-chain support | ✅ Built | Ready for x402 multi-chain |
| LI.FI swap integration | ✅ Built | Needs agent-triggered swaps |
| Biometric auth | ✅ Built | Extend to passkey-based agent approval |
| Transaction construction | ✅ Built | Needs agent tx preparation API |
| Secure key management | ✅ Built | Upgrade to MPC for agent isolation |

## 9.2 What FurlPay Needs to Build

### Layer 1: MPC Signing Infrastructure

```typescript
// Replace single-key signing with MPC threshold signing
import { TurnkeyClient } from "@turnkey/sdk-server";
// OR
import { LitNodeClient } from "@lit-protocol/lit-node-client";
// OR
import { PrivyClient } from "@privy-io/server-auth";

// Agent creates a signing request
const signRequest = {
  transaction: serializedTx,
  policyCheck: {
    maxAmount: "50.00",        // USD
    allowedAssets: ["USDC", "ETH"],
    allowedRecipients: ["0xLIFI...", "0xUNISWAP..."],
    requiresApproval: amount > 100,
  },
};

// MPC signing service validates policy and signs
const signature = await mpcSigner.signTransaction(signRequest);
```

### Layer 2: Agent Client & Grants System

```typescript
// apps/web/src/app/api/agent/clients/route.ts
// Create an "agent client" — a scoped identity for an AI agent

interface AgentClient {
  id: string;
  name: string;           // "Claude Shopping Assistant"
  model: string;          // "claude-4" | "gpt-5" | "custom"
  createdAt: Date;
  grants: AgentGrant[];
}

interface AgentGrant {
  id: string;
  walletId: string;        // Which wallet this agent can access
  permissions: {
    canReadBalance: boolean;
    canPreparePayments: boolean;
    canExecuteAutonomously: boolean;
    maxPerTransaction: number;   // USD
    maxPerDay: number;           // USD
    allowedAssets: string[];
    allowedRecipients: string[];  // Contract/address whitelist
    allowedChains: number[];
    expiresAt: Date;
  };
  mode: "always_ask" | "autonomous";
}
```

### Layer 3: FurlPay MCP Server

```typescript
// apps/web/src/mcp/server.ts
import { McpServer, StdioServerTransport } from "@modelcontextprotocol/sdk/server";

const server = new McpServer({
  name: "furlpay",
  version: "1.0.0",
});

// Tool: Get wallet balance
server.tool("get_balance", {
  chainId: { type: "number", optional: true },
}, async ({ chainId }, extra) => {
  const agentId = extra.authContext.agentId;
  const grant = await getAgentGrant(agentId);
  
  if (!grant.permissions.canReadBalance) {
    throw new Error("Agent not authorized to read balance");
  }
  
  const balances = await getWalletBalances(grant.walletId, chainId);
  return { content: [{ type: "text", text: JSON.stringify(balances) }] };
});

// Tool: Prepare payment (always returns approval URL)
server.tool("prepare_payment", {
  to: { type: "string", description: "Recipient address or service" },
  amount: { type: "string", description: "Amount in USD" },
  asset: { type: "string", description: "Token symbol" },
  reason: { type: "string", description: "Why this payment" },
}, async (params, extra) => {
  const agentId = extra.authContext.agentId;
  const grant = await getAgentGrant(agentId);
  
  // Policy check
  const amountUsd = parseFloat(params.amount);
  if (amountUsd > grant.permissions.maxPerTransaction) {
    return {
      content: [{
        type: "text",
        text: `Payment of $${params.amount} exceeds your limit of $${grant.permissions.maxPerTransaction}`,
      }],
    };
  }
  
  // Check daily velocity
  const todaySpent = await getDailySpend(grant.walletId, agentId);
  if (todaySpent + amountUsd > grant.permissions.maxPerDay) {
    return {
      content: [{
        type: "text",
        text: `Daily limit reached. Spent: $${todaySpent}, Limit: $${grant.permissions.maxPerDay}`,
      }],
    };
  }
  
  // Create pending payment
  const payment = await createPendingPayment({
    agentId,
    walletId: grant.walletId,
    ...params,
  });
  
  if (grant.mode === "autonomous" && amountUsd <= grant.permissions.maxPerTransaction) {
    // Auto-execute within limits
    const result = await executePayment(payment.id);
    return {
      content: [{
        type: "text",
        text: `Payment of $${params.amount} ${params.asset} executed. Tx: ${result.txHash}`,
      }],
    };
  }
  
  // Send push notification for approval
  await sendApprovalPush(grant.walletId, payment);
  
  return {
    content: [{
      type: "text",
      text: `Payment prepared. User has been notified to approve $${params.amount} ${params.asset} to ${params.to}. Reason: ${params.reason}`,
    }],
  };
});
```

### Layer 4: x402 Support (FurlPay as Both Client and Server)

**As Client** — FurlPay agents can pay for x402 services:
```typescript
// Agent wallet auto-handles 402 responses
import { createX402Client } from "@x402/core";

const x402Client = createX402Client({
  signer: furlpayMpcSigner,
  policyEngine: furlpayPolicyEngine,
  maxPaymentPerRequest: "1.00", // USD
});
```

**As Server** — FurlPay API routes can accept x402 payments:
```typescript
// apps/web/src/app/api/premium-data/route.ts
import { withX402 } from "x402-next";

export const GET = withX402(handler, {
  price: "0.001",
  asset: "USDC",
  network: "base",
  recipient: process.env.FURLPAY_TREASURY!,
});
```

### Layer 5: Passkey Approval Flow

```typescript
// Mobile app — passkey-based payment approval
import * as LocalAuthentication from "expo-local-authentication";

async function approvePayment(paymentId: string) {
  // Show payment details
  const payment = await fetchPendingPayment(paymentId);
  
  // Biometric verification
  const result = await LocalAuthentication.authenticateAsync({
    promptMessage: `Approve $${payment.amount} ${payment.asset} to ${payment.to}`,
    cancelLabel: "Reject",
    disableDeviceFallback: false,
  });
  
  if (result.success) {
    // Trigger MPC signing with user's share
    const signature = await mpcSign(payment.transaction, userKeyShare);
    await submitApproval(paymentId, signature);
  }
}
```

---

# 10. FurlPay Agent Vault Architecture

## 10.1 Complete System Design

```
┌───────────────────────────────────────────────────────────────────┐
│                    @furlpay/agent-vault                            │
│                                                                   │
│  packages/                                                        │
│  ├── core/                    # @furlpay/agent-vault-core         │
│  │   ├── AgentClientManager   # Create/manage agent identities    │
│  │   ├── GrantManager         # Define permissions & limits       │
│  │   ├── PolicyEngine         # Validate tx against policies      │
│  │   └── PaymentPreparer      # Build & simulate transactions     │
│  │                                                                │
│  ├── mcp-server/              # @furlpay/agent-vault-mcp          │
│  │   ├── tools/               # MCP tool definitions              │
│  │   │   ├── get_balance      # Read wallet balances              │
│  │   │   ├── get_portfolio    # Full portfolio view               │
│  │   │   ├── prepare_payment  # Prepare tx for approval          │
│  │   │   ├── execute_swap     # Prepare swap for approval        │
│  │   │   ├── check_status     # Check approval status            │
│  │   │   └── list_history     # Transaction history              │
│  │   └── server.ts            # MCP server entry point            │
│  │                                                                │
│  ├── x402/                    # @furlpay/agent-vault-x402         │
│  │   ├── client.ts            # x402 payment client               │
│  │   ├── server-middleware.ts # Next.js x402 middleware            │
│  │   └── facilitator.ts      # Optional self-hosted facilitator  │
│  │                                                                │
│  ├── mpc-signer/              # @furlpay/agent-vault-signer       │
│  │   ├── TurnkeySigner.ts     # Turnkey MPC integration           │
│  │   ├── LitSigner.ts         # Lit Protocol integration          │
│  │   ├── PrivySigner.ts       # Privy integration                 │
│  │   └── ISigner.ts           # Common signer interface           │
│  │                                                                │
│  ├── approval-ui/             # @furlpay/agent-vault-ui           │
│  │   ├── ApprovalSheet.tsx    # Push notification approval UI     │
│  │   ├── AgentDashboard.tsx   # Manage agents & grants            │
│  │   ├── SpendingHistory.tsx  # Agent spending analytics          │
│  │   └── PolicyEditor.tsx     # Visual policy configuration       │
│  │                                                                │
│  └── notifications/           # @furlpay/agent-vault-notify       │
│      ├── PushNotifier.ts      # Send approval requests to phone   │
│      ├── WebSocketRelay.ts    # Real-time agent ↔ app comms       │
│      └── ApprovalPoller.ts    # Agent polls for approval status   │
└───────────────────────────────────────────────────────────────────┘
```

## 10.2 User Setup Flow (Like Paybox Demo)

```
Step 1: User opens Claude/ChatGPT
        └── Adds "FurlPay" as MCP connector

Step 2: FurlPay app → "Create Agent Connection"
        └── Generate agent client + API key

Step 3: Paste API key into Claude/ChatGPT connector
        └── Agent now has scoped access to FurlPay vault

Step 4: Configure grants:
        ├── Mode: "Always Ask" or "Autonomous"
        ├── Daily limit: $500
        ├── Per-tx limit: $100
        ├── Allowed assets: USDC, ETH
        └── Allowed actions: payments, swaps

Step 5: Fund wallet (already funded via FurlPay)
        └── Agent can now prepare payments

Step 6: User says "Book dinner at Gramercy"
        ├── Agent prepares payment via MCP
        ├── FurlPay sends push notification
        ├── User taps "Approve" (passkey/biometric)
        └── MPC signs and broadcasts transaction
```

---

# 11. Implementation Roadmap

## 11.1 Phase 1: Foundation (Month 1–2)

| # | Task | Details | Effort |
|:---|:---|:---|:---|
| 1 | **MPC Signer Integration** | Integrate Turnkey or Privy as MPC provider. Replace single-key signing for agent wallets. | 3 weeks |
| 2 | **Policy Engine** | Build spending limit, velocity, whitelist, and approval-threshold enforcement. Store in Upstash Redis. | 2 weeks |
| 3 | **Agent Client & Grants API** | REST endpoints for creating agent clients, defining grants, issuing scoped API keys. | 2 weeks |
| 4 | **Passkey Approval Flow** | Push notification → biometric approval → MPC co-sign on mobile. | 2 weeks |

## 11.2 Phase 2: Agent Interface (Month 3–4)

| # | Task | Details | Effort |
|:---|:---|:---|:---|
| 5 | **FurlPay MCP Server** | Expose wallet tools (balance, prepare_payment, swap, history) via MCP. Stateless, load-balanced. | 3 weeks |
| 6 | **x402 Client** | Enable FurlPay wallets to auto-pay for x402-enabled APIs. `x402-fetch` + policy integration. | 2 weeks |
| 7 | **x402 Server Middleware** | Enable FurlPay API routes to accept x402 stablecoin payments (for premium features). | 1 week |
| 8 | **Agent Dashboard UI** | React Native screens: create agents, manage grants, view spending, configure policies. | 3 weeks |

## 11.3 Phase 3: Commerce Rails (Month 5–6)

| # | Task | Details | Effort |
|:---|:---|:---|:---|
| 9 | **Fiat On/Off Ramp** | Partner with MoonPay/Transak for regulated fiat ↔ crypto within agent vault. | 4 weeks |
| 10 | **Visa Agentic Tokens** | Integrate Visa TAP for card-based agent payments (restaurants, flights, Amazon). | 4 weeks |
| 11 | **AP2 Compliance Layer** | Implement Google's AP2 for enterprise-grade authorization/audit trails. | 2 weeks |
| 12 | **Multi-Agent Support** | Multiple AI agents per user, each with own grants and spending history. | 2 weeks |

## 11.4 Phase 4: Ecosystem (Month 7+)

| # | Task | Details | Effort |
|:---|:---|:---|:---|
| 13 | **FurlPay Agent SDK** | Publish `@furlpay/agent-sdk` for developers to integrate FurlPay vault into custom agents. | 3 weeks |
| 14 | **Autonomous DeFi Agent** | Pre-built agent that manages yield, rebalances portfolio, executes DCA within user-set limits. | 4 weeks |
| 15 | **Enterprise Multi-Signer** | Multi-approver workflows for team/corporate agent wallets. | 3 weeks |

## 11.5 Final Architecture Summary

```
╔═══════════════════════════════════════════════════════════════════════╗
║                FurlPay Agent Commerce Strategy                       ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  WHAT TO BUILD:                                                       ║
║  • Agent Vault with MPC signing (no single party has full key)       ║
║  • Policy Engine (spending limits, whitelists, velocity caps)        ║
║  • Dual-mode: "Always Ask" (passkey) + "Autonomous" (within limits)  ║
║  • MCP Server for Claude/ChatGPT/custom agent connectivity           ║
║  • x402 support (both client and server)                             ║
║  • Passkey-based approval via push notification                      ║
║                                                                       ║
║  PROTOCOLS TO IMPLEMENT:                                              ║
║  • x402 — Stablecoin micropayments (Linux Foundation, open)          ║
║  • MCP  — Agent ↔ tool communication (Anthropic, open)               ║
║  • AP2  — Agent payment authorization (Google, open)                 ║
║  • Visa TAP — Card-based agent identity (for fiat commerce)         ║
║                                                                       ║
║  MPC PROVIDER OPTIONS:                                                ║
║  • Turnkey (API-first, TEE-backed, recommended for startups)         ║
║  • Lit Protocol (decentralized, no vendor lock-in)                   ║
║  • Privy (fastest to integrate, React Native SDK)                    ║
║  • Sodot (MoonPay's choice, self-hostable, enterprise)               ║
║                                                                       ║
║  FURLPAY'S ADVANTAGE:                                                 ║
║  • Already has wallet + signing + multi-chain + biometric auth       ║
║  • Already has swap/bridge integration (LI.FI)                       ║
║  • Mobile-first (React Native) — no CLI/file system needed           ║
║  • Can combine hardware wallet support + agent vault (unique!)       ║
║                                                                       ║
║  THE KILLER COMBO:                                                    ║
║  Hardware wallet for high-value + Agent vault for daily AI commerce   ║
║  = The most secure agent payment system on the market                ║
╚═══════════════════════════════════════════════════════════════════════╝
```

> [!IMPORTANT]
> **The "last step" that Paybox solved** is the trust infrastructure — the vault where money sits safely while agents work. FurlPay can build this by combining:
> 1. **MPC signing** (Turnkey/Lit/Privy) so no single party has the full key
> 2. **Policy engine** so agents can only spend within user-defined limits
> 3. **MCP server** so any AI agent (Claude, ChatGPT, custom) can connect
> 4. **x402 protocol** so agents can pay for internet services autonomously
> 5. **Passkey approval** so high-value txs require one-tap biometric confirmation
>
> FurlPay's unique advantage: combining this with **hardware wallet support** — hardware for high-value custody, MPC vault for daily agent commerce.

---

## Appendix: Key Resources

| Resource | URL |
|:---|:---|
| Paybox | paybox.sh |
| x402 Protocol Spec | x402.org |
| x402 NPM Packages | npmjs.com/package/@x402/core |
| x402 Starter Kit | github.com/dabit3/x402-starter-kit |
| MCP Specification | modelcontextprotocol.io |
| AP2 Protocol | developers.google.com (Agent Payments) |
| Visa Intelligent Commerce | visa.com/intelligent-commerce |
| Sodot MPC | sodot.dev |
| Turnkey | turnkey.com |
| Lit Protocol | litprotocol.com |
| Privy | privy.io |
| Crossmint Agent Wallets | crossmint.com |
| MoonPay Developer Docs | docs.moonpay.com |

---

*Report compiled August 2026 from MoonPay Paybox demo analysis, x402 protocol documentation, MCP specification, Visa agentic commerce announcements, Sodot/MPC provider documentation, and live web research. All protocol specifications, package names, and architectural recommendations verified against current developer resources.*
