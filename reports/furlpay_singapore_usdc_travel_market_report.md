# Market Intelligence Report: Singapore & APAC USDC Travel Companies, Valuations & Partnership Hierarchies

**Date:** September 2026  
**Subject:** Singapore & APAC Travel Ecosystem Stablecoin/USDC Settlement Landscape  
**Prepared For:** FurlPay Core Strategy & Product Architecture  
**Scope:** Top-valued travel enterprises, OTA conglomerates, super-apps, B2B travel fintechs, luxury hospitality groups, and Web3 travel protocols accepting or settling in USDC and stablecoins.

---

## Executive Summary

The global travel industry—historically burdened by multi-day settlement windows (IATA BSP 7–14 days), 3–5% cross-border credit card interchange fees, high foreign exchange (FX) spreads, and chargeback fraud—is undergoing rapid institutional adoption of **USDC and stablecoin settlement**. 

**Singapore** has emerged as the global epicenter of this transformation due to:
1. The **Monetary Authority of Singapore (MAS)** regulatory clarity under the Payment Services Act (PS Act) and the MAS Stablecoin Regulatory Framework.
2. The presence of Tier-1 MAS Major Payment Institution (MPI) license holders specializing in digital payment token acquiring (Triple-A, dtcpay, StraitsX, Nium, Airwallex, Circle Singapore).
3. The convergence of mega-cap Online Travel Agencies (OTAs) like **Trip.com Group ($35B+)**, regional super-apps like **Grab ($15B+)**, cross-border B2B unicorns like **Airwallex ($5.6B)** and **Nium ($1.4B)**, and Web3-native AI travel pioneers like **Travala.com** and **Staynex / Sleap.io**.

This report maps the entire corporate hierarchy, valuation landscape, supported blockchain networks, settlement mechanics, and strategic disruption vectors for **FurlPay**.

---

## 1. Top Singapore & APAC Travel / Travel-Fintech Companies by Valuation

The following table benchmarks the leading companies operating in or connected to the Singapore travel ecosystem that accept, settle, or facilitate **USDC / stablecoin** payments:

