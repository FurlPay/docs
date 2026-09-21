# iOS release runbook

> Surface: `native-app/` — Expo / React Native, built with EAS.
> Companion Apple surfaces: `ios/FurlPayGuardian` (Swift) with
> `FurlPayGuardianWatch`, `FurlPayGuardianWidgets` and
> `FurlPayGuardianActivities`.

---

## The blocker, stated first

**iOS cannot ship today.** There is no paid Apple Developer account, and
therefore no distribution certificate, no provisioning profile, no
`ascAppId` and no `appleTeamId`.

`native-app/eas.json` records this deliberately rather than papering over it:

```jsonc
"ios": {
  "//": "BLOCKED ON AN APPLE DEVELOPER ACCOUNT. ascAppId, appleTeamId and
         either an App Store Connect API key or an Apple ID must be supplied
         before `eas submit -p ios` can run. They are deliberately absent
         rather than filled with placeholders: a wrong ascAppId submits this
         binary to somebody else's app record."
}
```

That reasoning is worth preserving. A placeholder `ascAppId` is not a
harmless stub — App Store Connect will accept the upload against whichever
app record the id actually names.

Until the account exists, the `preview` profile builds the **simulator**
target. That proves the native project compiles; it does not prove signing,
entitlements or store delivery work.

---

## 1. Prerequisites

| | Status |
|---|---|
| Apple Developer Program (paid) | ❌ blocking |
| Distribution certificate | ❌ depends on the above |
| Provisioning profile | ❌ |
| App Store Connect app record + `ascAppId` | ❌ |
| `appleTeamId` | ❌ |
| App Store Connect API key (for CI submission) | ❌ |
| Bundle identifier reserved | ❌ |

**Builds require macOS.** The Swift guardian targets have never been compiled
on Windows and cannot be — macOS CI is the compiler. EAS cloud builds supply
that; a local `eas build --local` does not work on Windows either.

---

## 2. SDK requirement

Apple requires apps be built against a current iOS SDK, and the floor moves.
`eas.json` documents the rule as it applied at the time of writing (iOS 18 SDK
/ Xcode 16 since 2025-04-24), and the current floor is **Xcode 26**.

> **Check the current SDK floor before every store submission.** This is the
> single most common cause of a rejected upload that built fine locally, and
> it changes on Apple's schedule, not yours.

EAS pins a default Xcode image per Expo SDK version. When the floor moves,
either the Expo SDK upgrade or an explicit image pin in `eas.json` is
required — verify which, rather than assuming the default is current.

---

## 3. Build

```bash
cd native-app

# Compile check, no signing required — works without a developer account.
eas build --profile preview --platform ios      # simulator target

# Store build — requires the full credential set above.
eas build --profile production --platform ios   # Release configuration
```

The `production` profile uses `credentialsSource: "local"` and
`autoIncrement: false`. Build numbers are managed explicitly; App Store
Connect rejects a duplicate, so bump deliberately as part of the release.

---

## 4. Submit

```bash
eas credentials -p ios     # first: establish certificate + profile
# then set ascAppId and appleTeamId in eas.json
eas submit -p ios --profile production
```

Then in App Store Connect: TestFlight internal → TestFlight external (requires
Beta App Review) → App Store review.

---

## 5. What review will ask about

This is a financial app that holds keys and moves stablecoins. Budget for
these, because they are the ones that generate rejections rather than
questions:

- **Account deletion must be in-app.** Required for any app with account
  creation. The codebase has an account-deletion path; verify it is reachable
  from the UI, not only from the API.
- **Sign in with Apple** must be offered wherever a third-party social login
  is. FurlPay offers Google, so Apple is mandatory — it is implemented
  (`AppleSignInButton`, `/api/auth/apple`), and note that the identity token
  is verified server-side against Apple's JWKS, which is what review
  expects.
- **Passkeys / biometrics.** The private key never leaves the Secure Enclave;
  Face ID / Touch ID gates its use. Say so plainly in review notes — it is a
  credit, not a risk, but only if explained.
- **Crypto and financial services** attract extra scrutiny on custody claims.
  The [Custody Disclosure](../docs/CUSTODY-DISCLOSURE.md) and
  [Balance Custody Map](../docs/BALANCE-CUSTODY-MAP.md) exist precisely so
  these answers are consistent. Do not improvise them.
- **Demo account** for review. `NEXT_PUBLIC_ALLOW_GUEST` exists for exactly
  this. It is disabled in production by default and must stay that way — the
  guest route already refuses outside preview.
- **Privacy manifest** (`PrivacyInfo.xcprivacy`) and the required-reason API
  declarations. Third-party SDKs need their own.
- **Export compliance.** The app uses encryption; the exemption declaration
  still has to be answered.

---

## 6. Environment matrix

| | Simulator | TestFlight | App Store |
|---|---|---|---|
| Profile | `preview` | `production` | `production` |
| Backend | staging | staging | production |
| `FURLPAY_ENV` | development | staging | production |
| Money | simulated | simulated | real |
| Signing | none | distribution | distribution |

The API base URL is a build-time constant. A TestFlight build pointed at
production is indistinguishable from a store build to the person holding the
phone — set it deliberately per profile.

---

## 7. The watch and widget targets

`ios/FurlPayGuardian` is a separate Swift project (XcodeGen, `project.yml`)
containing the phone app, `FurlPayGuardianWatch`, `FurlPayGuardianWidgets` and
`FurlPayGuardianActivities`.

watchOS ships **as part of the iOS app record**, not separately — the watch
app is embedded in the iOS bundle and submitted with it. It therefore inherits
every blocker above and adds:

- Its own bundle id under the phone app's
- A matching provisioning profile
- Independent review of any Live Activity / complication behaviour

Widgets and Live Activities are reviewed with the app but are a common source
of rejection when they display data the app has not justified collecting.

---

## 8. Checklist

- [ ] Apple Developer account active
- [ ] Certificate + provisioning profile created via `eas credentials`
- [ ] `ascAppId` and `appleTeamId` set — **verified against the real app record**
- [ ] Current Xcode/SDK floor confirmed against Apple's published requirement
- [ ] Build number bumped
- [ ] Backend URL matches the intended environment
- [ ] Account deletion reachable in-app
- [ ] Sign in with Apple present alongside Google
- [ ] Privacy manifest and export compliance answered
- [ ] Review notes explain the custody model and the Secure Enclave design
