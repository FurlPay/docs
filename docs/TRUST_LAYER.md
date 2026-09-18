# FurlPay Trust Layer

**Status:** core implemented and tested. Signal collection, persistence and the operations console are not built.
**Updated:** 6 September 2026

---

## Research caveat — read this first

This design was requested alongside a research brief on how named vendors detect fraud. **Web search hit its session limit before I could verify those claims**, so this document is built from the repository, from first principles, and from primary sources I could reach in earlier sessions.

I have therefore **not** attributed capabilities to any vendor. The brief itself contained a contradiction — one passage called a well-known browser-fingerprinting library MIT-licensed, another said v4 of the same library is under a Business Source Licence. Those cannot both be true, and **licence terms decide whether a library may be used commercially at all**. Treat every vendor and licence claim in that brief as **UNVERIFIED**, and check the licence file in each repository before anyone imports it.

What follows is labelled throughout:

- **IMPLEMENTED** — code in this repository, with tests
- **RECOMMENDATION** — my architectural judgement
- **ASSUMPTION** — plausible, unverified
- **REQUIRES VENDOR VALIDATION** — needs a contract and a technical evaluation

---

## 1. What replaced what

**IMPLEMENTED.** The prior engine was `packages/security/src/risk.ts`:

```ts
if (signals.automation) { score += 75; }
if (signals.emulator)   { score += 40; }
if (signals.vpn)        { score += 20; }
if (signals.newDevice)  { score += 15; }
score = Math.min(100, score);
```

Four boolean signals, fixed additive points, clamped at 100, returning prose strings. Three defects follow structurally:

| Defect | Consequence |
|---|---|
| **Saturation** | Two signals reach the block threshold. "New device on a VPN" scores identically to a confirmed fraud ring. Past the ceiling the score carries no information. |
| **No confidence** | Two weak signals and thirty corroborating ones are indistinguishable — the difference between "block this" and "we don't know enough to block anyone". |
| **No decay** | A signal from six months ago counted the same as one from ten seconds ago. A customer could never recover from one bad day. |

Replaced by four modules under `lib/trust/`, **60 tests**.

---

## 2. Reason codes — `lib/trust/reasons.ts`

**IMPLEMENTED.** 31 codes across 9 categories, each with a stable identity, severity, direction, analyst explanation and a **separately authored** customer message.

Two rules the registry enforces:

**Disclosure is deliberately asymmetric.** `analystExplanation` may name the detection method; `customerMessage` is vaguer, and is `null` for anything that reveals a threshold. A ring that learns "declined because 6 cards in 10 minutes" simply uses 5. `customerFacingMessage()` returns the single most severe *disclosable* message — never a concatenation, because three specific messages together re-disclose what each one alone was written to protect.

**Almost nothing is `sufficientAlone`.** Only `CMP.SANCTIONS_MATCH` and `CMP.JURISDICTION_BLOCKED`, and a test enforces that the standalone set stays ≤3 and stays in the compliance category. A single-signal decline is a single point of failure for a false positive, and the most damning-looking signals — shared device, VPN — have entirely ordinary explanations.

**Trust-lowering reasons are first-class.** `DEV.LONG_TENURE`, `ACC.ESTABLISHED_GOOD_HISTORY`, `IDN.DOCUMENT_VERIFIED`. Without them a risk engine only ratchets upward and every good customer eventually accumulates enough noise to be stopped.

---

## 3. Trust score — `lib/trust/score.ts`

**IMPLEMENTED.** 0–1000, banded as specified: trusted 0–199, low 200–399, medium 400–599, high 600–799, critical 800–1000.

### The model

Within a category, evidence combines by **noisy-OR**: `combined = 1 − Π(1 − pᵢ)`, where `pᵢ = severity × confidence × decay`. This treats each reason as independent evidence for the same conclusion, so two weak signals reinforce while ten weak signals approach but never reach one certain signal. Categories are then weighted and summed, because they are *not* independent evidence for one another.

### Two calibration defects the tests caught

**1. A detected card-testing run auto-approved.** `PAY.CARD_TESTING` alone scored `0.85 × 240 = 204` → "low" → approve. Fixed with a **critical floor**: any critical-severity reason with strength ≥ 0.3 floors the band at 600. It does *not* force a decline — confidence still governs that — so a lone critical signal reaches a human rather than refusing a customer on one input.

