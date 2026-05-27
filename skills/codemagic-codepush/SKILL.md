---
name: codemagic-codepush
version: 1.0.0
description: >
  Agentic skill for handling React Native CodePush (OTA updates) on Codemagic.
  Use when a customer reports issues with CodePush setup, releasing updates, rollbacks,
  access tokens, CI integration, or SDK configuration. Also use for how-to questions
  about OTA updates, deployment management, or production rollout strategies.
  Covers migration from AppCenter CodePush and Expo Updates (EAS Update).
---

## Version Check

**Run this every time the skill is loaded, before doing anything else.**

1. Fetch `https://agents.codemagic.io/skills/codepush-skill.md`
2. If the fetch succeeds → **use the fetched content as the active skill for the entire session**. Ignore everything below in this local file — the hosted version is the source of truth.
3. If the fetch fails (network error, URL unreachable) → fall back to this local file, and notify the user:
   > "Could not reach https://agents.codemagic.io/skills/codepush-skill.md — running local version v1.0.0 which may be outdated."
   Then proceed with the local file content below.

---

# Codemagic CodePush — Agent Skill

## What CodePush Is

CodePush is Codemagic's over-the-air (OTA) update service for React Native apps. It allows teams to push JavaScript bundle and asset updates directly to users without going through the App Store or Google Play.

**Server URL:** `https://codepush.pro/`
**CLI package:** `@codemagic/code-push-cli`
**SDK package:** `@code-push-next/react-native-code-push`

---

## What Can and Cannot Be Updated

| Can update via CodePush | Requires a new binary (store release) |
|---|---|
| JavaScript bugs and logic | Native module changes |
| UI, styling, layout | React Native version upgrades |
| Bundled assets (images, fonts) | Native config changes (permissions, entitlements) |
| Feature flags | Platform-specific native features |
| Performance improvements | |

**Key rule for customers:** If the change touches native code (Swift, Kotlin, Xcode config, Gradle), CodePush cannot deliver it — a new store binary is required.

---

## Core Concepts

- **App**: Registered per platform (`MyApp-Android`, `MyApp-iOS`). Keep platforms separate.
- **Deployment**: A release channel within an app. Default deployments are `Staging` and `Production`. Additional ones can be created.
- **Deployment key**: Embedded in the mobile binary at build time; directs the SDK to the correct deployment channel.
- **Update**: A JavaScript bundle + assets released to a deployment. Versioned and tied to a `targetBinaryVersion` range.

**How updates reach users:**
1. App launches → SDK checks CodePush server for updates
2. If available, bundle is downloaded and stored locally
3. Update applied on restart / resume / immediately (configurable)
4. If update crashes before calling `notifyAppReady()`, it auto-reverts to the previous bundle

---

## Setup

### 1. Install and authenticate CLI

```bash
npm install -g @codemagic/code-push-cli
code-push login "https://codepush.pro/" --accessKey $ACCESS_TOKEN
```

> `--accessKey` / `--access-key` is optional — you can also run `code-push login` without the flag and provide the token interactively when prompted. Source: https://docs.codemagic.io/rn-codepush/setup/#authenticate-the-cli

> Access tokens are self-service. Customers generate them via the Codemagic dashboard: **OTA Updates → Manage Access Keys → Generate key**. Enter a name and expiration — the key is shown **once**, copy it before closing. Keys can also be revoked from the same screen.

### 2. Create apps (one per platform)

```bash
code-push app add MyApp-Android
code-push app add MyApp-iOS
```

Each app gets `Staging` and `Production` deployments automatically.

### 3. Get deployment keys

```bash
code-push deployment list <app_name> -k
```

### 4. Install the SDK

```bash
npm install @code-push-next/react-native-code-push
```

### 5. Configure native projects

**iOS**

Install CocoaPods dependencies:

```bash
cd ios && pod install && cd ..
```

Add to **Info.plist:**
```xml
<key>CodePushServerURL</key>
<string>https://codepush.pro/</string>
<key>CodePushDeploymentKey</key>
<string>YOUR_IOS_DEPLOYMENT_KEY</string>
```

Update **AppDelegate** to load the JS bundle from CodePush instead of the embedded binary. This is what makes OTA updates take effect.

**Objective-C (`AppDelegate.m`)** — add import at the top:
```objc
#import <CodePush/CodePush.h>
```

Replace the existing bundle URL method with:
```objc
- (NSURL *)sourceURLForBridge:(RCTBridge *)bridge
{
  #if DEBUG
    return [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index"];
  #else
    return [CodePush bundleURL];
  #endif
}
```

