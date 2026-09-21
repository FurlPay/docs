# Environments: development, staging, production

> Audience: whoever is moving FurlPay off Vercel and onto AWS.
> Scope: what an environment *is* in this codebase, and the variables that
> decide it. Per-platform runbooks live beside this file.

FurlPay has **one codebase and five shipping surfaces** — web, Android, iOS,
Wear OS and watchOS — and they do not share a deployment model. What they do
share is the environment contract below.

---

## 1. The three environments

| | `development` | `staging` | `production` |
|---|---|---|---|
| Web host | localhost | AWS (pre-prod account) | AWS (prod account) |
| Money movement | simulated, fail-closed | simulated, fail-closed | real |
| Database | local / seeded | isolated Postgres | isolated Postgres |
| Redis / KV | in-memory fallback | dedicated | dedicated |
| Android | debug APK | Play **internal testing** | Play **production** |
| iOS | simulator | TestFlight | App Store |
| Secrets | `.env.local` | AWS Secrets Manager | AWS Secrets Manager |

Staging is a **separate AWS account**, not a separate namespace in the
production account. That is the single most load-bearing choice on this page:
an SCP boundary between the two is what stops a staging credential from ever
touching production money. See [AWS Security Baseline](AWS-SECURITY-BASELINE.md).

---

## 2. `FURLPAY_ENV` decides whether money is real

`apps/web/src/lib/flags.ts` classifies the *financial* environment, which is
deliberately **not** the same thing as `NODE_ENV`:

```ts
export function financialEnvironment(): FinancialEnvironment {
  const raw = (process.env.FURLPAY_ENV ?? "").trim().toLowerCase();
  if (raw === "production" || raw === "staging" || raw === "development") return raw;
  // Unset — infer conservatively.
  return process.env.NODE_ENV === "production" && process.env.VERCEL_ENV === "production"
    ? "production"
    : "development";
}
```

Read the fallback carefully, because **it is a Vercel-shaped inference**.

On AWS there is no `VERCEL_ENV`. The expression is therefore always false, and
an unset `FURLPAY_ENV` resolves to `development` — which disables every entry
in `MONEY_FLAGS`. The design is correct: it defaults to the *safest* value,
not the most permissive, so the failure mode is "payments silently refuse"
rather than "test money moves for real". But it is still a production outage
waiting to happen.

> **`FURLPAY_ENV` is mandatory on AWS.** Set it explicitly in every
> environment. Do not rely on inference — the variable the inference depends
> on does not exist outside Vercel.

---

## 3. Variables that break on AWS

These are the runtime reads of Vercel-injected variables. Each one needs an
owner before the first AWS deploy.

| Location | Reads | What happens on AWS | Action |
|---|---|---|---|
| `lib/webauthn.ts` | `VERCEL_PROJECT_PRODUCTION_URL`, `VERCEL_URL` | RP ID falls back to `localhost` | **Set `FURLPAY_RP_ID` + `FURLPAY_ORIGIN`** |
| `lib/flags.ts` | `VERCEL_ENV` | financial env → `development` | **Set `FURLPAY_ENV`** |
| `lib/authz.ts` | `VERCEL_ENV` | correctly reads as production | none — fails safe |
| `api/auth/guest` | `VERCEL_ENV` | guest login disabled | none — fails safe |
| `lib/markets/instrumentQuote.ts` | both URL vars | self-referencing base URL breaks | set an explicit base URL |
| `api/ops/health` | `VERCEL_GIT_COMMIT_SHA` | version reports `dev` | inject the commit SHA at build |

### The passkey one is the dangerous one

`resolveRpIDs()` has a three-step precedence: explicit `FURLPAY_RP_ID`, then
the Vercel URLs, then `["localhost"]`. Deploy to AWS without
`FURLPAY_RP_ID` and every registration binds credentials to the RP ID
`localhost`, while every assertion from `furlpay.com` fails the RP ID check.

Passkeys are the primary sign-in method. Getting this wrong does not degrade
the product — it removes the front door.