| Company Name | Entity Type / Travel Sector | Corporate Valuation / Market Cap | Headquarters / Singapore Presence | Primary Stablecoins Accepted | Supported Blockchain Networks | Licensed Settlement / Acquisition Partner | Key Travel Inventory & Distribution Partners |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Trip.com Group** *(NASDAQ: TCOM)* | Global Mega-OTA (Hotels, Flights, Rail, Tours) | **$35.0B – $38.0B** *(Public)* | Shanghai, CN (Global HQ) / Singapore (APAC Regional Hub) | **USDC**, USDT | **Ethereum, Solana, Polygon, Arbitrum One, TRON, TON** | **Triple-A** *(MAS MPI)* | Direct Airline NDC, Global GDS (Amadeus, Sabre), 1.4M+ Hotels |
| **Grab Holdings** *(NASDAQ: GRAB)* | Super-App (Ride-hailing, Airport Transfers, Flights/Hotels) | **$15.0B – $18.0B** *(Public)* | One-North, Singapore *(Global HQ)* | **USDC**, USDT, XSGD, BTC, ETH | **Ethereum, Solana, Polygon** | **Triple-A** *(Wallet Top-up)*, **Circle** *(Web3 Pilot)*, **StraitsX** *(SGQR)* | Agoda, Booking.com, Klook (In-app travel partners) |
| **Airwallex** | B2B Travel Payments & Corporate Cards | **$5.6B** *(Series E+ Unicorn)* | Melbourne / Hong Kong / Singapore *(MAS MPI)* | **USDC** | **Ethereum** *(API Payouts & Inbound)* | **Self-Licensed** *(MAS Major Payment Institution)* | Global Airlines, OTAs, Tour Operators, Travel Marketplaces |
| **Nium / Nium Travel** | B2B Airline & Hotel Settlement Rails, VCCs | **$1.4B** *(Series E Unicorn)* | Singapore *(Global HQ, MAS MPI)* | **USDC** | **Ethereum, Solana** *(via Circle/Visa pilot)* | **Circle** *(CPN)*, **Coinbase**, **Visa** *(Stablecoin Pilot)* | IATA (Airlines), Agoda, Hotel Wholesalers, Global TMCs |
| **Wego (Wego Group)** | Metasearch & OTA (Flights, Hotels) | **$300M – $500M** *(Raised $65M+, Tiger Global backed)* | Dual HQ: HarbourFront, **Singapore** & Dubai, UAE | **USDC**, USDT, PYUSD, EURC | **Ethereum, Polygon, Solana, TRON, Arbitrum** | **Triple-A** *(MAS MPI)* | 700+ Airlines, 500,000+ Accommodation Providers, Cleartrip ME |
| **Travala.com** *(AVA Foundation)* | Web3-Native OTA & AI Travel Protocol | **$100M – $200M** *(Token FDV / Enterprise Value)* | Remote / Dubai / Singapore APAC Community Hub | **USDC**, USDT, AVA, 100+ Cryptos | **Base (L2), Solana, Ethereum, Polygon, BNB Chain, Avalanche** | Native Smart Contracts, **x402 Protocol**, **Coinbase AgentCore** | Expedia Partner Solutions, Booking Network, Agoda, Duffel API |
| **Capella Hotel Group** *(via dtcpay)* | Ultra-Luxury Hospitality (Capella, Patina) | **$500M+** *(Brand Asset Value, Pontiac Land Group)* | Millenia Tower, **Singapore** *(Global HQ)* | **USDC**, USDT, WUSD, FDUSD | **Ethereum, Solana, Polygon, TRON** | **dtcpay** *(MAS MPI)* | Sentosa Development Corp, Leading Hotels of the World |
| **Staynex Group / Sleap.io** | Web3 AI Vacation Club & Hotel Booking | **€12.8M ($14M+)** *(Acquisition valuation of Sleap.io)* | **Singapore** *(Global HQ)* | **USDC**, USDT, $STAY | **Camino Network (Travel L1), BNB Chain, Ethereum** | Self-hosted Smart Contracts & Camino Consortium Validators | 1M+ Hotels, Priceline/Booking.com Founder Network |
| **Dtravel (TRVL)** | Decentralized Vacation Rental Infrastructure | **$20.0M** *(Post-money valuation, $5M raised)* | Remote / Singapore Web3 Ecosystem Hub | **USDC**, USDT, TRVL, ETH | **BNB Smart Chain, Polygon, Ethereum** | Direct Smart Contracts (P2P Non-Custodial) | Direct Vacation Property Managers, Hostaway, Hospitable, Uplisting |
| **Capa Jet** *(via dtcpay)* | Private Jet Charter & Aviation Services | **$10M – $25M** *(Private Fleet Operator)* | Singapore / Dubai Hubs | **USDC**, USDT | **Ethereum, Solana, Polygon** | **dtcpay** *(MAS MPI)* | Private FBOs (Seletar Airport Singapore), Business Aviation Networks |

---

## 2. Partnership Hierarchy & Settlement Architecture