**Swift (`AppDelegate.swift`)** — add import at the top:
```swift
import CodePush
```

Replace the existing `bundleURL()` method with:
```swift
override func bundleURL() -> URL? {
  #if DEBUG
    RCTBundleURLProvider.sharedSettings().jsBundleURL(forBundleRoot: "index")
  #else
    CodePush.bundleURL()
  #endif
}
```

In both cases: in `DEBUG` the app still loads from the Metro bundler as normal; in release builds it loads from CodePush (falling back to the embedded bundle if no update is installed).

**Android**

Add to `android/app/build.gradle` (at the top, below other `apply from` lines):
```gradle
apply from: "../../node_modules/@code-push-next/react-native-code-push/android/codepush.gradle"
```

Add to `android/app/src/main/res/values/strings.xml`:
```xml
<string moduleConfig="true" name="CodePushServerUrl">https://codepush.pro/</string>
<string moduleConfig="true" name="CodePushDeploymentKey">YOUR_ANDROID_DEPLOYMENT_KEY</string>
```

Update `MainApplication` to override the JS bundle file path. CodePush serves the updated bundle; without this override the app always loads the embedded binary bundle.

**Kotlin — React Native 0.76 (`MainApplication.kt`):**
```kotlin
import com.microsoft.codepush.react.CodePush

class MainApplication : Application(), ReactApplication {
    override val reactNativeHost: ReactNativeHost =
        object : DefaultReactNativeHost(this) {
            override fun getPackages(): List<ReactPackage> = PackageList(this).packages

            override fun getJSBundleFile(): String {
                return CodePush.getJSBundleFile()
            }
        }
}
```

**Kotlin — React Native 0.82+ (`MainApplication.kt`):**
```kotlin
import com.microsoft.codepush.react.CodePush

class MainApplication : Application(), ReactApplication {
    override val reactHost: ReactHost by lazy {
        getDefaultReactHost(
            context = applicationContext,
            packageList = PackageList(this).packages,
            jsBundleFilePath = CodePush.getJSBundleFile(),
        )
    }
}
```

### 6. Wrap root component and configure JS-side SDK

**Basic setup — default behaviour (check on app start, install on next restart):**
```javascript
import codePush from '@code-push-next/react-native-code-push';

function App() { /* ... */ }
export default codePush(App);
```

**With custom sync options:**
```javascript
import codePush, { InstallMode } from '@code-push-next/react-native-code-push';

const codePushOptions = {
  checkFrequency: codePush.CheckFrequency.ON_APP_RESUME,
  installMode: InstallMode.ON_NEXT_RESUME,
  minimumBackgroundDuration: 60, // only restart if backgrounded for 60+ seconds
};

function App() { /* ... */ }
export default codePush(codePushOptions)(App);
```

**Custom sync() flow** — use when you need control over when the check happens (e.g. after login, on a button tap):
```javascript
import codePush from '@code-push-next/react-native-code-push';

async function checkForUpdate() {
  const update = await codePush.checkForUpdate();
  if (update) {
    await update.download();
    await update.install(codePush.InstallMode.ON_NEXT_RESTART);
  }
}
```

**notifyAppReady()** — REQUIRED in any custom flow that does not use the default `codePush(App)` wrapper. Call it after the bundle has loaded successfully. If it is never called, CodePush treats the update as crashed and rolls back on next restart:
```javascript
import codePush from '@code-push-next/react-native-code-push';

// Call this once the app is ready — e.g. in useEffect on the root component
codePush.notifyAppReady();
```

> Note: `notifyAppReady()` is called automatically when using `codePush(App)`. Only add it manually in custom `sync()` flows.

> Source: `@code-push-next/react-native-code-push` SDK. Code examples based on SDK documentation — verify against the npm package README for latest API.

### 7. Verify setup

```bash
code-push release-react <app_name> <ios_or_android>
```

This releases to `Staging` by default.

---

## Migrating from Expo Updates (EAS Update)

Customers moving from Expo Updates to Codemagic CodePush need to understand the concept mapping:

| Expo Updates | CodePush |
|---|---|
| EAS project / `projectId` | CodePush app registration |
| Channel (e.g., production) | Deployment (Staging / Production) |
| `eas update --channel` | `code-push release-react` |
| Runtime version | Target binary version |
| `expo-updates` package | `@code-push-next/react-native-code-push` |

### Migration steps

