# FurlPay — Agent Rules

Turborepo monorepo. Web app + API: `apps/web` (Next.js 15 App Router). Typecheck
before committing: `cd apps/web && npx tsc --noEmit`.

**Deploying to production: `npm run deploy:prod` from the repo root.** Never
`npx vercel deploy --prod` on its own. Pushing to GitHub does not deploy, so CI
is advisory to the thing that actually ships — and ci.yml's own header records
what that cost: two production deployments broken by a lint error on commits
that had never run CI. `npm run deploy:prod` runs `predeploy` first, which
refuses unless the tree is clean, HEAD is pushed, and the CI workflow reports
success for that exact SHA (`apps/web/scripts/check-deploy-gate.mjs`).
`FURLPAY_DEPLOY_FORCE=1` skips it, loudly, for an incident hotfix.

## Security rules (mandatory for all API routes)

1. Every POST/PUT/PATCH handler MUST validate its body with Zod — either a
   schema in `apps/web/src/lib/schemas.ts` via `parse()`, or inline via
   `validateBody()` from `@/lib/validate` (which also enforces the 100KB body
   cap). Money amounts always get `.positive()` and an upper bound.
2. Public (unauthenticated) routes MUST call `rateLimitOr429()` from
   `@/lib/rateLimit` with a budget matched to the endpoint. Authenticated
   routes are backstopped by the middleware baseline (150 mutating req/min/IP)
   — add a tighter per-route limit for anything money-moving or SMS/email-sending.
3. Webhook handlers MUST verify an HMAC signature over the raw body
   (constant-time compare) and reject stale timestamps. Fail closed when the
   secret is unset.
4. NEVER merge raw client JSON into stored objects (mass assignment). Whitelist
   fields explicitly — `u.security` in particular holds `totpSecret` and must
   only be written through `/api/security/mfa`.
5. NEVER return `totpSecret`, private keys, session secrets, or full PII in any
   response. GET/PATCH responses strip secrets the same way.
6. Use `crypto.randomUUID()` / `crypto.getRandomValues()` for anything
   security-relevant — never `Math.random()`.
7. No `eval()`, no `new Function()`, no `dangerouslySetInnerHTML` with user
   input. JSON-LD goes through `components/JsonLd.tsx` (it escapes `<`).
8. Oracle prices need sanity bounds (see `solUsdPrice()` — SOL clamped to
   $1–$2000) and a degradation path; an oracle outage must not block payments.
9. Validate externally-supplied Solana addresses with `new PublicKey()` in
   try/catch; EVM addresses against `/^0x[0-9a-fA-F]{40}$/`.
10. Secrets live in env vars WITHOUT the `NEXT_PUBLIC_` prefix, registered in
    `apps/web/src/lib/env.ts` AND documented in `apps/web/.env.example`.
11. Never add a dependency without confirming it exists on npm (hallucinated
    package names are a supply-chain attack vector). `package-lock.json` is
    the version pin — commit it with any dependency change.
12. Known-accepted npm audit advisories live in `.github/audit-allowlist.json`
    (with reasons). Never run `npm audit fix --force`.

## Coordination

Multiple agent sessions may commit to `extension-redesign` concurrently.
Run `git log --oneline -3` and `git status` before editing; commit and push
small logical units promptly.