The travel stablecoin ecosystem operates as a **4-tier modular stack**. Capital flows from decentralized liquidity pools down to traditional physical travel suppliers (airlines, hotel owners, ground transportation fleets).

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ LEVEL 1: REGULATED STABLECOIN ISSUERS & PRIMARY PROTOCOLS                              │
│ - Circle Inc. (USDC, CCTP V2 Cross-Chain Protocol)                                     │
│ - StraitsX / Xfers (XSGD, XUSD on Solana & Polygon)                                    │
│ - Tether Operations (USDT)                                                             │
│ - PayPal / Paxos (PYUSD)                                                               │
│ Networks: Base (L2), Solana (SPL), Ethereum (ERC-20), Arbitrum One, Polygon            │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ Minting / Redemption / Liquidity Bridging
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ LEVEL 2: LICENSED PAYMENT GATEWAYS & B2B FINTECHS (MAS MPI LICENSEES)                  │
│ - Triple-A: Consumer crypto payment gateway (Powers Trip.com, Wego, GrabPay)           │
│ - Nium Travel: B2B virtual cards (VCC) & airline clearing (Circle CPN + Visa pilot)    │
│ - dtcpay: POS & corporate invoicing for luxury hospitality (Capella Singapore)         │
│ - Airwallex: Multi-currency enterprise accounts & Ethereum USDC API payouts            │
│ - StraitsX / OKX Pay: SGQR merchant rail integration for GrabPay retail terminals     │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ Guaranteed FX Conversion / Virtual Cards (VCC)
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ LEVEL 3: TRAVEL DISTRIBUTION SYSTEMS, GDS & B2B WHOLESALERS                            │
│ - Global Distribution Systems (GDS): Amadeus, Sabre, Travelport                        │
│ - Direct Airline NDC & Consolidators: Duffel API (Flights), IATA Settlement Modernization │
│ - B2B Hotel Aggregators: Hotelbeds, WebBeds, DidaTravel, Expedia Partner Solutions     │
│ - Travel Blockchains: Camino Network (Consortium L1 for travel data exchange)          │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ Real-Time Inventory / Rate Parity / Instant PNR
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ LEVEL 4: CONSUMER & AGENTIC TRAVEL FRONT-ENDS                                          │
│ - Global Public OTAs: Trip.com Group ($35B+), Wego ($300M+)                            │
│ - Southeast Asia Super-Apps: Grab ($15B+ Transport & Airport Rides)                   │
│ - Web3 AI Native Platforms: Travala.com (Base MCP Protocol), Staynex / Sleap.io        │
│ - Ultra-Luxury & Hospitality: Capella Hotel Group, Capa Jet Private Aviation           │
│ - Peer-to-Peer Vacation Rentals: Dtravel (P2P Smart Contracts)                         │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Deep-Dive Profiles of Market Leaders

### 3.1 Trip.com Group ($35B+ Market Cap)
- **Corporate Profile:** The dominant OTA conglomerate in Asia (comprising Trip.com, Skyscanner, Ctrip, and Qunar), listed on NASDAQ (`TCOM`) and HKEX (`9961`). Maintains its primary international headquarters and APAC technology hub in Singapore.
- **Stablecoin Integration:** Integrated **Triple-A** to process stablecoin payments for international prepaid hotel and flight reservations.
- **Accepted Stablecoins & Blockchains:** **USDC and USDT** across 6 blockchains: **Ethereum, TRON, Polygon, Solana, Arbitrum One, and TON**.
- **Settlement Mechanics:** 
  1. The international traveler selects "Pay with Cryptocurrency" at checkout.
  2. Triple-A dynamically locks a 15-minute exchange rate between the travel currency (USD, SGD, EUR) and USDC/USDT.
  3. The customer signs the transaction from any non-custodial wallet (Phantom, MetaMask, OKX Wallet).
  4. Triple-A confirms the transaction on-chain, absorbs all volatility risk, and settles fiat directly into Trip.com Group's corporate accounts.
  5. The Passenger Name Record (PNR) or hotel voucher is immediately issued.
- **Geographic Restrictions:** Strictly restricted to international users based on IP filtering; unavailable to residents in mainland China and Hong Kong to comply with national regulatory frameworks.

### 3.2 Grab Holdings ($15B+ Market Cap)
- **Corporate Profile:** Southeast Asia’s undisputed everyday super-app (NASDAQ: `GRAB`), headquartered at One-North, Singapore. Operates ride-hailing, food delivery, parcel logistics, and integrated travel services (flights and hotel bookings powered by Agoda and Booking.com).
- **Stablecoin Integration:**
  - **Triple-A Crypto Top-Up:** Since 2024, Singapore Grab users can top up their GrabPay balance with **USDC**, USDT, XSGD, BTC, and ETH. Triple-A converts the crypto instantaneously to Singapore Dollars (SGD), which the user spends across airport rides, food delivery, or in-app travel reservations.
  - **Circle Web3 Wallet Collaboration:** Grab partnered with Circle in Singapore under the MAS **Project Orchid** initiative to pilot "programmable money" (Purpose Bound Money - PBM), testing NFT travel vouchers and digital collectibles during major Singapore events like the Formula 1 Singapore Grand Prix.
  - **StraitsX / OKX SGQR Integration:** In late 2025, OKX Pay partnered with StraitsX to allow users to scan GrabPay / SGQR merchant codes at airport terminals and tourist spots in Singapore, settling the bill in USDC while the merchant receives instant SGD.