1. Install CLI and authenticate with CodePush token
2. Register separate apps per platform (`MyApp-iOS`, `MyApp-Android`)
3. Remove `expo-updates` package, EAS config from `app.json`, and native EAS metadata
4. Install `@code-push-next/react-native-code-push`
5. Configure deployment keys in `Info.plist` (iOS) and `strings.xml` (Android)
6. Update native app delegates to load JS bundle from CodePush (same as standard setup — see Step 5 above)
7. Wrap root component with `codePush(App)`
8. Test with `code-push release-react` to Staging
9. Update CI/CD pipeline to use CodePush commands instead of `eas update`
10. Adopt Staging → Production promotion workflow

### Agent checklist for Expo migration tickets

1. Have they removed `expo-updates` and EAS config from `app.json`?
2. Is the server URL `codepush.pro` in `Info.plist` and `strings.xml`?
3. Have they re-registered apps on Codemagic's server? (EAS channels do not carry over)
4. Have they shipped a new store binary with the new deployment keys?
5. Do they understand users on old Expo binaries won't receive CodePush updates until the new binary reaches them?

---

## Migrating from AppCenter CodePush

AppCenter CodePush was shut down in March 2025. Codemagic runs its own independent server at `codepush.pro` — nothing carries over from AppCenter. This is not a redirect or data import.

### What needs to change

**1. Swap the npm package**

```bash
npm uninstall react-native-code-push
npm install @code-push-next/react-native-code-push
cd ios && pod install && cd ..
```

Update all JS imports:
```javascript
// Before
import codePush from 'react-native-code-push';

// After
import codePush from '@code-push-next/react-native-code-push';
```

**2. Swap the CLI**

```bash
npm uninstall -g appcenter-cli
npm install -g @codemagic/code-push-cli
```

**3. Update the server URL in three places**

- `Info.plist` (iOS) — `CodePushServerURL`
- `strings.xml` (Android) — `CodePushServerUrl`
- CLI login command — `code-push login "https://codepush.pro/" --accessKey $TOKEN`

**4. Re-register apps and get new deployment keys**

AppCenter apps and deployment keys do not exist on Codemagic's server. Start fresh:

```bash
code-push app add MyApp-iOS
code-push app add MyApp-Android
code-push deployment list MyApp-iOS -k
```

Update the new keys in `Info.plist` and `strings.xml`.

**5. Ship a new store binary**

This is the critical step customers miss. Until a new binary containing the `codepush.pro` server URL and new deployment keys reaches users, existing installs will never receive OTA updates — the old AppCenter keys are permanently dead. There is no way to migrate existing users without a store release.

### Agent checklist for migration tickets

When a customer says "CodePush stopped working" or "I migrated from AppCenter":

1. Are they still using `react-native-code-push` (old) or `@code-push-next/react-native-code-push` (new)?
2. Is the server URL `codepush.pro` in all three locations?
3. Have they re-registered their apps on Codemagic's server?
4. Have they shipped a new binary with the updated keys?
5. Do they understand users on old binaries are permanently cut off until that new binary reaches them?

---

## Releasing Updates

### Standard release flow: Staging → Production

```bash
# Release to Staging first
code-push release-react MyApp-Android android --deployment-name Staging

# After testing, promote to Production
code-push promote MyApp-Android Staging Production
```

### Key release flags

| Flag | Purpose |
|---|---|
| `--deployment-name` | Target deployment (default: Staging) |
| `--targetBinaryVersion` | Semver range of native binaries this update is compatible with (e.g. `~1.2.0`) |
| `--mandatory` | Force install — users cannot skip |
| `--rollout <n>` | Gradual rollout to n% of users |
| `--description` | Release notes shown in update dialog |

---

## Production Controls

### Gradual rollout

```bash
# Start at 25%
code-push release-react MyApp-Android android --rollout 25

# Increase without a new release
code-push patch MyApp-Android Production --rollout 50
```

**Constraints:**
- Only one active partial rollout per deployment
- Must reach 100% before publishing a new update to the same deployment

### Mandatory updates

Use for security fixes, critical bugs, or breaking changes:

```bash
code-push release-react <app_name> <platform> --mandatory
# Or patch an existing release
code-push patch <app_name> Production --mandatory true
```

`mandatoryInstallMode` in the SDK controls when the mandatory update is actually applied.

### Manual rollback

```bash
code-push rollback <app_name> <deployment_name>
```

**Automatic rollback:** If the app crashes before calling `notifyAppReady()`, CodePush reverts to the previous bundle automatically on next restart.

---

## Security and Access

### Access tokens (CLI / CI authentication)

- Tokens are self-service — generated via Codemagic dashboard: **OTA Updates → Manage Access Keys → Generate key**
- Enter a name and expiration; the key is shown **once** — copy before closing
- Tokens can be revoked from the same screen
- Store tokens as environment variables / CI secrets, never hardcode

