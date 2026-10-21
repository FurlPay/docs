# Resilience: design principles and status

> Audience: integrators, partners and engineers evaluating how FurlPay behaves when part of the infrastructure fails.
> **Status: PROPOSED.** Every capability below is labelled **Live**, **In review** or **Planned**. Nothing labelled Planned should be read as running today.

## Why this document exists

On 20 October 2026, a power event in a single Google Cloud zone in London (`europe-west2-b`) disrupted card payments, transfers and sign-ins across UK financial services for most of a day. Customer funds were never at risk, but availability was lost.

The lesson applies to anyone building payment infrastructure, including us: **availability is part of the product, and a system is only as resilient as its most concentrated dependency.**

## Principles

1. **No single availability zone in the request path.** Losing one zone must reduce capacity, not stop service.
2. **Every money-moving request is safe to retry.** A client that loses a response must be able to ask again without paying twice.
3. **Fail closed on money, degrade gracefully everywhere else.** When a shared safety dependency is unavailable, sign-in and money movement refuse; browsing and read paths keep working.
4. **Settlement is decoupled from the application host.** Stablecoin settlement runs on public blockchains, whose ledgers do not live in any single company's data centre. The application layer still runs on cloud providers, and we design for that.
5. **Redundancy is claimed only after it is drilled.** A failover path that has never been exercised is not counted.

## Status

| Capability | Status | Notes |
|---|---|---|
| Application tier spread across multiple availability zones | **Live** | Serverless functions are multi-AZ by default on our hosting platform. A single-zone failure like 20 October is absorbed at this tier. |
| Idempotent money APIs | **Live** | See [the contract below](#retrying-safely-the-idempotency-contract). |
| One key per payment intent, reused on retry | **Live**: iOS, Android, Wear OS, mobile app · **In review**: web, browser extension | Each client mints the key once per user action, never once per network attempt. |
| Fail-closed sign-in and money limits | **Live** | If the shared rate-limit store is unreachable, authentication and money-movement endpoints refuse rather than run unthrottled. Other endpoints fall back to per-instance limits. |
| Three-AZ AWS deployment (standby) | **Planned** | Infrastructure-as-code is written: 3 AZs selected by zone ID, region-agnostic, default `eu-west-1`. **Not yet applied.** |
| Automatic zonal shift away from an impaired AZ | **Planned** | Requires the AWS deployment, N+1 capacity, and practice runs with outcome alarms. |
| Independent uptime monitoring and automated post-deploy verification | **Planned** | Monitoring outside our hosting provider, and automatic rollback when a release fails health checks. |
| Multi-region application tier | **Planned** | Not claimed today. |

## Retrying safely: the idempotency contract

Money-moving endpoints accept an `Idempotency-Key` header (a UUID is recommended).

| Situation | Server behaviour | What the client should do |
|---|---|---|
| First request with a key | Runs the operation | — |
| Retry with the same key after a **successful (2xx)** result | Returns the stored result verbatim; the operation does **not** run again | Treat it as the original response |
| Same key while the first request is **still in progress** | `409` with `error: "idempotency_conflict"` | Wait briefly, retry with the **same** key |
| Duplicate-protection store temporarily unavailable | `503` with `error: "idempotency_unavailable"` and `Retry-After`; **nothing was executed** | Retry with the **same** key after the indicated delay |
| Same key after an **error** response | Errors are not pinned to the key; the operation is attempted again | Retry with the **same** key if the error was transient |

Rules for clients:

- **Mint the key once per user intent** (one payment, one order) and reuse it for every retry of that intent. A new key per attempt defeats the protection.
- **Use a new key only when the request itself changes.** A different amount, recipient or product is a new intent. A key is bound to the first request body seen under it.
- **Retry only transient failures:** network errors, `502`, `503`, `504`, and `409 idempotency_conflict`. Treat other `4xx` responses as final.
- Successful results are kept for **24 hours**.

## What we do not claim

- We do not claim automatic failover between regions or between cloud providers today.
- We do not claim zero downtime. The goal is to degrade gracefully and never to charge twice.