### 3.3 Nium & Nium Travel ($1.4B Valuation)
- **Corporate Profile:** Singapore-founded global fintech unicorn (valued at $1.4B, backed by Temasek, GIC, Riverwood Capital), holding an MAS Major Payment Institution (MPI) license and operating in over 40 countries.
- **Travel Industry Dominance:** Nium Travel is one of the world's largest B2B travel payment engines. It provides Just-In-Time (JIT) virtual credit cards (VCCs) for OTAs, airlines (direct IATA integration), and hotel aggregators to pay each other across borders without standard 3% card fees or slow SWIFT wire transfers.
- **USDC Settlement Engine (Launched August 2026):**
  - Enables enterprise travel platforms to **fund their Nium treasury accounts directly with USDC**.
  - Partnered with **Circle** to tap the Circle Payments Network (CPN), allowing real-time settlement rails.
  - Integrated with **Coinbase** for institutional custody and automated liquidation into local fiat currencies across 190+ countries.
  - Partnered with **Visa** on the Visa Stablecoin Settlement Pilot, testing 24/7 on-chain card clearing over Solana and Ethereum, eliminating weekend bank holidays for airline ticket settlements.

### 3.4 Travala.com & The Base MCP Agentic Revolution
- **Corporate Profile:** The leading cryptocurrency-native OTA, offering 3,000,000+ flights, hotels, and travel activities across 230 countries. Backed by Binance and governance-anchored by the AVA Foundation.
- **2026 Breakthrough — Travala Travel MCP on Base Layer 2:**
  - In June 2026, Travala unveiled the **Travala Travel MCP (Model Context Protocol)** server built natively on Coinbase's **Base Layer 2**.
  - **Agentic Travel AI:** Autonomous AI agents (running via Claude Desktop, OpenAI Swarm, or custom LangChain/AutoGen agent pipelines) can query hotel inventory, negotiate rates, and initiate bookings autonomously.
  - **Gasless USDC Settlement:** Utilizes the **x402 payment protocol** on Base, allowing AI agents to settle hotel rooms in **USDC** with near-instant finality and sub-$0.01 gas costs.
  - **Non-Custodial Security Model:** AI agents construct the travel booking and trigger the payment intent, but final signing authority is retained by the human traveler or delegated to a secure session key with strict spending boundaries (utilizing Amazon Bedrock AgentCore or Coinbase Developer Platform wallets).
  - **Developer Incentives:** Provides a 10% cbBTC (Coinbase Wrapped Bitcoin) rebate back to developers for bookings routed through their travel agents.

### 3.5 Wego Group ($300M–$500M Scale)
- **Corporate Profile:** The primary travel marketplace across the Middle East, North Africa, and Asia-Pacific, dual-headquartered in Singapore (HarbourFront) and Dubai. Backed by Tiger Global Management and MBC Group.
- **Stablecoin Integration:** In May 2026, Wego launched cryptocurrency checkout in partnership with **Triple-A**.
- **Supported Assets & Rails:** Accepts **USDC**, USDT, PYUSD (PayPal USD), and EURC across **Ethereum, Solana, Polygon, Arbitrum, and TRON**.
- **Strategic Purpose:** Solves international card decline rates for travelers booking cross-border itineraries between the GCC (UAE, Saudi Arabia) and Southeast Asia (Singapore, Thailand, Indonesia).

### 3.6 dtcpay & Luxury Singapore Hospitality
- **Corporate Profile:** Singapore-headquartered Major Payment Institution licensed by MAS, backed by a $16.5M Series A round.
- **Hospitality Portfolio:**
  - **Capella Hotel Group:** Integrated dtcpay at **Capella Singapore** (Sentosa Island) and **Patina Maldives**, enabling high-net-worth travelers to pay for multi-thousand-dollar luxury villa bookings, dining, and spa treatments using **USDC and USDT**.
  - **Capa Jet:** Enabled private jet charter clients to settle long-haul executive aviation flights directly in stablecoins, removing the multi-day friction of traditional international bank wires.
  - **Pure Stablecoin Architecture:** As of 2025/2026, dtcpay discontinued volatile crypto assets (Bitcoin/Ethereum) at point-of-sale to focus exclusively on fiat-backed stablecoins (**USDC, USDT, WUSD, FDUSD**), eliminating price-slippage disputes between guests and luxury hotel concierges.