```bash
code-push login "https://codepush.pro/" --accessKey $ACCESS_TOKEN
```

### Cryptographic signing (RSA)

For production apps, sign bundles so the SDK verifies authenticity before installing:

```bash
# Generate key pair
openssl genrsa -out codepush_private.key 2048
openssl rsa -in codepush_private.key -pubout -out codepush_public.key
```

- **Private key**: Stored in CI secrets; signs the bundle during `release-react`
- **Public key**: Embedded in the app binary

**iOS — Info.plist:**
```xml
<key>CodePushPublicKey</key>
<string>-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----</string>
```

**Android — strings.xml:**
```xml
<string name="CodePushPublicKey">-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----</string>
```

---

## CI Integration (Codemagic YAML)

```yaml
scripts:
  - name: Release CodePush update
    script: |
      npm install -g @codemagic/code-push-cli
      code-push login "https://codepush.pro/" --accessKey $CODEPUSH_ACCESS_KEY
      code-push release-react MyApp-Android android \
        --deployment-name Production \
        --targetBinaryVersion "~$APP_VERSION" \
        --rollout 10
```

Store `CODEPUSH_ACCESS_KEY` as a Codemagic environment variable (App Settings → Environment variables).

### Release strategies

| Strategy | Trigger |
|---|---|
| Automatic on main merge | Every successful build deploys to Staging |
| Tag-based | Only on version tags (e.g. `v1.2.0`) |
| Manual | Developer triggers pipeline manually |

---

## Analytics

Accessed via Codemagic dashboard → **OTA Updates**.

| Metric | What it tells you |
|---|---|
| Downloads | How many devices received the bundle |
| Successful installs | How many applied it successfully |
| Failed installs | Installation errors — investigate if high |
| Rollout adoption | How update is spreading during gradual rollout |

**Data updates hourly.**

High downloads + low installs = likely installation failure or SDK misconfiguration. Check `notifyAppReady()` call and `targetBinaryVersion` compatibility.

---

## Advanced Sync Options

For customers using custom `sync()` flows instead of the default `codePush(App)` wrapper:

| Option | Description |
|---|---|
| `InstallMode.ON_NEXT_RESTART` | Apply on full restart (default for optional) |
| `InstallMode.ON_NEXT_RESUME` | Apply when app returns from background |
| `InstallMode.IMMEDIATE` | Apply immediately |
| `minimumBackgroundDuration` | Minimum seconds in background before resume-trigger fires |
| `mandatoryInstallMode` | Controls timing of mandatory update installation |
| `downloadProgressCallback` | Receives `receivedBytes` / `totalBytes` for progress UI |
| `updateDialog` | Show native prompt before installing update |
| `appendReleaseDescription` | Show server-side release notes in dialog |
| `disallowRestart()` / `allowRestart()` | Prevent updates interrupting critical flows (e.g. checkout) |
| `getUpdateMetadata()` | Query metadata for current/pending/previous update |
| `isFirstRun` | True on first launch after a new bundle activates |
| `isPending` | True if an update is downloaded but not yet applied |

**Important:** Any custom flow that skips `sync()` MUST call `notifyAppReady()` after the bundle loads, or CodePush will treat the update as a failure and roll back on next restart.

---

## Debugging and Common Issues

### CLI debug tool

```bash
code-push debug android  # requires ADB + connected device
```

Filter device logs by the `[CodePush]` prefix to isolate update events.

### Common issues, error patterns, and fixes

| Log / error pattern | Cause | Fix |
|---|---|---|
| `[CodePush] Checking for update` then silence / no install | `targetBinaryVersion` mismatch — device app version is outside the semver range specified at release time | Re-release with correct `--targetBinaryVersion` (e.g. `~1.2.3`) matching the installed binary |
| `[CodePush] App is up to date` but customer expects an update | Update released to wrong deployment, or device has a different deployment key embedded | Verify deployment key in `Info.plist` / `strings.xml` matches the intended channel; run `code-push deployment list <app> -k` |
| App silently rolls back to previous version on next launch | `notifyAppReady()` was never called — CodePush assumes the update caused a crash | Add `codePush.notifyAppReady()` early in the app lifecycle in any custom sync flow |
| `[CodePush] Update failed to install` | Corrupted bundle, incompatible JS/native, or signing mismatch | Check if RSA signing is enabled — public key in app must match private key used to sign the release |
| `Error: Cannot find module '...'` after OTA update | Native module added in the update — CodePush cannot deliver native changes | Requires a new store binary; CodePush only updates JS and assets |
| `code-push release-react` fails with `not a React Native project` | Command run from wrong directory | Run from project root where `package.json` lives |
| `Invalid semver version` on release | `versionName` (Android `build.gradle`) or `CFBundleShortVersionString` (iOS `Info.plist`) is not valid semver | Fix to valid semver format e.g. `1.2.3` — do not use `1.0` or `1.0.0.0` |
| `Unauthorized` / `403` on CLI login or release | Access token expired or revoked | Customer can generate a new token self-service: Codemagic dashboard → **OTA Updates → Manage Access Keys → Generate key** |
| New release blocked: `Cannot publish updates when a rollout is in progress` | A partial rollout (< 100%) is still active on that deployment | Patch the existing rollout to 100% first: `code-push patch <app> Production --rollout 100`, then publish |
| `[CodePush] An update is available but it is not targeting the binary version of your app` | Device is on a binary version outside the `targetBinaryVersion` range of the latest release | Release a new update with a broader semver range or one targeting the device's specific binary version |

