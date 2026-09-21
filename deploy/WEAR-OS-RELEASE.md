# Wear OS release runbook

> Surface: `guardian/app/wear` — Kotlin, part of the FurlPay × Life Guardian
> multi-module Gradle project (phone "brain" + watch companion).

---

## 1. The identity rule

```kotlin
namespace     = "com.furlpay.guardian.wear"
applicationId = "com.furlpay.guardian"   // SAME as the phone app
minSdk        = 30
targetSdk     = 35
```

The watch app's `applicationId` is **identical to the phone app's**, and the
build file says so explicitly:

```kotlin
// No applicationIdSuffix — the watch app must keep the same id as …
```

This is not an oversight to tidy up. Google Play pairs a Wear OS app to its
phone app by application id. Add a suffix and Play treats the watch build as a
separate, unrelated app: it will not be offered automatically to a paired
watch, and the two become independently versioned strangers.

> **Never add an `applicationIdSuffix` to the wear module.** A debug build
> that needs to coexist on a device is the one legitimate exception, and it
> must never reach a release track.

`minSdk = 30` is Wear OS 3. Below that is Wear OS 2, which has a different
packaging model and is not supported by this build.

---

## 2. Signing

The watch APK and the phone APK must be signed with the **same key**. A
mismatch is not a warning — Play rejects the upload, because a shared
application id signed by two different keys is exactly what an app-takeover
attempt looks like.

`guardian/keystore.properties.example` documents the expected shape. The real
`keystore.properties` is never committed.

Custody rules mirror the Android runbook — see
[Play Store Signing](../docs/PLAY-SIGNING-RUNBOOK.md). Once Play App Signing
is enabled, the upload key and the app-signing key are different keys, and
both SHA-256 fingerprints matter downstream:

- `assetlinks.json` for app links
- `FURLPAY_ANDROID_CERT_SHA256` for WebAuthn origins, where a native caller
  presents `android:apk-key-hash:` rather than an HTTPS origin

Miss the Play app-signing fingerprint and passkeys fail on real devices while
working perfectly on the build machine — because the build machine's APK is
signed with the upload key, and Play re-signs it.

---

## 3. Build

```bash
cd guardian
./gradlew :app:wear:assembleRelease     # APK, local verification
./gradlew :app:wear:bundleRelease       # AAB, for Play
```

Play accepts Wear OS apps in two shapes. Choose deliberately:

| Shape | When |
|---|---|
| Single app bundle with a wear module | phone + watch versioned and released together |
| Separate Wear APK on its own track | watch iterates independently |

This project is one Gradle project producing both, so the **single bundle** is
the default and the one the shared `applicationId` above assumes.

---

## 4. Tracks

| Track | Purpose |
|---|---|
| Internal testing | up to 100 testers, immediate — where every build starts |
| Closed | wider testing on real watch hardware |
| Open | public beta |
| Production | staged rollout, 5% → 20% → 50% → 100% |

Wear OS builds must be tested on **physical watch hardware**. The emulator does
not reproduce complication rendering, tile refresh behaviour, ambient-mode
transitions or the real power envelope — and this product's whole value on the
wrist is that it works when the wearer is not looking at it.

---

## 5. What Play review checks

- **Wear OS quality guidelines** — tiles, complications and ambient mode all
  behave, and the app is usable without the phone in reach.
- **No phone-sized UI.** A scaled-down phone layout is the most common
  rejection.
- **Permissions** justified independently of the phone app.
- **Health/financial data** handling declared in the Data Safety form.

The M3 Expressive pass (morphing, EdgeButtons, QR scan mode, `singleTop`
routing) was verified on an emulator; re-verify on hardware before a
production track.

---

## 6. Environment matrix

| | Debug | Internal | Production |
|---|---|---|---|
| Backend | staging | staging | production |
| Signing | debug key | upload key | upload key → Play re-signs |
| `FURLPAY_ENV` | development | staging | production |
| Money | simulated | simulated | real |

---

## 7. Checklist

- [ ] `applicationId` matches the phone app — **no suffix**
- [ ] Watch and phone signed with the same key
- [ ] Both SHA-256 fingerprints (upload + Play app-signing) registered in
      `assetlinks.json` **and** `FURLPAY_ANDROID_CERT_SHA256`
- [ ] Tested on physical watch hardware, not only the emulator
- [ ] Tiles, complications and ambient mode verified
- [ ] Backend URL matches the intended environment
- [ ] Data Safety form updated
- [ ] Staged rollout, not 100% on day one
