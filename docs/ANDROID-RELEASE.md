# Android release runbook

Two Android artifacts exist in this repo. **They ship the same package id,
`com.furlpay.app`, so only one can ever own the Play listing.**

| | `apps/android` | `native-app` |
|---|---|---|
| What | Bubblewrap TWA (a browser frame around furlpay.com) | Expo / React Native native app |
| versionCode | 1 | 3 |
| Status | **DEPRECATED** — build.sh refuses to run without `FURLPAY_BUILD_DEPRECATED_TWA=1`; outputs are named `twa-DEPRECATED-DO-NOT-UPLOAD.*` | **the app we ship** |

The TWA's Java is stock Bubblewrap with no custom logic, so nothing is lost by
retiring it. Its keystore, however, is the one thing that must survive: it is
the identity of `com.furlpay.app`. (It now lives outside the repo — see
`apps/android/KEYSTORE-MOVED.md`.)

Because both artifacts are signed with the **same upload key**, a TWA bundle
uploaded by mistake would pass `jarsigner -verify`, `verify-assetlinks.sh` and
Play's own signature check, then silently replace the real app with a browser
frame that has no FCM, no native passkeys and no in-app account deletion. That
is why the TWA build is now gated rather than merely documented as superseded.
See `apps/android/DEPRECATED.md`.

## The one rule

**Every build of `com.furlpay.app`, forever, must be signed with the upload key
(alias `furlpay`) — now at `C:\Users\ashut\.furlpay-signing\android.keystore`,
outside the OneDrive-synced repo. See `apps/android/KEYSTORE-MOVED.md`.**

Its certificate — `15:92:F6:8B:…:29:13` — is the fingerprint published in
`apps/web/public/.well-known/assetlinks.json`. If a build is signed with any
other key:

- Play rejects the upload as a different app, and
- Digital Asset Links verification fails, which is **silent**: the TWA renders
  with a browser URL bar and the RN app stops opening `furlpay.com` deep links.
  Nothing logs an error. You just ship a broken app.

By default EAS generates a *fresh* keystore, which would do exactly that. That
is why `eas.json` sets `credentialsSource: "local"` and `credentials.json`
points at the existing keystore. Do not remove either.

## Building

```bash
# Preferred: EAS cloud build (uses credentials.json → the upload key)
cd native-app && npm run build:android

# Local, Windows-friendly (eas build --local does not support Windows)
cd native-app && ./build-android.sh      # → furlpay-mobile-release.aab (~61 MB)
```

Both paths sign with the upload key. Verified: a local `build-android.sh` run
produces an AAB whose certificate is `15:92:F6:8B:…:29:13` — the fingerprint
published in `assetlinks.json`.

`build-android.sh` exists because two defaults will otherwise bite you.

**1. Expo's generated Gradle debug-signs release builds.**
`android/app/build.gradle` ships `release { signingConfig signingConfigs.debug }`,
so a plain `gradlew bundleRelease` yields a **debug-signed** bundle — rejected by
Play, and never matching assetlinks. The script overrides this with AGP's
`android.injected.signing.*` properties (which take precedence over the
buildType) rather than patching a file that prebuild and EAS both regenerate.
Confirm at any time with:

```bash
cd native-app/android && ./gradlew :app:signingReport   # look for Variant: release
```

**2. Windows MAX_PATH.** RN's New Architecture codegen names C++ object files
after their full source path, which from `OneDrive/Documents/Payment App/…`
exceeds 260 chars — ninja fails with *"Filename longer than 260 characters"*.
The script `subst`s a drive at the repo root and builds from `X:\native-app\android`.