**2. A 12-account ring scored 212.** The first weights were set so their *sum* couldn't overshoot 1000, which meant no single category could reach even "medium". The engine could only act when three unrelated categories agreed — in practice, rarely. Weights were recalibrated so one fully-corroborated category reaches medium, two reach high, three-plus saturate to critical. Overshoot is expected and clamped.

Network stays lowest by a wide margin (180 max). An IP-derived signal can never on its own push a customer past "trusted" — the intended outcome for a control whose false positives land on travellers and VPN users.

### The confidence guard

**The most important property in the system.** `MIN_CONFIDENCE_TO_DECLINE = 0.6`. A high score on thin evidence routes to `manual_review`, never `decline`.

Rationale: declining on thin evidence concentrates false positives on the customers we know *least* about — new, foreign, thin-file — and those customers cannot argue, because nobody looked.

### Decay

Half-lives by category: behaviour and network 3 days, device 30, account 90, payment and identity 180, graph 365, compliance 3650. An unparseable timestamp is treated as **fully decayed, not fully fresh** — unknown provenance must never be the strongest evidence in the set.

---

## 4. Entity graph — `lib/trust/graph.ts`

**IMPLEMENTED.** Pure graph analysis over a supplied edge set. No database, no I/O, so it runs over an in-memory fixture in a test or a Postgres recursive CTE in production without either leaking in.

### Per-type sharing thresholds

The central design point. Shared attributes are **not** evidence of fraud by default — a family shares a tablet, a household shares an address, a campus shares an IP, a refurbished handset carries a device id to a stranger.

| Entity | Threshold | Why |
|---|---:|---|
| phone, email, card, bank_account | 2 | Meant to be personal; few benign explanations |
| wallet | 3 | — |
| device | 5 | A household device serves 4–5 people |
| address | 6 | Family, shared flat, small office |
| **ip** | **50** | Corporate NAT, campus wifi, CGNAT |
| asn, merchant | 100,000 | Effectively never evidence |

**IP and ASN never produce a graph reason at all**, at any count. A test pins this with 400 accounts on one IP: zero reasons. Flagging that flags an office building.

### Distance and link quality

`via` carries the **intermediate** hops only — never the origin's own type (a bug the tests caught; it made every path look like it ran through an "account"). A direct card link to confirmed fraud scores confidence 0.85; the same link two hops out through an IP discounts to ~0.13.

### Ring detection

Density = accounts ÷ distinct shared resources. Calibrated against the fixtures: twelve ordinary customers with their own device and card produce **0.5**; twelve accounts funnelled through three devices and two cards produce **2.4**. Threshold sits at **2.0**, with margin both sides.

Traversal depth defaults to **6**, not 3 — the graph is bipartite in practice, so every account-to-account hop costs two levels. At depth 3 only 8 of 12 ring accounts were reachable and the density of a *fragment* was reported as the whole.

### Feedback-loop protection

`auto_declined` **does not** propagate as fraud evidence. Only human-`confirmed_fraud` does. An automated decline may itself be a false positive, and treating it as ground truth is how one bad decision propagates through a graph into a hundred — the mechanism that poisons a fraud system from the inside. A test pins this.

---

## 5. Adaptive verification — `lib/trust/verification.ts`

**IMPLEMENTED.** Maps (band, action, what we already hold) → the cheapest verification that resolves the doubt.

**The premise:** maximum verification on everyone is not the safe choice, it is a different failure. It abandons the honest funnel while a ring completes a liveness check with a stolen ID, because for them the expected value justifies the effort.

**The second premise:** friction is unevenly distributed. Document checks fail more often for worn IDs, non-Latin names, unusual formats; liveness fails more in poor lighting and on cheap cameras. Every step-up is a small tax landing on those least able to absorb it.

### The routing rule

Prefer methods that collect **no new personal data**. A passkey re-assertion and a document capture both answer "is this the account holder" — only one adds a biometric template and a government identifier to our retention obligations. Unheld data cannot be breached.

| Doubt | Response |
|---|---|
| Trusted / low | Nothing |
| Medium, **presence** in doubt | Passkey → TOTP → OTP, cheapest first, any one suffices |
| Medium, **identity** in doubt | Document **and** liveness, both required |
| High | `human_review` — do not ask the customer for documents on evidence a human hasn't confirmed |
| Critical / sanctions | Nothing asked — nothing the customer provides changes the answer |

`document` and `liveness` are always issued **together**: a document proves a document is authentic, liveness proves someone is present, and only the pair proves the presenter owns it.

