# How the Patch SDK works in a React Native app

Read this before you edit anything. Every integration step exists because of one of the mechanisms below. When a project does not look like the templates, work from these rules rather than from a snippet.

## The pieces

- **Native module** `CodemagicPatchClient`. iOS ships it as a CocoaPods pod or, in React Native's SwiftPM mode, as a Swift package. Android ships it as a Gradle module with a JNI library built by CMake: zstd, HDiffPatch and SHA-256 are compiled into the app. It owns downloads, storage, bundle selection at launch and rollback.
- **JavaScript package** `@codemagic/react-native-patch`. It finds the native module as the TurboModule `NativeCodemagicPatch` when it is imported, and throws if the module is not linked. It provides `Patch.wrap`, `sync`, `notifyAppReady` and the step APIs.
- **The server's storage.** The device fetches static files from the *download base URL*. It never asks the API whether an update exists. The *API URL* only receives metrics (`POST {apiUrl}/v1/metrics/events`).

## Configuration is read from the built binary

| Key | Required | What it does |
| --- | --- | --- |
| `CodemagicPatchDeploymentKey` | yes | Selects the deployment, and is part of every manifest URL |
| `CodemagicPatchDownloadBaseUrl` | yes | Where manifests and bundles are fetched; the server's exact `PUBLIC_BASE_URL` |
| `CodemagicPatchApiUrl` | yes | Where metrics are sent; a wrong value breaks metrics, not updates |
| `CodemagicPatchPublicKey` | no | PEM public key. When set, every release must be signed, and unsigned ones are rejected |
| `CodemagicPatchMaxLaunchAttempts` | no | Launches a new update gets to confirm readiness before rollback; default `2`. On Android it must be a `<string>` resource; an `<integer>` is ignored |

- iOS reads them from the **main app bundle's** `Info.plist`. App extensions have their own bundle and do not count.
- Android reads them as **string resources looked up by name**, so any resource source works: `res/values/strings.xml`, a flavor's or build type's `res/values/`, or Gradle `resValue`.
- There is no JavaScript configuration call. A different value means a different binary. Build-time indirection works: an `Info.plist` value of `$(PATCH_IOS_DEPLOYMENT_KEY)` filled by a build setting, or a Gradle `resValue` per build type or flavor.
- A missing value reads as empty, and an empty value silently turns OTA off. Android logs one warning (`CodemagicPatchModule` tag) when the deployment key is blank.
- In `Info.plist`, write a PEM public key with real line breaks. iOS strips whitespace before decoding, but a literal `\n` stays as a backslash and an `n`, and the key then fails to parse. (`\n` is fine in `strings.xml`, in JSON, and in JavaScript strings.)

## Launch: the host asks the SDK which bundle to run

React Native resolves its JavaScript bundle before the bridge or host starts. The host code must ask Patch first and fall back to the bundle inside the app:

- iOS: `CodemagicPatch.bundleURL() ?? Bundle.main.url(forResource: "main", withExtension: "jsbundle")`
- Android: `CodemagicPatch.getJSBundleFile(applicationContext)`, where `null` means "use the embedded `index.android.bundle`"

What the selector does on each launch:

1. **New binary version.** If any stored update was installed for a different binary version than the running app, it wipes Patch's state and returns nothing, so the embedded bundle runs. The binary version is iOS `CFBundleShortVersionString` or Android `versionName`, trimmed. The build number does not count: a new store build with the same marketing version keeps its OTA updates.
2. **Crash rollback.** If a pending update has already been started `CodemagicPatchMaxLaunchAttempts` times without confirming readiness, it is marked failed and dropped. The previous update, or the embedded bundle, runs instead.
3. **Selection.** Otherwise it picks the pending update (and charges it one launch attempt), else the current one, else nothing.

The selector records the launch. Call it only on the path that really boots its result, which is the release path. Debug builds keep loading from Metro, so OTA never applies there; that is expected. Updates live in `Documents/codemagic-patch/` on iOS and `filesDir/codemagic-patch/` on Android. The bundle files inside are named `main.jsbundle` and `index.android.bundle`, the same names as the embedded bundles.

## Readiness and rollback

A newly installed update is *pending* until JavaScript running that update calls `notifyAppReady()`. That call promotes it to *current*. If the app crashes or is killed before then on `CodemagicPatchMaxLaunchAttempts` launches, the next launch rolls it back.

- `Patch.wrap(App)` renders the root immediately and calls `sync()` from a mount effect, and `sync()` calls `notifyAppReady()` first. Readiness is therefore confirmed once the root component has mounted.
- An app that is not usable until asynchronous initialization finishes (auth, remote config, migrations) should call `sync()` itself after that, instead of using the wrapper. A crash during initialization then still counts toward rollback.
- Confirming too early hides crashes from rollback. Confirming never leaves every update to roll back.

## Checking for and installing updates

`sync()` runs, in order: a concurrency guard (an overlapping call returns `"sync-in-progress"`), `notifyAppReady()`, the update check, then download and install. It never throws. It resolves to `"up-to-date"`, `"update-installed"`, `"embedded-revert-applied"`, `"sync-in-progress"` or `"error"`.

- **Manifest URLs**, built from the download base URL:
  - `{downloadBaseUrl}/{deploymentKey}/{binaryVersion}/{runningPackageHash}/manifest.json` while an update is running;
  - otherwise, or after a 404 on that URL, `{downloadBaseUrl}/{deploymentKey}/{binaryVersion}/manifest.json`.
  - A 404 means "no update". A wrong key or binary version therefore looks exactly like "up to date", with no error. Any other failure (a 403 from storage, a network error) makes `sync()` resolve `"error"`, which release builds do not log either.
- **Downloads** are a binary diff against the running update when one applies, else the full `bundle.tar.zst`. A failed diff retries the full bundle once. The package hash is verified before install, and the signature too when a public key is configured.
- **Install modes**: `ON_NEXT_RESTART` is the default for regular updates; mandatory updates use `IMMEDIATE`. `ON_NEXT_RESUME` and `ON_NEXT_SUSPEND` also exist, with `minimumBackgroundDuration` in milliseconds. `Patch.wrap(App, { checkFrequency: ON_APP_RESUME })` also checks when the app returns to the foreground. `checkFrequency` belongs to the wrapper; an app that calls `sync()` itself checks only when it calls it.
- **Restart.** iOS reloads React Native with the new bundle URL. Android needs the `Application` to implement `ReactApplication`; without it, `IMMEDIATE` installs and `restartApp()` fail. On hosts where the bundle loader cannot be swapped, the update applies on the next cold start.

## Rules that follow for every integration

1. The three required keys are in the binary that runs: the main target's `Info.plist`, and a string resource Android resolves for that build variant.
2. The bundle hook is the release path's only bundle source, and it falls back to the embedded bundle.
3. The SDK is a dependency of the app package itself, so autolinking links the native module.
4. Readiness is confirmed once the app is usable: `Patch.wrap` on the root, or `sync()` after initialization.
5. Each native change ships with a new marketing version or `versionName`. Releases target that string exactly; fingerprint expansion can also deliver a release to other versions with the same native fingerprint.
6. Only one OTA system chooses the bundle: no CodePush hook and no enabled `expo-updates` alongside Patch.