```bash
# Required on every AWS environment, staging included.
FURLPAY_RP_ID=furlpay.com
FURLPAY_ORIGIN=https://furlpay.com
# Native callers do not present a web origin — see lib/webauthn.ts.
FURLPAY_ANDROID_CERT_SHA256=<play app-signing SHA-256>,<upload key SHA-256>
```

Staging needs its **own** RP ID matching its own domain. A passkey is scoped
to an RP ID by design, so staging and production credentials are naturally and
correctly non-interchangeable.

---

## 4. Scheduled jobs

Five cron entries live in `apps/web/vercel.json` and have no AWS equivalent
until someone builds one:

| Path | Schedule (UTC) |
|---|---|
| `/api/cron/reconcile` | `15 6 * * *` |
| `/api/cron/health` | `45 6 * * *` |
| `/api/cron/market-brief` | `30 21 * * *` |
| `/api/cron/price-cards` | `5 13 * * *` |
| `/api/cron/discord-welcome` | `20 6 * * *` |

On AWS these become **EventBridge Scheduler** rules invoking the same HTTP
paths. Two requirements carry over:

1. **The cron secret must be set.** These routes authenticate the caller. A
   production deployment once shipped without the secret and every scheduled
   job returned 503 — including the reconciliation job, which is the one that
   would have told anyone.
2. **Reconciliation is not optional.** `/api/cron/reconcile` is the ledger
   integrity check. Treat a missed run as an incident.

---

## 5. Managed services to decide on

The app reads these today. Migration means choosing, per service, between
"keep the SaaS" and "move to the AWS equivalent" — and the honest answer is
not the same for all of them.

| Today | AWS equivalent | Note |
|---|---|---|
| Upstash Redis (`KV_REST_API_*`) | ElastiCache for Valkey | see the incident below |
| Supabase Postgres | Aurora PostgreSQL Serverless v2 | already the target in the [architecture blueprint](../aws/FURLPAY_USDC_PAYMENT_FLOW_AND_AWS_ARCHITECTURE.md) |
| Resend | SES | domain + DKIM re-verification required |
| Twilio Verify | keep | no equivalent worth building |
| Persona (KYC) | keep | regulated vendor |
| Stripe Issuing | keep | regulated vendor |
| Helius (Solana RPC) | dedicated node | already blueprinted |

### A cache should not be able to break authentication

On 2026-09-21 passkey sign-in returned HTTP 500 in production. The cause was
not the auth code:

```
kv redis error 400: ERR DB capacity quota exceeded.
Threshold: 268435456 bytes, Usage: 270697125 bytes
```

The Redis instance was 2.2 MB over its 256 MiB quota. Upstash serves **reads**
at quota and rejects **writes** — and minting a WebAuthn challenge is a write.
99.5% of the space was a single market-data cache (`mkt:sec:facts:*`, 149 keys
averaging 1.7 MB). Those keys had correct TTLs; there was no leak. A cache
simply filled the store inside its TTL window and starved every other writer.

The code behaved correctly throughout — it failed closed and surfaced the real
error. The architecture was the defect: one KV store shared between bulk
caching and authentication state.

> **Migration requirement.** Auth-critical state (WebAuthn challenges,
> credential ownership, rate limits, session revocation) gets its own Valkey
> cluster, separate from bulk caches. Size and alarm them independently.
> Nothing in a `/markets` code path should be able to take down sign-in.

---

## 6. Pre-flight checklist

Before the first AWS production deploy:

- [ ] `FURLPAY_ENV` set explicitly in all three environments
- [ ] `FURLPAY_RP_ID` and `FURLPAY_ORIGIN` set, matching each environment's domain
- [ ] `FURLPAY_ANDROID_CERT_SHA256` lists **both** the upload key and the Play app-signing key
- [ ] Cron secret set, all five EventBridge rules created, reconcile alarmed
- [ ] Auth KV separated from cache KV, with its own capacity alarm
- [ ] Commit SHA injected at build so `/api/ops/health` reports a real version
- [ ] Staging is a separate AWS account with an SCP boundary
- [ ] Every secret in Secrets Manager, none in task definitions

Variable names and descriptions are maintained in-repo, not here:
`apps/web/.env.example` (621 lines) and the registry in
`apps/web/src/lib/env.ts`. Those are the source of truth; this page covers
only what *changes* when the host changes.