---

## 6. What is NOT built

**NOT STARTED**, honestly labelled:

| Component | Note |
|---|---|
| Signal collection SDKs (web/iOS/Android) | `lib/fingerprint.ts` and `lib/behaviorTracker.ts` exist as a browser collector; no mobile SDK |
| Graph persistence | The analysis layer is pure; nothing populates or stores edges |
| Feature store, event streaming | No Kafka/Flink; no `trust_*` tables |
| ML models | No GBDT, no GNN. **RECOMMENDATION: do not start here** — rules and graph features first, models when labels exist |
| Fraud operations console | No analyst UI |
| Case management | No investigation workflow |
| Model monitoring | Nothing to monitor yet |
| Continuous trust / score evolution | Scores are computed per-call, not persisted or evolved |

---

## 7. Build vs buy — RECOMMENDATION

| Layer | Position | Reasoning |
|---|---|---|
| Reason codes, trust score, decision engine | **BUILD** — done | The moat. Correlation and decisioning is the product; a vendor's score is a number you cannot defend to a regulator |
| Entity graph | **BUILD** — done | Ditto. Also, no vendor sees your full graph |
| Adaptive verification policy | **BUILD** — done | Encodes your risk appetite and your fairness posture |
| Document verification, liveness | **BUY** | Capital-intensive, adversarial, commoditised. **REQUIRES VENDOR VALIDATION** |
| Sanctions / PEP / adverse media | **BUY** | Data-licensing problem, not an engineering one |
| Phone / email intelligence | **BUY** | Data access is the product |
| Device intelligence | **HYBRID** | Own the persistent device↔account↔outcome relationship; a vendor signal is one input. **Check licences before importing anything** |
| Blockchain analytics | **BUY** | Already flagged in `PRODUCTION_READINESS.md` |

**The distinction that matters:** vendors supply *signals*. FurlPay owns *correlation and decision*. A vendor score that arrives as an opaque number cannot be explained to a customer or an auditor, and cannot be tuned against your own outcomes.

---

## 8. Red team — where this design is weakest

Honest self-attack. **ASSUMPTION** throughout; none of this is tested against live adversaries.

| Attack | Status |
|---|---|
| **Slow-drip ring** — 3 accounts per device, one card each, spread over months | **Bypasses this.** Every threshold is per-entity and instantaneous. Density needs ≥6 accounts. **Mitigation not built:** temporal ring detection over rolling windows. |
| **Deepfake liveness** | Liveness is scored 0.65 assurance and the comment says it is far weaker than that against a competent real-time face-swap. **Mitigation not built:** document+liveness pairing helps; injection-attack detection does not exist. |
| **Residential proxies** | `NET.RESIDENTIAL_PROXY_SUSPECTED` exists as a code; **nothing populates it.** |
| **Synthetic identity** | `IDN.SYNTHETIC_PATTERN` exists as a code; **nothing populates it.** Bank-account proof is the strongest available counter and is unbuilt. |
| **Graph poisoning** — deliberately linking a victim to a fraud entity | Partly mitigated: only human-confirmed labels propagate, and distance discounts. **Not mitigated:** an attacker who can cause a shared IP or a shared merchant edge. |
| **Confidence gaming** — keep signals below the corroboration bar | **Bypasses this.** The confidence guard that prevents false positives also raises the bar for true positives. This is a deliberate trade, not an oversight. |
| **Score probing** — vary one input, observe the outcome | Partly mitigated by asymmetric disclosure. **Not mitigated:** no rate limiting on decision endpoints, because there are no decision endpoints yet. |

---

## 9. Next, in order

1. **Populate the graph.** The analysis is built and nothing feeds it. `trust_entity` / `trust_edge` / `trust_label` tables + edge extraction on the existing event stream.
2. **Persist assessments.** Continuous trust needs a score history; today every call recomputes from nothing.
3. **Wire `assessRisk` call sites to `assessTrust`.** The old 4-signal engine is still what `/api/wallets/transfer` calls.
4. **Temporal ring detection.** The slow-drip bypass above is the largest known gap.
5. **Shadow mode before enforcement.** Log decisions without acting on them for weeks; measure the false-positive rate against real outcomes *before* anything blocks a customer.

Step 5 is not optional. Every threshold in this document is **policy, not measurement** — reasoned but uncalibrated against real fraud. Enforcing uncalibrated thresholds on real customers is how a fraud system does more damage than the fraud.
