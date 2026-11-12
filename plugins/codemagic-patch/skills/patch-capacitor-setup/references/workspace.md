# Map the workspace

Work through this table before you change anything. Read files only; do not install or build yet. Each row says what to find, where to look, and which integration step it shapes ([integration.md](integration.md) sections in brackets).

| Find | Where to look | What it decides |
| --- | --- | --- |
| **App root** | The directory with `capacitor.config.ts` / `.js` / `.json`, whose `package.json` (or the workspace's) depends on `@capacitor/core`. In Nx or other monorepos, usually `apps/<name>`. | Where the plugin dependency goes [1], and where `npx cap sync`, `cmpatch-capacitor doctor` and the release command run. With several Capacitor apps, ask which one; each gets its own Patch apps. |
| **Workspace root and package manager** | The nearest ancestor with `workspaces` in `package.json`, `pnpm-workspace.yaml`, or `nx.json`; the `packageManager` field or the lockfile. Nx "integrated" repos often keep one root `package.json` for every app. | The install command, and which `package.json` holds the dependency [1]. |
| **Capacitor version** | `@capacitor/core`, `@capacitor/ios` and `@capacitor/android` versions. | Patch supports Capacitor 7 and 8 [1]. |
| **Config format and values** | `capacitor.config.ts` / `.json`: `appId`, `webDir`, `server`, `ios.path`, `android.path`, `plugins`, `loggingBehavior`; whether it reads `process.env`. | Where the Patch block goes and how Staging and Production keys are selected [2, 9]. |
| **Native project locations** | `ios.path` and `android.path` in the config (default `ios`, `android`). The iOS project `<ios.path>/App/App.xcodeproj` and target `App`; the Android module `<android.path>/app`. | Which files to check after sync [3], and native overrides [2]. `cmpatch-capacitor doctor` expects the `App` target and `App/App/capacitor.config.json`. |
| **iOS dependency manager** | Swift Package Manager: `<ios.path>/App/CapApp-SPM/`. CocoaPods: `<ios.path>/App/Podfile`. Deployment target in the Xcode project. | Both work; `cap sync` updates either [3]. iOS 15+ is required. |
| **Android toolchain** | `<android.path>/variables.gradle` (`minSdkVersion`), the Gradle JDK, NDK and CMake in the Android SDK. | minSdk 24+, JDK 21, NDK `27.0.12077973` and CMake are needed to build the plugin [6]. |
| **Web build and `webDir`** | `webDir` in the config, relative to the app root (in Nx often `../../dist/apps/<name>`). The build command: `ionic build`, `ng build`, `nx build <app>`, `vite build`, `npm run build`. Angular 17+ `outputPath` (a `browser/` subfolder unless flattened). | What `--bundle-path` is, and whether `index.html` sits at its top level [7]. |
| **Framework and first screen** | Angular (`AppComponent`, the first page), React (root component, `main.tsx`), Vue (`App.vue`, `main.ts`), Svelte, or plain JS; asynchronous initialization before the first usable screen (auth, storage migration, splash held until ready). | Where `start()` goes [4]. |
| **Things that set the server path** | `server.url` (including live-reload setups); another live-update plugin (`@capacitor/live-updates`, `cordova-plugin-ionic`, `@capgo/capacitor-updater`); `DisableDeploy` in `config.xml` or Cordova preferences; `setServerBasePath` / `persistServerBasePath` calls; a custom `CAPBridgeViewController` subclass or Android `Bridge.Builder`. | What to remove or change so Patch controls the WebView [5]. A second live-update plugin means a migration plan with the user. Appflow: [Migrating from Appflow](https://patch.codemagic.io/docs/migration/migrating-from-appflow.md). |
| **Native overrides already present** | `CodemagicPatch*` keys in `Info.plist` or `res/values*/strings.xml`, per flavor or scheme. | They win over the config [2]. |
| **Versions** | iOS `MARKETING_VERSION` in `App.xcodeproj/project.pbxproj` (`CFBundleShortVersionString` is usually `$(MARKETING_VERSION)`), Android `versionName` in `<android.path>/app/build.gradle`, plus any flavor or build-type `versionNameSuffix` (the device reports the suffixed name); versions injected by CI or computed. Flavor copies of `src/<flavor>/assets/capacitor.config.json` override the synced `src/main` copy. | The exact `--target-binary-version` per platform [8]. |
| **Service worker** | `ngsw-config.json`, `@angular/service-worker`, Workbox or `vite-plugin-pwa`, and a registration in native builds. | A cache-first service worker can keep serving old files after an update. Patch does not cover this case; disable registration in native builds, or confirm with the user [7]. |
| **Release builds and CI** | `codemagic.yaml`, `.github/workflows/`, Fastlane, Appflow config, environment files. | How Staging and Production binaries get their keys [9], and where publishing can run later. |
| **Working tree** | `git status`. | Ask the user to commit or stash before you start. |

## Workspace map and plan

Show the user what you found and what you will change, before step 2 of the workflow, for example:

```text
App root:        apps/shop  (Ionic Angular 18, Capacitor 7)
Workspace:       Nx integrated repo, one root package.json, npm
Config:          apps/shop/capacitor.config.ts, webDir ../../dist/apps/shop/browser
iOS:             apps/shop/ios/App, SPM, MARKETING_VERSION 2.4.0
Android:         apps/shop/android, minSdk 23 → raise to 24, versionName 2.4.0
First screen:    AppComponent → start() in ngAfterViewInit
Conflicts:       server.url set for live reload in capacitor.config.ts → keep it out of release builds
Plan:            npm install @codemagic/capacitor-patch (root package.json); Patch block per platform
                 read from env; start(); raise minSdk; nx build shop; npx cap sync from apps/shop
```