### 3.7 Staynex Group & Sleap.io (€12.8M Acquisition)
- **Corporate Profile:** Singapore-based Web3 hospitality company chaired by **Jeff Hoffman** (co-founder of Priceline / Booking.com).
- **Acquisition of Sleap.io:** In April 2026, Staynex acquired Swiss-based Sleap.io in a deal valued at €12.8M. Sleap.io was the pioneer of booking travel over the **Camino Network** (a travel-specific consortium Layer 1 blockchain founded by European travel tech executives).
- **Core Technology:** AI-driven personalized hotel bundles with NFT-based booking receipts and utility rewards settled through the platform's native $STAY token and stablecoin pairs (USDT/USDC).

---

## 4. Blockchain Network Comparison for Travel Settlement

Travel platforms require high throughput, immediate transaction finality, and ultra-low fees to preserve margins on ticket sales. Here is how the primary blockchains compare in travel deployments:

| Blockchain Network | Average Transaction Fee (USD) | Settlement Finality | Primary Travel Platforms Utilizing | Key Advantage for Travel | Major Weakness / Bottleneck |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Base (Coinbase L2)** | **<$0.01** | Sub-second (~1.5s) | **Travala.com** *(Travel MCP)*, Coinbase Travel Agent | Native ERC-4337 Account Abstraction & seamless integration with autonomous AI agents; zero friction for USDC. | Dependent on Ethereum L1 data availability; centralized sequencer rollup architecture. |
| **Solana** | **<$0.001** | ~400 milliseconds | **Trip.com** *(via Triple-A)*, **Wego**, **Visa/Nium Pilot**, **StraitsX** | Highest TPS (2,500+ real TPS), instantaneous airline ticket ticketing, native SPL token extensions (confidential transfers). | Historical RPC flakiness under high congestion; non-EVM tooling requires separate SDKs. |
| **Polygon (PoS / AggLayer)** | **$0.01 – $0.03** | ~2 seconds | **Trip.com**, **Grab** *(Circle Pilot)*, **Wego**, **Dtravel** | Broad consumer wallet compatibility, high liquidity, trusted enterprise history in Singapore. | Slightly higher reorg risk than Ethereum L1; transitioning to zkEVM AggLayer architecture. |
| **Arbitrum One** | **$0.02 – $0.05** | Sub-second (~1.0s) | **Trip.com**, **Wego** | EVM battle-tested security, deep stablecoin liquidity pools (Aave, Uniswap), high institutional trust. | 7-day fraud-proof withdrawal challenge window if bridging back to L1 fiat on-ramps. |
| **Ethereum (Mainnet)** | **$1.50 – $15.00+** | 12–15 minutes | **Airwallex** *(B2B API)*, **Nium** *(Treasury)*, **dtcpay** *(Luxury)* | Maximum institutional security, multi-million-dollar B2B batch clearing for airlines and hotel chains. | Prohibitive gas fees make it completely unusable for consumer micro-bookings, airport taxis, or budget hotels. |
| **TRON** | **$0.50 – $1.50** | ~3 seconds | **Trip.com**, **Wego** | Massive retail USDT liquidity across emerging markets (Southeast Asia, Latin America, Middle East). | Centralized super-representative consensus; dominated by Tether USDT rather than compliant USDC. |

---

## 5. Strategic Opportunities & Disruption Vectors for FurlPay

