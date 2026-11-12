# Integrate the SDK into a Capacitor or Ionic app

Make these edits yourself, shaped by the workspace map. Each section names the rule from [how-it-works.md](how-it-works.md) it serves. Reference page: [Native setup (Capacitor)](https://patch.codemagic.io/docs/setup/native-setup-capacitor.md).

## 1. Add the plugin to the app package

```bash
npm view @codemagic/capacitor-patch version peerDependencies
```

It supports Capacitor 7 and 8 and is a 0.x release, so check the docs for changes. Install it where the app's dependencies live:

| Layout | Command |
| --- | --- |
| single app | `npm install @codemagic/capacitor-patch` (or the repo's manager) in the app root |
| npm / Yarn / pnpm workspaces | `npm install @codemagic/capacitor-patch -w <app workspace>`, `yarn workspace <app> add …`, `pnpm --filter <app> add …` |
| Nx integrated repo (one root `package.json`) | install at the root |

Then run `npx cap sync` from the app root, the directory with `capacitor.config.*`. It links the plugin into both native projects: Podfile or `CapApp-SPM` on iOS, the Gradle module list on Android. You do not need `pod install` or AppDelegate / `MainActivity` edits.

## 2. Configure each platform

Add a block per platform you ship to `capacitor.config.ts` (or `.json`). Get the keys and URLs from [server-and-release.md](server-and-release.md). Use the **Staging** keys for the first test build.

```ts
plugins: {
  CodemagicPatch: {
    ios: {
      deploymentKey: "ios-staging-deployment-key",
      apiUrl: "https://updates.example.com",
      downloadBaseUrl: "https://storage-updates.example.com/codemagic-patch",
    },
    android: {
      deploymentKey: "android-staging-deployment-key",
      apiUrl: "https://updates.example.com",
      downloadBaseUrl: "https://storage-updates.example.com/codemagic-patch",
    },
  },
},
```

- Merge into an existing `plugins` object; do not replace other plugins' settings.
- iOS and Android must use different deployment keys. A shared key makes the platforms overwrite each other's releases.
- `publicKey` only when the user enforces code signing. In a TypeScript or JSON string, `\n` line breaks are fine. In an `Info.plist` override, use real line breaks: a literal `\n` there breaks the key on iOS.
- **Per build (Staging and Production, flavors, schemes):** either read the keys from environment variables in `capacitor.config.ts` (resolved when `cap sync` runs), or leave the config on Staging and set `CodemagicPatchDeploymentKey` as a native override in the scheme's `Info.plist` / build setting or the flavor's `strings.xml`. Overrides win key by key, so look for existing ones first.

## 3. Sync and check what the binary will carry

```bash
npx cap sync
npx codemagic-patch-check-config <android.path>/app/src/main/assets/capacitor.config.json
```

Open both synced copies and confirm each platform's block is there with the values you meant:

- iOS: `<ios.path>/App/App/capacitor.config.json`
- Android: `<android.path>/app/src/main/assets/capacitor.config.json`

`codemagic-patch-check-config` reads JSON only. It checks that the platforms do not share a key and that `maxLaunchAttempts` is valid. It does not check for missing values; you do that by reading the files.

## 4. Confirm readiness with `start()`

Call `start()` once, after the first screen has rendered, never earlier in bootstrap:

| Framework | Where |
| --- | --- |
| Angular / Ionic Angular | `ngAfterViewInit()` of `AppComponent`, or of the first page if the root renders nothing usable |
| React / Ionic React | a `useEffect(() => { void start(); }, [])` in the root component |
| Vue / Ionic Vue | `onMounted(() => { void start(); })` in `App.vue` |
| Other or plain JS | after the first render, from the app's own bootstrap |

```ts
import { Capacitor } from "@capacitor/core";
import { start } from "@codemagic/capacitor-patch";

if (Capacitor.isNativePlatform()) void start();
```

- The `isNativePlatform()` guard keeps the browser build quiet; on the web `start()` would only resolve `"error"`.
- With asynchronous initialization before the app is usable (auth, storage migration, a splash screen held until ready), call `start()` after it succeeds.
- `start()` is safe to call twice (React StrictMode in development): an overlapping call returns `"sync-in-progress"`.
- Leave the install timing at its defaults unless the user asks. For a quick test build only, `start({ installMode: InstallMode.IMMEDIATE })`; revert before shipping.

## 5. Make sure nothing else sets the server path

For release builds:

- **`server.url`.** Remove it, or set it only for live-reload development, for example from an environment variable that release builds do not set. With it, the WebView loads that URL and updates have no effect.
- **Another live-update plugin.** Remove it, its config block, and its calls, after agreeing the cut-over with the user. Devices keep the old system until they install the new binary.
- **`DisableDeploy`** in `config.xml` or Cordova preferences: remove it. It turns off Android's persisted-path boot.
- **Calls to `setServerBasePath` / `persistServerBasePath`** in app code: remove them, or ask why they exist.
- **Custom native shells.** A `CAPBridgeViewController` subclass that changes the path in `capacitorDidLoad()`, or an Android `Bridge.Builder.setServerPath()`, overrides Patch. Remove the override, or ask the user what it is for.

## 6. Native requirements

- **iOS:** deployment target 15.0 or later. With Swift Package Manager, `cap sync` updates `CapApp-SPM`. With CocoaPods, it updates the Podfile and runs the install.
- **Android:** `minSdkVersion` 24 or later in `<android.path>/variables.gradle`. JDK 21 for Gradle, plus NDK `27.0.12077973` and CMake installed through the Android SDK Manager, because the plugin compiles C code.

## 7. The web build

- `--bundle-path` is the built `webDir`, with `index.html` at its top level.
  - Angular 17+ writes `dist/<app>/browser/` by default: point `webDir` there, or flatten the output with `outputPath: { base: "www", browser: "" }`.
  - Build before every release. The CLI does not run the build, and publishing does not need `cap sync`.
- Everything the page loads must be inside `webDir`. Cordova files are injected from the binary.
- **Service workers.** A cache-first service worker (Angular `ngsw`, Workbox, `vite-plugin-pwa`) can keep serving old files after an update. Do not register it in native builds unless the user has a reason; ask.

## 8. Versions

Releases target the installed app's version exactly:

- iOS: `CFBundleShortVersionString`, usually `MARKETING_VERSION`;
- Android: `versionName`.

The CLI does not detect it, so read it from the project, or from the build the user installed. Each release of native changes needs a new version, because a release cannot be retargeted later.

## 9. Staging and Production binaries

Each binary carries one deployment key per platform. Select it at build time (section 2), and run `cap sync` before each native build. See "Build for different deployments" in [Native setup (Capacitor)](https://patch.codemagic.io/docs/setup/native-setup-capacitor.md).

## 10. A plain-HTTP server for testing

The plugin downloads with native networking, so platform rules apply.

- **Android** blocks cleartext in release builds unless the app allows it, for example with `android:usesCleartextTraffic="true"` or a network security config for the host. Allow it only for the local test build, and revert before any store build. Reach a host on your machine from an emulator or USB device with `adb reverse tcp:<port> tcp:<port>`, for each server port.
- **iOS** needs an App Transport Security exception for plain HTTP to hosts other than those already allowed.

Prefer an HTTPS server (an ngrok install) for device tests.

## 11. Check the integration

- [ ] The plugin is in the app's dependencies, and `cap sync` has run.
- [ ] Both synced configs carry complete, per-platform blocks with different keys, and no stray `CodemagicPatch*` overrides.
- [ ] `start()` runs once after the first screen, on native platforms.
- [ ] No `server.url`, second live-update plugin, `DisableDeploy` or custom server path in release builds.
- [ ] iOS 15+, Android minSdk 24+, JDK 21, NDK and CMake.
- [ ] The web build puts `index.html` at the top of `webDir`.

Then run `cmpatch-capacitor doctor` from the app root (`--project-root <app root>` from elsewhere; `--format json` to read it).