### Source maps

When releasing a CodePush update, upload source maps to your crash reporting tool (Sentry, Datadog) so stack traces reference original code rather than minified bundles.

---

## CLI Quick Reference

```bash
# Auth
code-push login "https://codepush.pro" --accessKey $CODEPUSH_ACCESS_KEY

# App management
code-push app add MyApp-Android
code-push app add MyApp-iOS
code-push app list
code-push deployment list MyApp-Android -k

# Release
code-push release-react MyApp-Android android
code-push release-react MyApp-iOS ios
code-push release-react MyApp-Android android --deployment-name Production --rollout 10 --mandatory --targetBinaryVersion "~1.2.0"

# Promote / patch / rollback
code-push promote MyApp-Android Staging Production
code-push patch MyApp-Android Production --rollout 50
code-push rollback MyApp-Android Production

# Debug
code-push debug android
```

---

## Agent Decision Tree

### "The update is not reaching users / the app is not updating"

1. Did the customer release to the right deployment? Ask: did they use `--deployment-name Production` or are they still on Staging?
2. Is the deployment key in the native config correct? Run `code-push deployment list <app> -k` and compare against `Info.plist` / `strings.xml`.
3. Does the device's app version satisfy `targetBinaryVersion`? E.g. if binary is `1.3.0` and release targets `~1.2.0`, it will not be delivered.
4. Is there an active partial rollout? If < 100%, the user may simply not be in the rollout percentage. Check with `code-push deployment history <app> Production`.
5. Has enough time passed? The SDK checks on app launch by default — the user needs to restart the app.

---

### "The update applied but the app rolled back / is crashing"

1. Is `notifyAppReady()` being called? This is the most common cause. If the app restarts before `notifyAppReady()` runs, CodePush treats it as a failed update and reverts. Check for it in the codebase.
2. Does the update contain native code changes? CodePush cannot deliver those — the update will load but crash if it depends on a native module not in the binary.
3. Is RSA signing enabled? If the public key in the app doesn't match the private key used to sign the release, the update will be rejected at install time.
4. Check `[CodePush]` logs on device — look for `Update failed to install` or silent rollback messages.

---

### "The CLI is failing / authentication errors"

1. `Unauthorized` or `403` → token is expired or revoked. Customer can generate a new token self-service: Codemagic dashboard → **OTA Updates → Manage Access Keys → Generate key**.
2. `release-react` fails immediately → check they are running from the project root (where `package.json` is).
3. `Invalid semver` error on release → `versionName` in `build.gradle` (Android) or `CFBundleShortVersionString` in `Info.plist` (iOS) must be valid semver (`1.2.3`, not `1.0` or `1.0.0.0`).
4. `Cannot publish updates when a rollout is in progress` → patch the current rollout to 100% before publishing a new release.

---

### "Updates are applying but the wrong bundle loads"

1. Check that `AppDelegate` (iOS) or `MainApplication` (Android) was actually updated to use `CodePush.bundleURL()` / `CodePush.getJSBundleFile()`. If the original embedded bundle path is still there, CodePush updates are downloaded but never loaded.
2. Verify the correct deployment key is embedded — a Staging key in a Production build will pull Staging updates.

---

### "Analytics show high download count but low installs"

1. First check: is `notifyAppReady()` being called? High downloads + low successful installs is the classic symptom of a rollback loop.
2. Check failure count in OTA Updates dashboard — if failures are high, cross-reference with Sentry/Datadog using uploaded source maps to find the crash.
3. Check `targetBinaryVersion` — if too narrow, some devices download but fail compatibility check at install time.
