# Map the workspace

Work through this table before you change anything. Read files only; do not install or build yet. Each row says what to find, where to look, and which integration step it shapes ([integration.md](integration.md) sections in brackets).

| Find | Where to look | What it decides |
| --- | --- | --- |
| **App root** | The directory whose `package.json` depends on `react-native` (or `expo`), next to `ios/` and `android/` or an Expo app config (`app.json`, `app.config.js`, `app.config.ts`). In a monorepo, usually under `apps/` or `packages/`. | Where the SDK dependency goes [1], where `cmpatch` runs and `codemagic-patch.config.json` lives, and where `release-react` runs. With several React Native apps, ask which one; each gets its own Patch apps. |
| **Workspace root and package manager** | The nearest ancestor whose `package.json` `workspaces`, or whose `pnpm-workspace.yaml`, includes the app root. The `packageManager` field, else the lockfile (`bun.lock`, `pnpm-lock.yaml`, `yarn.lock`, `package-lock.json`). Yarn Plug'n'Play: `.pnp.cjs`, or `nodeLinker: pnp` in `.yarnrc.yml`. `nx.json`, `turbo.json` and `lerna.json` are task runners on top of the workspace. | The install command [1]. The SDK belongs in the app's `package.json`, never the workspace root's. |
| **How JavaScript is bundled** | `metro.config.js` (`watchFolders`, `resolver.nodeModulesPaths`), hoisting settings (`.npmrc` `node-linker`, `nohoist`), the release build's bundling step. | `release-react` runs this bundler from the app root. A release bundle must already build there; Patch does not change Metro. |
| **React Native version and architecture** | `react-native` in the app's `package.json`. `newArchEnabled` in `android/gradle.properties`; `RCT_NEW_ARCH_ENABLED` in the Podfile or environment. | Whether the SDK supports the app at all (0.73+; New Architecture from 0.76) [1], and which host shapes to expect [3, 4]. |
| **Expo, and who owns the native projects** | `expo` in dependencies. Generated (Continuous Native Generation): `ios/` and `android/` are absent or git-ignored, and builds run `expo prebuild` or EAS. Maintained: they are committed and edited by hand. | Generated: the config plugin and prebuild [2]. Maintained: edit native files like bare React Native, keeping Expo's delegate and host [3, 4]. Expo SDK 52+; no Expo Go. |
| **iOS host shape** | The app target's `AppDelegate.swift` or `AppDelegate.mm`. Find the function that returns the embedded `main.jsbundle` URL (`RCTDefaultReactNativeFactoryDelegate`, `RCTAppDelegate`, `ExpoReactNativeFactoryDelegate`, or a custom host). | The bundle hook [3]. |
| **iOS targets, plists and configurations** | App targets in the `.xcodeproj` (extensions excluded); each target's `INFOPLIST_FILE`; build configurations; `.xcconfig` files; `MARKETING_VERSION`. | Which `Info.plist` gets the keys and how keys differ per configuration [2]. `--xcode-target-name` / `--build-configuration-name` at release. |
| **iOS dependency manager** | React Native's SwiftPM mode: `ios/*.xcodeproj/.spm-injected.json` exists. Otherwise CocoaPods: `ios/Podfile`, plus a `Gemfile` if pods run through Bundler. | `pod install` versus `npx react-native spm` [6]. SwiftPM mode needs React Native 0.88+, the New Architecture and SDK 0.7.0+. |
| **Android host shape** | `MainApplication.kt` / `.java`: `ReactNativeHost` or `getDefaultReactHost(...)`, Expo's `ReactNativeHostWrapper` or `ExpoReactHostFactory`; whether it implements `ReactApplication`. | The bundle hook [4]. |
| **Android variants and versions** | `android/app/build.gradle(.kts)`: `defaultConfig.versionName`, `productFlavors`, build types, `versionNameSuffix`, `resValue`, versions computed in Gradle or injected by CI; per-variant source sets under `src/`. | Where the keys go per variant [2]. `--target-binary-version` at release when the version is not a static `versionName`. |
| **Native project locations** | `ios/` and `android/` in the app root, or elsewhere through `react-native.config.js` (`project.ios.sourceDir`, `project.android.sourceDir`). | Which files you edit [2–4]. `cmpatch doctor` may need `--plist-file`, `--android-strings-file` or `--gradle-file`. |
| **Entry and root component** | `package.json` `main`, then `index.js` / `index.ts`: `AppRegistry.registerComponent(name, () => App)`. Expo: `registerRootComponent`, `expo/AppEntry` (then `App`), `expo-router/entry` (then `app/_layout`). Existing wrappers around the root; asynchronous initialization before the first usable screen. | Where readiness is confirmed: `Patch.wrap` or `sync()` after initialization [5]. |
| **Other OTA systems** | `react-native-code-push`, `@code-push-next/react-native-code-push` or another CodePush fork, `expo-updates`, and their native hooks in `AppDelegate` and `MainApplication`. | Only one system may choose the bundle. Follow the migration guide and plan the cut-over with the user. |
| **Release builds and CI** | `codemagic.yaml`, `.github/workflows/`, `eas.json` build profiles, Fastlane, `.xcconfig` files, Gradle `resValue`. | How Staging and Production binaries get their keys [9], and where OTA publishing can run later. |
| **Tests** | `jest.config.*` or `jest` in `package.json`. | The Jest setup [7]. |
| **Local server reachability** | The server URL the user gave: `http://localhost`, a LAN address, ngrok, or a real domain. | Cleartext and `adb reverse` for local tests [8]. |
| **Working tree** | `git status`. | Ask the user to commit or stash before you start, so your changes are reviewable on their own. |

## Workspace map and plan

Show the user what you found and what you will change, before step 2 of the workflow, for example:

```text
App root:        apps/mobile  (React Native 0.84, bare, New Architecture)
Workspace:       repo root, npm workspaces (hoisted node_modules); packages/ui shared
iOS:             apps/mobile/ios, CocoaPods, 1 app target (ShopApp), MARKETING_VERSION 1.0,
                 Swift ReactNativeDelegate → edit bundleURL() release branch
Android:         apps/mobile/android/app, no flavors, versionName 1.0,
                 getDefaultReactHost → add jsBundleFilePath; implements ReactApplication
Entry / root:    apps/mobile/index.js → App.tsx; no async init → Patch.wrap(App)
Other OTA:       none
Tests:           jest (react-native preset) → add the SDK's jest-setup
Release builds:  none in CI yet; Staging keys in the files for the first test build
Plan:            npm install @codemagic/react-native-patch -w @shop/mobile; Info.plist and
                 strings.xml keys; AppDelegate and MainApplication hooks; Patch.wrap; pod install
```
