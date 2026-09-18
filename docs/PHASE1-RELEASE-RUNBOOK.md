# Phase 1 (Self-Custodial Wallet Linking) — Release Runbook

> Executes the release path after engineering hardening passed. Order:
> **staging Supabase → production migration → rebuild AAB → Play internal →
> real-device acceptance → enable Stage 1 → observe → Phase 2.**
> Prereqs proven: migration 0005 applied to real Postgres 18.4, 3 concurrency
> races + cold hydration, web 25/25, native 26/26 (commit `8080a92`).
> `FURLPAY_SELF_CUSTODY_STAGE` stays **0** until step 5.

---

## A. Staging Supabase validation (do first)

**Environment:** a NON-PRODUCTION Supabase project. Do not use production as the
first Supabase-specific test. If none exists, create one (same region/plan as
prod for fidelity).

**A1. Apply the schema** (0001 must exist for the `users` FK; 0005 is the new one):
```bash
# via Supabase CLI (preferred)
supabase link --project-ref <STAGING_REF>
supabase db push          # applies supabase/migrations/*.sql in order
# — or paste supabase/migrations/0005_linked_wallets.sql into the SQL editor
```

**A2. Schema + concurrency (direct Postgres):** get the staging connection string
(Project → Settings → Database), then:
```bash
npm i pg    # if not present
DATABASE_URL='postgres://...staging...' node scripts/db/link-wallet-race.mjs
```
Expect: Race A → one owner per wallet; Race B → one wallet per user; Race C (8×)
→ one row, no errors; cold hydration PASS. (Proven on ephemeral PG 18.4.)

**A3. Supabase transport (PostgREST/RPC + role permissions):**
```bash
SUPABASE_ENV=staging \
SUPABASE_URL=https://<STAGING_REF>.supabase.co \
SUPABASE_SERVICE_ROLE_KEY=<staging service key> \
SUPABASE_ANON_KEY=<staging anon key> \
node scripts/db/supabase-validate.mjs
```
Verifies through the REAL transport: `link_wallet()` / `get_linked_wallet()` via
RPC, cross-account `conflict_address`, idempotency, zero-address rejected through
RPC, cold hydration, and **anon key CANNOT execute the RPC or read the table**.
The script refuses `SUPABASE_ENV=production` and prints only the host.

**A4. Manual checks (Supabase dashboard):** table + PK + per-user unique index +
CHECK constraints exist; RLS enabled with **zero policies**; function grants show
`service_role` only; no secrets in logs.

**Pass criteria:** A2 + A3 green and A4 confirmed → staging validated.

---

## B. Production migration (only after A passes + explicit approval)

Applying the schema does **not** expose the feature — the flag stays 0, so this
is a safe dormant deploy.

```bash
supabase link --project-ref <PROD_REF>
supabase db push        # or paste 0005 into the prod SQL editor
```

**Post-migration verify (prod, read-only):** table/constraints/RLS/grants exist;
existing routes unaffected — smoke `GET /api/overview` (401 unauth), auth still
works, and confirm **no balance / deposit / settlement behavior changed** (this
migration adds a table + 2 functions and touches nothing else).

**Rollback (non-destructive — no other table references it):**
```sql
drop function if exists get_linked_wallet(uuid);
drop function if exists link_wallet(uuid, text, text, text);
drop table if exists linked_wallets;
```

---

## C. Rebuild the production AAB

The current signed AAB **no longer exists** (deleted to reclaim disk) and
predated Phase 1 regardless — a fresh build from the latest commit is required.
It must contain: `lib/wallet.ts` (EIP-191 `signMessage`, `linkDevice`,
`sendGasless`), `app/wallet.tsx` (link UI), session-bound step-up handling.

**Prereqs:** keystore relocated to the local-only `C:\Users\ashut\.furlpay-signing\`
(done 2026-07-14 — SHA-256-verified copy, originals removed from the OneDrive
tree; see `apps/android/KEYSTORE-MOVED.md`). Build:
```bash
cd native-app && ./build-android.sh        # local, or: npm run build:android (EAS)
apps/android/verify-assetlinks.sh native-app/furlpay-mobile-release.aab
```
**Validate the release/minified build** (not dev mode): Hermes release bundle;
R8/ProGuard keep-rules didn't strip react-native-passkey/expo modules; SecureStore
biometric access; mnemonic decrypt; EIP-191 signing; passkeys; app links;
API base = `https://furlpay.com/api` (default in `lib/api.ts`, no override); no
debug endpoints / embedded secrets; `allowBackup=false` honored.

---

## D. Real-device Phase 1 acceptance checklist

Install the **Play-distributed** artifact (internal track), not a local APK —
Play App Signing/app-links/passkeys can differ. Then:

**Happy path (must pass):**
- [ ] Fresh account → authenticate → create device wallet (address shown).
- [ ] Complete fresh OTP/passkey step-up in the same session.
- [ ] Tap "Link this device" → biometric prompt → mnemonic decrypts → EIP-191 signs.
- [ ] Server recovers the correct address, consumes the challenge, `link_wallet()` stores the row.
- [ ] App shows the verified linked address.
- [ ] Kill + relaunch app → linked status reloads from durable storage (cold hydration).

**Failure / lifecycle (must behave safely):**
- [ ] Cancel biometric → no link, clear message.
- [ ] Wrong biometric → no link.
- [ ] Expired challenge (wait >5 min) → rejected.
- [ ] Network loss before signing → no partial state.
- [ ] Network loss after signing, before response → retry is safe (nonce single-use).
- [ ] Double-tap Link → exactly one link, no duplicate.
- [ ] App backgrounded / process-killed mid-link → no corrupt state on return.
- [ ] Reinstall app → wallet gone unless recovered from phrase (self-custody).
- [ ] Second device / different address → **409**, no silent replace.
- [ ] Same address, another account → **409** (cross-account).
- [ ] Missing step-up → **403 stepup_required**; step-up from another session → **403**.
- [ ] Expired step-up → **403**.

**Observability during the run:** challenge_issued / link_established /
link_rejected audit events present; owner email sent; replay + wrong-session
rejections logged; **no private key, mnemonic, or raw token in any log**; no
unexpected 500s.

---

## E. Enable Stage 1 (only after C+D on a real device)

Set `FURLPAY_SELF_CUSTODY_STAGE=1` (staging/internal-allowlist first if possible),
re-run the device happy path, then observe the signals in D before considering
Phase 1 complete in production. **Do not** enable before the new AAB is
distributed — the shipped app must contain `linkDevice`.

---

## Remaining blockers before this runbook can start
1. A **staging Supabase** project + its service/anon keys (none available in the
   engineering environment — this is the gating item).
2. Explicit approval for the production migration (step B).
3. ~~Keystore moved out of OneDrive~~ (done 2026-07-14); Play internal track set up.
