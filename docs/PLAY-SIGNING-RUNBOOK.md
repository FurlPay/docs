# Play signing chain — runbook

Everything in this file is **blocked on values that only exist after the first
Play upload**. No fingerprint here is invented, and none may be guessed: a wrong
fingerprint in `assetlinks.json` fails exactly like a missing one, but silently
and with more confidence.

## What is broken until this runbook is executed

The app has never been uploaded, so Google has never seen a certificate for
`com.furlpay.app`. Three things depend on that, and **all three fail silently**:

| Depends on | Symptom when unregistered |
|---|---|
| **Native passkeys** (`delegate_permission/common.get_login_creds`) | Passkey sign-in fails in the Play build. This is the app's primary auth. |
| **App Links** (`autoVerify=true` for furlpay.com) | `https://furlpay.com/...` opens in the browser, not the app. Approval deep links from FCM land in Chrome. |
| **Google Sign-In** (Android OAuth client) | The button does nothing / returns `DEVELOPER_ERROR`. |

Current state, verified:

```
$ curl -s https://furlpay.com/.well-known/assetlinks.json | grep -c 'SHA-256 fingerprints'
1        # the UPLOAD key only
$ node -e "console.log(require('./native-app/google-services.json').client[0].oauth_client.length)"
0        # NO Android OAuth client registered in Firebase
```

## Why one fingerprint is not enough

You sign the AAB with the **upload key**. Google Play then **re-signs it with the
Play App Signing key** before it reaches a device. The certificate a real user's
phone presents is therefore *not* the one in `assetlinks.json` today.

```
  local APK  ──signed by──>  upload key       ──> in assetlinks.json ✅
  Play build ──signed by──>  Play App Signing ──> NOT in assetlinks.json ❌  <-- users
```

`apps/android/verify-assetlinks.sh` will still print **OK** for your local
artifact while every Play user is broken. It now prints an explicit WARNING when
only one fingerprint is published, precisely because that check is otherwise
reassuring and wrong.

## The sequence

**1. Upload the canonical AAB to Play Internal Testing.**

```bash
cd native-app && ./build-android.sh       # -> furlpay-mobile-release.aab
```

Upload `native-app/furlpay-mobile-release.aab`.
Do **not** upload anything from `apps/android` — see `apps/android/DEPRECATED.md`.

**2. Read the Play App Signing certificate.**

Play Console → **Test and release → Setup → App integrity → App signing key
certificate**. Copy **both**:

- `SHA-256` — for `assetlinks.json` (App Links + passkeys)
- `SHA-1` — for Firebase / Google Sign-In

Also copy the **Upload key certificate** SHA-1 from the same page (you need it in
Firebase too, so that locally-installed debug/release builds keep working).

**3. Add the Play SHA-256 to `assetlinks.json` — do not replace the existing one.**

`apps/web/public/.well-known/assetlinks.json`:

```json
"sha256_cert_fingerprints": [
  "15:92:F6:8B:BE:16:5A:EF:47:EB:51:30:39:A2:64:EF:41:95:6E:52:77:CB:8A:29:9B:9E:AD:E6:C1:13:29:13",
  "<PASTE PLAY APP SIGNING SHA-256 HERE>"
]
```

Keeping the upload key is deliberate: it is what locally-installed APKs are
signed with, and dropping it would break App Links and passkeys for every
sideloaded test build.

**4. Register the SHA-1 fingerprints in Firebase.**

Firebase Console → project **furlpay-production** → Project settings → Your apps →
Android app `com.furlpay.app` → **Add fingerprint**. Add **both**:

- Play App Signing SHA-1  (Play-installed users)
- Upload key SHA-1        (local/sideloaded builds)

This is what creates the **Android OAuth client** that `lib/oauth.ts` needs.
Without it, `oauth_client` in `google-services.json` stays `[]` and Google
Sign-In cannot work — no amount of client-side code fixes it.

**5. Re-download `google-services.json`** (it now contains the `oauth_client`
entries) into `native-app/google-services.json`, and update the EAS file secret:

```bash
eas secret:delete --scope project --name GOOGLE_SERVICES_JSON
eas secret:create --scope project --type file \
  --name GOOGLE_SERVICES_JSON --value ./google-services.json
```

**6. Set the OAuth client ids** (from Firebase → the new Android + Web clients):

```bash
eas secret:create --scope project --name EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID --value <android-client-id>
eas secret:create --scope project --name EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID     --value <web-client-id>
```

Also add the Android client id to the server's `GOOGLE_CLIENT_ID` allow-list
(`apps/web/src/lib/auth/google.ts` accepts a comma-separated list — the Android
id must be in it, or the server rejects the id_token the app presents).

**7. Rebuild and redeploy.**

```bash
cd native-app && ./build-android.sh                 # rebuild with the new config
cd .. && npx vercel deploy --prod --yes             # publish the new assetlinks.json
```

**8. Verify — on a Play-installed build, not a sideloaded one.**

```bash
./apps/android/verify-assetlinks.sh native-app/furlpay-mobile-release.aab
# must now print TWO published fingerprints and no WARNING
```

Then on a device that installed **from Play Internal Testing**:

- [ ] Passkey sign-up and sign-in
- [ ] Google Sign-In (new user AND existing user)
- [ ] `https://furlpay.com/approvals` opens the app, not Chrome (app foregrounded, backgrounded, and killed)
- [ ] An FCM push arrives and its tap lands on `/approvals`
- [ ] Account deletion (Profile → Delete account) completes, and the old session cannot be reused

Verify App Links verification actually passed:

```bash
adb shell pm get-app-links com.furlpay.app     # expect: verified
```

## Why this cannot be automated here

Steps 2 and 4 require signing in to Play Console and Firebase Console as the
account owner. The fingerprints do not exist anywhere else, and inventing one
produces exactly the same silent failure it is supposed to prevent.
