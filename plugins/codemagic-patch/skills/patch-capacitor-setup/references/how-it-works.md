# How the Patch SDK works in a Capacitor app

Read this before you edit anything. Every integration step exists because of one of the mechanisms below. When a project does not look like the templates, work from these rules rather than from a snippet.

## The pieces

- **The plugin** `@codemagic/capacitor-patch`. It is a Capacitor plugin with native iOS and Android code; `npx cap sync` links it.
  - It downloads updates, verifies and stores them, chooses which web bundle the WebView loads, and rolls back bad updates.
  - Android compiles C code (zstd, ustar, HDiffPatch, SHA-256) into a JNI library with the NDK and CMake.
- **JavaScript API**: `start()`, `sync()`, `notifyAppReady()` and step APIs. On the web (not a native platform) every call fails, and `start()` / `sync()` resolve `"error"`.
- **The server's storage.** The device fetches static files from the *download base URL*. It never asks the API whether an update exists. The *API URL* only receives metrics (`POST {apiUrl}/v1/metrics/events`).

## Configuration comes from the synced Capacitor config

The settings live in the Capacitor config, per platform:

```ts
plugins: {
  CodemagicPatch: {
    ios:     { deploymentKey: "…", apiUrl: "…", downloadBaseUrl: "…" },
    android: { deploymentKey: "…", apiUrl: "…", downloadBaseUrl: "…" },
  },
}
```

- **What each platform reads.** Only `plugins.CodemagicPatch.ios.*` and `plugins.CodemagicPatch.android.*`. Keys placed directly under `CodemagicPatch` are ignored.
  - Each block you add needs `deploymentKey`, `apiUrl` and `downloadBaseUrl`, and at least one block must exist.
  - Optional: `publicKey` (a PEM; when it is set, unsigned releases are rejected) and `maxLaunchAttempts` (a positive integer; default `2`).
  - Values are trimmed, and a blank value counts as absent.
- **The native side reads the copy `npx cap sync` writes into the native project**, not your `capacitor.config.ts`:
  - iOS: `<ios.path>/App/App/capacitor.config.json`
  - Android: `<android.path>/app/src/main/assets/capacitor.config.json`
  - A `.ts` config is evaluated at sync time, so `process.env.X` is resolved then. A variable that is not set simply leaves the key out.
  - A config change needs `cap sync` and a new binary.
- **Native overrides win, key by key.** These keys in `Info.plist` or Android string resources override the config:
  - `CodemagicPatchDeploymentKey`, `CodemagicPatchApiUrl`, `CodemagicPatchDownloadBaseUrl`, `CodemagicPatchPublicKey`, `CodemagicPatchMaxLaunchAttempts`.
  - They let a scheme, build configuration or flavor carry its own key. They are also the leftovers to look for when the config seems ignored.
- **Invalid or missing config is logged** (iOS via Capacitor's logger, Android under the `CodemagicPatch` tag; release builds show logs only with Capacitor's top-level `loggingBehavior: "production"`).
  - Checking and downloading then fail with `CONFIGURATION_INVALID`.
  - Updates already installed keep booting.

## Launch: the plugin points the WebView at an update

Capacitor normally serves the web assets bundled in the app (`public/`). When a plugin loads (`load()`), Patch decides what the WebView will serve:

- **iOS**: `bridge.setServerBasePath(<update>/contents)`, before the WebView loads. The plugin records the original app location as "embedded".
- **Android**: the plugin writes the path to Capacitor's persisted `serverBasePath` setting (SharedPreferences `CapWebViewSettings`), which Capacitor applies when it loads the WebView. It removes that setting when the embedded bundle should run.

How it chooses, on each launch:

1. **New binary version.** If any stored update belongs to a different binary version than the running app, Patch wipes its state and the embedded bundle runs. The binary version is iOS `CFBundleShortVersionString` or Android `versionName`. The build number does not count.
2. **Crash rollback.** If a pending update has been started `maxLaunchAttempts` times without confirming readiness, it is marked failed and dropped.
3. **Selection.** Otherwise it picks the pending update (and charges one launch attempt), else the current one, else the embedded bundle.

