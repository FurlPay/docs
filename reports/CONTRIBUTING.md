# Contributing to FurlPay

Thanks for taking the time. This document is short and specific — it covers the
few things that are genuinely non-obvious about a repository that moves money.

## Before anything else

**Do not open a public issue for a security vulnerability.** See
[SECURITY.md](SECURITY.md). Publishing one is itself the harm; it cannot be
undone by deleting the comment.

## Repository layout

Turborepo monorepo. The pieces you are most likely to touch:

| Path | What it is |
| --- | --- |
| `apps/web` | Next.js 14 App Router — the web app and every API route |
| `apps/extension` | MV3 browser extension (WXT + React) |
| `native-app` | Expo React Native app (Android) |
| `guardian` | Kotlin phone + Wear OS apps |
| `packages/*` | Published SDKs and libraries |
| `supabase/migrations` | Durable schema |

## Getting set up

```sh
npm install
cd apps/web && npx tsc --noEmit   # typecheck
npm test                          # publishable packages
```

Nothing needs credentials to run. `INTEGRATION_MODE` defaults to `mock`, every
integration degrades to a deterministic simulation, and the money paths **fail
closed** rather than fabricating a settlement. That is deliberate — the test
suite must never be able to move real funds.

## The rules that are actually enforced

[`AGENTS.md`](AGENTS.md) is the source of truth and CI enforces a good deal of
it. The ones that reject a pull request most often:

1. **Every mutating handler validates its body with Zod.** Money amounts get
   `.positive()` and an upper bound.
2. **Public routes call `rateLimitOr429()`.** The middleware baseline only
   covers *mutating* methods, so an unauthenticated `GET` has no backstop of its
   own and must bring one.
3. **Authenticated routes call `requireUser()`** — not a hand-rolled
   `verifySession()`. The shared guard also applies the cross-instance
   revocation denylist, MCP connector scopes, the deleted-account tombstone and
   the account lock. A hand-rolled check silently skips all four.
4. **Never merge raw client JSON into stored objects.** Whitelist fields.
5. **Never add a dependency without confirming it exists on npm.** Hallucinated
   package names are a live supply-chain attack vector.

`security/route-manifest.json` is executable policy, not documentation:
`routeManifest.spec.ts` parses every route with the TypeScript compiler and
fails CI when a classification contradicts the implementation. If you add a
route, classify it. If your change removes a guard, the build will say so.

## Tests

Add them for behaviour, not for coverage. Two conventions worth copying:

- **Tests that need a real backend SKIP loudly rather than passing.** The
  Postgres and Redis suites report as skipped when unconfigured, so a green run
  on a laptop is never mistaken for evidence.
- **Assert the thing that would actually break.** Byte layouts, concurrency
  outcomes and failure modes — not that a function returns a value.

## Branches

One long-lived branch, `main`, plus short-lived work branches off it. There is
no `develop` and no release branch — the repository is small enough that a
second permanent branch would only ever be a queue with no one in it.

```
main ──────────────────────────────────────────►  production
  │                                             ▲
  └── feat/markets-real-data ── PR ── CI ── preview ──┘
```

Name a branch `<type>/<what>`, using the same types as the commit convention:

```
feat/markets-real-data      fix/vercel-build
fix/ios-watch-build         security/npm-audit
chore/bump-next             docs/api-auth
refactor/market-providers
```

Not personal names, not `test`, not `final-v2`. The branch name is the first
thing anyone reads about the change.

Keep work branches short-lived. `extension-redesign` currently runs 20+ commits
ahead of `main` for weeks at a time. A branch that never lands accumulates risk
faster than any gate removes it, and it also means CI failures pile up on a PR
nobody is about to merge — which is how two red commits sat on the branch while
production deployments failed against them.

## Commits and pull requests

- Conventional commits (`feat:`, `fix:`, `chore:`, `refactor:`, `security:`,
  `ci:`, `docs:`) with a scope.
- Explain *why* in the body. The diff already shows what.
- Run `npx tsc --noEmit`, the test suite, **and the production build** before
  opening — see below.
- Add a changeset (`npx changeset`) for any change to a published package.
- Squash on merge. One reviewable change becomes one commit on `main`; the
  discussion stays on the PR where it is searchable.

### `next build` is not implied by the other two

`npx tsc --noEmit` and `vitest` can both pass on a commit that fails
`next build`, because only the build runs ESLint. That is exactly how commits
`024b84b` and `941ab52` broke two production deployments — with an
`// eslint-disable-next-line @typescript-eslint/no-explicit-any` comment, for a
rule this ESLint config (`next/core-web-vitals`) never registers. ESLint treats
a disable comment for an unknown rule as an error.

So: run `npm run build --workspace @furlpay/web` before you push, and do not
reach for a disable comment when the fix is to type the thing properly.

## Deployment

Vercel builds a **preview** for every branch and PR. Production is a separate,
deliberate step:

```
push branch → CI + Vercel preview → PR → review → merge to main
                                                      │
                                    npx vercel deploy --prod --yes
```

Pushing to GitHub does **not** deploy production. The production deploy is run
manually from the repo root, and it uploads your **working directory** — not the
last commit — so deploy from a clean tree on the branch you mean to ship.

## Database migrations

`supabase/migrations/` is append-only and numerically ordered. Applied
migrations are never edited afterwards; a mistake is corrected by a new
migration, because staging and production have already run the old one and will
not run it again. Ship the migration and the code that needs it together, and
apply to staging before the PR merges.

## Security fixes

Branch `security/<what>`, keep it to the fix alone, and do not fold in a
refactor — a security PR should be reviewable in one sitting. Never open a
public issue describing an unpatched vulnerability; see [SECURITY.md](SECURITY.md).

## Reporting a bug

Include what you expected, what happened, and the smallest reproduction you
can. For anything money-related, include the correlation id from the response —
the API echoes one on every request and it ties directly to the server-side log.
