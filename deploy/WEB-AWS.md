# Web deployment: Vercel → AWS

> Surface: `apps/web` — Next.js 15.5 (App Router), React 18.3.
> **314 API routes, 219 pages.** This is the backend as well as the website;
> the mobile apps call it.

---

## 1. Target: ECS Fargate behind CloudFront

The [payment flow architecture](../aws/FURLPAY_USDC_PAYMENT_FLOW_AND_AWS_ARCHITECTURE.md)
already specifies ECS Fargate, Aurora PostgreSQL Serverless v2 and ElastiCache
for Valkey. That choice holds, and it is worth saying why, because the
alternatives are tempting.

| Option | Verdict for FurlPay |
|---|---|
| **ECS Fargate** ✅ | container parity with local, full control of runtime and networking, private subnets, straightforward Nitro Enclave adjacency |
| Amplify Hosting | fine for marketing sites; not for an app where caching and compute need fine control |
| OpenNext + Lambda | cheap at low traffic, but cold starts on a card-authorization path are the wrong trade |
| **App Runner** ❌ | **stops accepting new customers 30 April 2026** — AWS directs new workloads to ECS Express Mode |

The deciding factor is not cost. It is that a Rain JIT card authorization has
a latency budget measured against Visa's 200 ms threshold, and Lambda cold
starts are not a variable you want inside it.

```
Route 53 → CloudFront → WAF → ALB → ECS Fargate (private subnets)
                                        ├── Aurora PostgreSQL Serverless v2
                                        ├── ElastiCache Valkey — auth state
                                        └── ElastiCache Valkey — bulk cache
```

Two Valkey clusters, deliberately. See
[Environments §5](ENVIRONMENTS.md) for the incident that made that
non-negotiable.

---

## 2. What has to change in the code

Nothing structural. Next.js runs in a container as-is. What breaks is the
handful of places that read Vercel-injected variables — enumerated with
consequences in [Environments §3](ENVIRONMENTS.md).

The short version, in priority order:

1. **`FURLPAY_RP_ID` + `FURLPAY_ORIGIN`** or passkeys break entirely.
2. **`FURLPAY_ENV=production`** or every money flag silently disables.
3. An explicit base URL for `lib/markets/instrumentQuote.ts`.
4. The commit SHA injected at build for `/api/ops/health`.

`standalone` output keeps the image small:

```js
// next.config.mjs
export default { output: "standalone" /* … */ };
```

Note the existing region pin in `apps/web/vercel.json`:
`src/app/api/markets/perps/**` runs in `fra1`. Decide deliberately whether
that latency requirement survives the move, or whether it was a Vercel-shaped
workaround.

---

## 3. Scheduled jobs

Five Vercel crons become **EventBridge Scheduler** rules. Paths and schedules
are tabulated in [Environments §4](ENVIRONMENTS.md).

Each rule invokes the existing HTTP path with the cron secret. Two failure
modes have already happened in production and must be designed out:

- **No secret → every job 503s**, including `/api/cron/reconcile`, which is
  the job that would otherwise have reported the problem.
- **Silent failure.** Alarm on the *outcome* of reconciliation, not on whether
  the invocation returned 200.

---

## 4. Pipeline

```
push → CI (lint, tsc, vitest, route-security scanner)
     → build image → scan in ECR → push by digest
     → deploy STAGING → smoke tests
     → manual approval
     → deploy PRODUCTION (blue/green) → smoke tests → auto-rollback on alarm
```

The repo already has the gates this pipeline should enforce:

| Gate | Command |
|---|---|
| Typecheck | `cd apps/web && npx tsc --noEmit` |
| Tests | `npx vitest run` (3,900+ assertions) |
| Route security | `node apps/web/scripts/check-route-security.mjs` |
| Route manifest | classification of all 314 routes, CI-enforced |

`check-route-security.mjs` enforces the two mandatory rules in `AGENTS.md` —
Zod body validation and rate limiting on public routes. It currently reports
**0 FAIL**. Keep it that way: it is the cheapest guard in the repo against a
new route shipping without validation.

### Keep the deploy gate

Today `npm run deploy:prod` refuses unless the tree is clean, HEAD is pushed,
and CI reports success for **that exact SHA**. That gate exists because two
production deployments were broken by a lint error on commits that had never
run CI. Whatever replaces it on AWS must keep the same property: *the artefact
deployed is the artefact that passed CI*. Deploying from a local working
directory is how the original incident happened.

---

## 5. Cutover

1. Run AWS staging against production-shaped data. Exercise sign-in — all four
   methods — before anything else.
2. Run **both** hosts in parallel; compare `/api/ops/health` and reconciliation
   output daily.
3. Move DNS with a low TTL, weighted if CloudFront is fronting both.
4. Keep Vercel warm for one full reconciliation cycle before decommissioning.

**Rollback is DNS** until Vercel is torn down. Keep it that way for at least a
week of clean reconciliations.

### Passkeys are the thing to test first

The RP ID is bound into every stored credential. If staging and production
have different domains — they should — then their credentials are correctly
non-interchangeable, and a passkey enrolled against staging will not
authenticate against production. That is the standard working as designed, and
it means **passkey sign-in cannot be validated by a staging test alone**.
Verify the production RP ID configuration explicitly, before cutover, against
the real domain.

---

## 6. Post-cutover

- [ ] All four sign-in paths verified on the real domain: passkey, Google, Apple, phone OTP
- [ ] All five scheduled jobs ran once and reconciliation balanced
- [ ] `/api/ops/health` reports a real commit SHA
- [ ] Auth Valkey separate from cache Valkey, alarmed at 70%
- [ ] WAF in count mode first, then blocking, once false positives are understood
- [ ] Vercel project disabled only after a week of clean reconciliations