Updates live in `Documents/codemagic-patch/` on iOS and `filesDir/codemagic-patch/` on Android.

The same mechanism means **anything else that sets Capacitor's server path competes with Patch**:

- `server.url` in the config (live reload, or a hosted URL);
- another live-update plugin (Appflow Live Updates, Capgo, …) and its persisted path;
- the Cordova preference `DisableDeploy`;
- a custom `Bridge.Builder.setServerPath()`;
- a `CAPBridgeViewController` subclass that changes the path in `capacitorDidLoad()`.

Capacitor's Android bridge also ignores the persisted path on the first launch after any `versionCode` or `versionName` change. So a build that changes only `versionCode` can show the embedded bundle on that one launch before the update runs again. This comes from reading Capacitor's source; it is not covered by Patch's tests.

## Readiness and rollback

A new update is *pending* until JavaScript running that update confirms readiness with `notifyAppReady()`, which `start()` and `sync()` call first. If the app crashes or is killed before then on `maxLaunchAttempts` launches, the next launch rolls it back.

- `start(options)` runs `sync()` immediately. It never rejects. With `checkFrequency: CheckFrequency.ON_APP_RESUME`, it syncs again each time the app returns to the foreground. Foreground changes are detected natively; no extra plugin is needed.
- Readiness is recorded at the first `start`, `sync` or `notifyAppReady` call. Make that call after the first screen has rendered, or after asynchronous initialization has finished, so a crash before then still counts toward rollback.

## Checking for and installing updates

`sync()` runs:

1. a concurrency guard (an overlapping call returns `"sync-in-progress"`);
2. `notifyAppReady()`;
3. the update check;
4. download and install.

It never throws. It resolves to `"up-to-date"`, `"update-installed"`, `"embedded-revert-applied"`, `"sync-in-progress"` or `"error"`. The error detail is only available through the step APIs, which throw `CodemagicPatchError` with a `code`.

- **Manifest URLs**, from the download base URL:
  - `{downloadBaseUrl}/{deploymentKey}/{binaryVersion}/{runningPackageHash}/manifest.json` while an update runs;
  - `{downloadBaseUrl}/{deploymentKey}/{binaryVersion}/manifest.json` otherwise, or after a 404 on that URL.
  - A 404 means "no update". A wrong key, binary version or base URL therefore looks like "up to date".
- **Downloads.** A binary diff against the running update when one applies, else the full `bundle.tar.zst`; a failed diff retries the full bundle once. The package hash is verified, and the signature too when a public key is configured.
- **Install modes.** `ON_NEXT_RESTART` is the default; mandatory updates use `IMMEDIATE`. `ON_NEXT_RESUME` also exists, with `minimumBackgroundDuration`.
- **Activation.** Activating reloads the WebView. iOS keeps the current URL. Android goes back to the start URL.

## What a bundle is

A release is the built `webDir`, with `index.html` at its top level. It becomes the web root of the same origin the embedded bundle used, so:

- relative paths, root-absolute paths, `<base href="/">` and path or hash routing behave as they do in the embedded bundle;
- browser storage stays with the origin;
- `cordova.js`, `cordova_plugins.js` and `plugins/` are injected from the binary;
- everything else the page loads must be inside `webDir`;
- a bundle uploaded one directory level too high installs, and the app shows a blank screen.

## Rules that follow for every integration

1. The plugin is a dependency of the app package, and `npx cap sync` has run, so both native projects link it and carry the current config.
2. Each binary resolves the three required values per platform, from the synced config or a native override. iOS and Android use different deployment keys.
3. Nothing else sets Capacitor's server path in release builds: no `server.url`, no second live-update plugin, no `DisableDeploy`, and no custom bridge path.
4. Readiness is confirmed after the first screen renders: `start()` in the right lifecycle hook.
5. Each native change ships with a new marketing version or `versionName`, because releases target that string exactly.
6. The published bundle is the built `webDir`, built for the same native shell.