Two obvious fixes that **do not** work, so don't retry them:
- a **directory junction** — Gradle canonicalises it back to the real long path;
- `subst`ing at `native-app` itself — the project then sits at a drive root
  (`X:\`) and Metro fails with `Unable to resolve module ./node_modules/expo-router/entry.js`.

None of this affects EAS, which builds on Linux.

## Verify before every upload

```bash
apps/android/verify-assetlinks.sh native-app/furlpay-mobile-release.aab
```

This extracts the artifact's signing certificate and checks it against the live
`assetlinks.json`. It fails loudly on the mismatch that otherwise fails silently.

## ⚠️ The Play App Signing step — NOT yet done

This is the live blocker. **Google Play re-signs your upload with its own App
Signing key.** The certificate on real users' devices is therefore *not* the one
in `android.keystore`; ours becomes only the *upload* key.

`assetlinks.json` currently lists **only the upload key**. That is why
`verify-assetlinks.sh` passes today against a sideloaded APK — and why app links
will nonetheless break for every Play user the moment the app goes live.

After the first Play upload:

1. Play Console → **Test and release → App integrity → App signing key
   certificate** → copy the **SHA-256 fingerprint**.
2. Add it to the `sha256_cert_fingerprints` array in
   `apps/web/public/.well-known/assetlinks.json`, **keeping the upload key
   alongside it** (sideloaded/internal-test builds still use the upload key):

   ```json
   "sha256_cert_fingerprints": [
     "15:92:F6:8B:BE:16:5A:EF:47:EB:51:30:39:A2:64:EF:41:95:6E:52:77:CB:8A:29:9B:9E:AD:E6:C1:13:29:13",
     "<PLAY APP SIGNING SHA-256 GOES HERE>"
   ]
   ```
3. Deploy the web app, then confirm:
   `curl https://furlpay.com/.well-known/assetlinks.json`
4. Install from Play and confirm the TWA/app opens `furlpay.com` links without a
   URL bar.

## Keystore custody

**Done (2026-07-14):** `android.keystore` and `signing-credentials.txt` have been
moved OUT of the OneDrive-synced repo tree to `C:\Users\ashut\.furlpay-signing\`
— a cloud-replicated signing key plus a plaintext password is a custody risk for
a financial app. `build-android.sh` reads them from there
(`FURLPAY_SIGNING_DIR` / `FURLPAY_KEYSTORE_PASSWORD` override). Details and the
verification that was done: `apps/android/KEYSTORE-MOVED.md`.

**Still on you:** keep an offline backup of that directory (password manager /
encrypted drive). If this key is lost, `com.furlpay.app` can never be updated
again — the listing is dead and users must reinstall a differently-named app. It
is the single most irreplaceable file in the project, and it is now in exactly
one place.

---

## Local cache encryption decision (2026-07-14)

**Decision: ship with the current unencrypted SQLite cache.** Do not migrate to
`@op-engineering/op-sqlite` for this release.

`native-app/lib/db.ts` (`furlpay.db`) is not encrypted at rest — `expo-sqlite`
exposes no SQLCipher key. That is acceptable **only** while it holds no secrets,
so here is the actual classification, taken from the code rather than assumed:

| Table | Contents | Class |
|---|---|---|
| `kv_cache` | JSON snapshots of GET responses (overview, cards, portfolio) | Financial metadata + personal data |
| `transactions` | id, category, direction, title, amount_usd, token, chain, card_id, status | Financial transaction metadata |
| `pending_actions` | offline outbox: destination, amount, token, chain, idempotency key | Financial metadata — **not a bearer instrument** |
| `notifications` | title, body, url | Cosmetic inbox |

What is **not** there, verified by grep and pinned by
`lib/__tests__/dbClassification.test.ts`:

- session JWT → `SecureStore` (`furlpay.session`), Android Keystore
- wallet mnemonic → `SecureStore` (`furlpay.wallet.mnemonic`) with
  `requireAuthentication: true` + `WHEN_UNLOCKED_THIS_DEVICE_ONLY` (biometric-gated)
- no private key, TOTP secret, API key or passkey secret anywhere in SQLite

The outbox is worth calling out because it is the one thing that *looks* like it
might authorize money: it does not. The queued transfer payload carries
`demoSignature()`, which `app/(tabs)/send.tsx` documents as a random blob that
**authorizes nothing** — replay still requires the session JWT from SecureStore.
The real signing path (`signTransferAuthorization`, genuine EIP-3009) is
biometric-gated and is never queued.

**Therefore:** exfiltrating `furlpay.db` from a rooted device yields transaction
*history* and balances — a privacy loss, not an account takeover. Nothing in it
can authenticate, sign, or move funds. `allowBackup=false` plus the data-extraction
rules already stop the DB leaving the device via backup or `adb`.

**Why not encrypt anyway:** swapping to `@op-engineering/op-sqlite` is a native
dependency change. It alters the native module set (requiring re-verification of
16 KB page alignment), and forces a re-test of the offline outbox and the
`user_version` migration chain — on an app that has never shipped, weeks before a
target-SDK deadline. That is a materially destabilizing change for a residual risk
that is already contained.

**Revisit when** either of these becomes true — both invalidate the reasoning above:

1. A real signed authorization is ever persisted (i.e. `/wallets/transfer` is
   wired to real settlement and `send.tsx` switches to `signTransferAuthorization`
   — the code comment there already flags this), or
2. the cache starts holding full PII.

`dbClassification.test.ts` fails the build if a secret-shaped column appears in
`db.ts`, so this decision cannot rot silently.