Understanding the current partnerships of Trip.com, Grab, Nium, and Travala reveals **critical structural gaps** that FurlPay is uniquely positioned to exploit:

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                        THE FURLPAY TRAVEL SETTLEMENT ADVANTAGE                                  │
├────────────────────────────────┬────────────────────────────────┬──────────────────────────────┤
│ Legacy Stablecoin Travel Model │ FurlPay Next-Gen Architecture  │ Commercial Impact            │
│ (Trip.com, Wego, Triple-A)     │ (FurlPay Mobile & Agentic SDK) │ for FurlPay                  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────┤
│ 1. Centralized Gateway Markup  │ 1. Direct Non-Custodial Pay    │ Saves 1.5%–2.5% per booking. │
│ Triple-A / dtcpay charge 1.5%  │ Direct EIP-3009 Gasless USDC   │ FurlPay captures high margin │
│ to 2.5% spread on FX and       │ transfer on Base or Solana to  │ while offering cheaper rates │
│ settlement to fiat.            │ Duffel / Travala backend.      │ to end consumers.            │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────┤
│ 2. Web Redirect Checkout Only  │ 2. Native Mobile Hardware Enclave│ Sub-second, 1-tap booking  │
│ User redirected to a 3rd party │ FaceID / Secure Enclave signs  │ without leaving mobile app   │
│ payment gateway URL with QR.   │ EIP-712 payload; relayer pays  │ or waiting for web QR scans. │
│ 15-minute timeout window.      │ gas on Base / Solana.          │                              │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────┤
│ 3. Rigid Manual Web Search     │ 3. Multi-Model Travel MCP      │ Enables autonomous AI trip   │
│ User must manually browse      │ Integrates Travala MCP (Hotels)│ planning and programmatic    │
│ filters, flights, and rooms.   │ + Duffel API (Flights) with    │ execution within FurlPay chat│
│ No agentic delegation.         │ scoped AI session allowance.   │ and mobile surfaces.         │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────┤
│ 4. Single-Chain Lock-in        │ 4. Cross-Chain CCTP V2 & SOLR  │ Zero failed transactions due │
│ If user holds USDC on Arbitrum │ Smart Order Liquidity Routing  │ to network mismatch. User    │
│ but merchant wants Polygon,    │ auto-burns & mints across Base,│ spends any chain, supplier   │
│ transaction fails or requires  │ Solana, and Arbitrum with zero │ receives exact expected rail.│
│ manual bridge.                 │ manual bridging.               │                              │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────┘
```

### Actionable FurlPay Implementation Roadmap
1. **Activate On-Chain Gasless Relayer for Travel Bookings (`apps/web`):**
   - Connect the client device EIP-3009 `TransferWithAuthorization` signatures to our on-chain paymaster contract on **Base** and **Solana**.
   - Enable users to book flights via **Duffel** and hotels via **Travala** directly using self-custodial USDC with $0.00 gas fees.
2. **Launch FurlPay Travel MCP Tooling:**
   - Expose an MCP-compliant server allowing AI agents to search, reserve, and settle flights and accommodations via FurlPay wallet session keys (capped at user-specified daily allowances).
3. **Capture the Singapore Expat & Regional Tourist Corridor:**
   - Partner with **StraitsX** to allow instant 1:1 XSGD $\leftrightarrow$ USDC swaps, giving Singapore visitors zero-spread local spending across SGQR, MRT, and luxury hotel accommodations.

---

## 6. References & Primary Sources
- **Trip.com Group & Triple-A Official Partnership:** *Trip.com enables cryptocurrency payments for international travel bookings via Triple-A (South China Morning Post, Tech in Asia, Triple-A Press Release).*
- **Grab Web3 & Triple-A Top-Up:** *Grab introduces crypto top-ups via Triple-A; Circle & Grab MAS Project Orchid Web3 pilot (Fintech News Singapore, Circle Corporate Announcements).*
- **Nium USDC Funding & Settlement:** *Nium launches USDC funding for cross-border B2B corporate payments; Circle CPN integration & Visa stablecoin pilot (Nium Press, PYMNTS, Coinbase Developer Blog, August 2026).*
- **Travala Travel MCP on Base:** *Travala launches agentic AI hotel booking protocol using USDC on Base Layer 2 (Travala Developer Documentation, CoinMarketCap, Binance Research).*
- **Wego Stablecoin Integration:** *Wego partners with Triple-A for flight and hotel stablecoin bookings across APAC and MENA (Triple-A Announcements, Financial IT, May 2026).*
- **dtcpay Hospitality Deployments:** *Capella Hotel Group (Capella Singapore & Patina Maldives) and Capa Jet accept stablecoins via dtcpay (Capella Official Press, Forbes, Fintechnews).*
- **Staynex & Sleap.io:** *Staynex Group acquires Sleap.io for €12.8M to expand Web3 hotel booking ecosystem on Camino Network (EU-Startups, HospitalityNet, April 2026).*
