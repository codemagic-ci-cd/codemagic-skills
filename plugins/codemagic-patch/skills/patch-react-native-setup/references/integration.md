# Integrate the SDK into a React Native or Expo app

Make these edits yourself, shaped by the workspace map. Each section names the rule from [how-it-works.md](how-it-works.md) it serves. For a version-specific snippet beyond the ones here, read [Native setup](https://patch.codemagic.io/docs/setup/native-setup.md).

## 1. Add the SDK to the app package

Check what is published and what it supports before you install anything:

```bash
npm view @codemagic/react-native-patch version peerDependencies
```

The SDK supports React Native 0.73 and later: Old Architecture only on 0.73–0.75, both architectures from 0.76. Expo needs SDK 52 or later and a development build; Expo Go cannot load it. React Native's SwiftPM mode needs an SDK version that ships `Package.swift` (0.7.0 or later). If the published version is older, the app cannot use SwiftPM mode with Patch yet; tell the user.

Install it into the **app's** `package.json` with the repository's package manager:

| Manager | Command |
| --- | --- |
| npm workspaces | `npm install @codemagic/react-native-patch -w <app workspace>` from the workspace root |
| Yarn | `yarn workspace <app package name> add @codemagic/react-native-patch`, or `yarn add …` in the app root |
| pnpm | `pnpm --filter <app package name> add @codemagic/react-native-patch` |
| Bun | `bun add @codemagic/react-native-patch` in the app root |
| single-app repo | the manager's `add` / `install` in the app root |

Autolinking reads the app package's dependencies, so a copy hoisted from somewhere else is not enough. In a monorepo, confirm afterwards that `node_modules/@codemagic/react-native-patch` resolves from the app root.

## 2. Put the configuration into each binary

Get the deployment keys, API URL and download base URL from [server-and-release.md](server-and-release.md). Use the **Staging** keys for the first test build.

### iOS: the app target's `Info.plist`

Find the plist of the target that builds the app (its `INFOPLIST_FILE` build setting), not the plist of an extension:

```xml
<key>CodemagicPatchDeploymentKey</key>
<string>ios-staging-deployment-key</string>
<key>CodemagicPatchDownloadBaseUrl</key>
<string>https://storage-updates.example.com/codemagic-patch</string>
<key>CodemagicPatchApiUrl</key>
<string>https://updates.example.com</string>
```

- For different keys per build configuration (Staging and Production builds from one project), set the value to `$(PATCH_IOS_DEPLOYMENT_KEY)` and define that build setting per configuration, or in the configuration's `.xcconfig`.
- `CodemagicPatchPublicKey`, only when the user enforces code signing: put the PEM with real line breaks inside `<string>`, not `\n`.

### Android: a string resource for the build variant

The usual place is `android/app/src/main/res/values/strings.xml`:

```xml
<string name="CodemagicPatchDeploymentKey" translatable="false">android-staging-deployment-key</string>
<string name="CodemagicPatchDownloadBaseUrl" translatable="false">https://storage-updates.example.com/codemagic-patch</string>
<string name="CodemagicPatchApiUrl" translatable="false">https://updates.example.com</string>
```

- **Product flavors or per-build-type keys:** put the deployment key in that variant's source set (`src/<flavor>/res/values/strings.xml`), or generate it with `resValue "string", "CodemagicPatchDeploymentKey", …` in the flavor or build type. Never define the same name in both `main/` and `resValue`; the build fails on the duplicate.
- Keep `translatable="false"`, so translation tooling leaves the values alone.

### Expo with generated native projects: the config plugin

Do not edit `ios/` or `android/`; prebuild regenerates them. Add the plugin to the app config:

```json
["@codemagic/react-native-patch", {
  "ios":     { "deploymentKey": "ios-staging-deployment-key",     "apiUrl": "https://updates.example.com", "downloadBaseUrl": "https://storage-updates.example.com/codemagic-patch" },
  "android": { "deploymentKey": "android-staging-deployment-key", "apiUrl": "https://updates.example.com", "downloadBaseUrl": "https://storage-updates.example.com/codemagic-patch" }
}]
```

- Each platform block needs all three values. `publicKey` and `maxLaunchAttempts` are optional. Missing values fail `expo prebuild`.
- In a dynamic `app.config.js` / `app.config.ts`, read the keys from environment variables (for example per EAS build profile), and set the same variables when you run prebuild locally.
- The plugin also writes the bundle hooks (sections 3 and 4) during prebuild. It finds them by the template's exact expressions, and prebuild fails if a customized `AppDelegate` or `MainApplication` no longer contains them. In that case make the edit by hand in a config plugin of the project's own, or move the app to maintained native projects.
- Disable or remove `expo-updates` first; two systems cannot both pick the bundle.

## 3. Hook bundle selection on iOS

**Rule:** in the code that returns the bundle URL, the release branch returns `CodemagicPatch.bundleURL()` and falls back to the embedded `main.jsbundle`. The `DEBUG` branch stays on Metro.

Find the expression that returns the embedded bundle (`Bundle.main.url(forResource: "main", withExtension: "jsbundle")` or `[[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"]`) and wrap it. Where it lives depends on the template:

| Host shape | Where | Edit |
| --- | --- | --- |
| React Native 0.79+ (Swift `ReactNativeDelegate: RCTDefaultReactNativeFactoryDelegate`) | `bundleURL()` in `AppDelegate.swift` | Swift, below |
| React Native 0.77–0.78 (Swift `AppDelegate: RCTAppDelegate`) | `override func bundleURL()` | Swift, below |
| React Native 0.73–0.76 (Objective-C++ `AppDelegate.mm`) | `- (NSURL *)bundleURL` or `sourceURLForBridge:` | Objective-C++, below |
| Expo SDK 53+ (`ExpoReactNativeFactoryDelegate`) | `bundleURL()`; keep the `.expo/.virtual-metro-entry` debug root and Expo's superclass | Swift, below |
| Expo SDK 52 (`AppDelegate.mm`) | the release branch of `bundleURL` | Objective-C++, below |
| Brownfield or custom host (`RCTRootView`, `RCTReactNativeFactory` built by hand) | wherever the host passes a bundle URL | same rule |

Swift (add the import at the top of the file):

```swift
import CodemagicPatchClient

override func bundleURL() -> URL? {
#if DEBUG
  RCTBundleURLProvider.sharedSettings().jsBundleURL(forBundleRoot: "index")
#else
  CodemagicPatch.bundleURL() ?? Bundle.main.url(forResource: "main", withExtension: "jsbundle")
#endif
}
```

On the React Native 0.79+ delegate, `sourceURL(for:)` should return `self.bundleURL()`, as the template does.

Objective-C++: the module's generated Swift header is not on the app's search paths, so forward-declare the class above `@implementation`:

```objc
@interface CodemagicPatch : NSObject
+ (NSURL *_Nullable)bundleURL;
@end

// in the release branch:
return [CodemagicPatch bundleURL] ?: [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
```

## 4. Hook bundle selection on Android

**Rule:** the host's JavaScript bundle file is `CodemagicPatch.getJSBundleFile(applicationContext)`. It returns `null` when the embedded bundle should run, and React Native then loads `index.android.bundle` from assets.

| Host shape (in `MainApplication`) | Edit |
| --- | --- |
| `ReactNativeHost` / `DefaultReactNativeHost` (React Native ≤0.81, and every Old Architecture app) | inside the host object: `override fun getJSBundleFile(): String? = CodemagicPatch.getJSBundleFile(applicationContext)` |
| `getDefaultReactHost(...)` (React Native 0.82+) | add the argument `jsBundleFilePath = CodemagicPatch.getJSBundleFile(applicationContext)` |
| Expo SDK 52–54 (`ReactNativeHostWrapper`) | the `getJSBundleFile()` override, inside the wrapped `DefaultReactNativeHost` |
| Expo SDK 55+ (`ExpoReactHostFactory.getDefaultReactHost(...)`) | the `jsBundleFilePath = …` argument; keep the other arguments |
| Java `MainApplication` | `return CodemagicPatch.getJSBundleFile(getApplicationContext());` in `getJSBundleFile()` |

Import `io.codemagic.patch.CodemagicPatch`. The `Application` must implement `ReactApplication`, as the templates do; otherwise `IMMEDIATE` installs and `restartApp()` fail.

## 5. Confirm readiness from JavaScript

Find the root component through the entry: `index.js` → `AppRegistry.registerComponent(name, () => App)`, Expo `registerRootComponent(App)`, `expo/AppEntry` → `App`, or `expo-router/entry` → the root `app/_layout`.

- **Usual case:** export the root through the wrapper, `export default Patch.wrap(App)`, with `import * as Patch from "@codemagic/react-native-patch"`.
- **Already wrapped by another library** (Sentry, a store provider HOC): compose them, for example `export default Sentry.wrap(Patch.wrap(App))`. Remove a CodePush wrapper when migrating.
- **Asynchronous initialization** (auth bootstrap, remote config, database migration, a splash screen held until ready): do not use the wrapper. Call `void Patch.sync()` once initialization has succeeded, so a crash during initialization still counts toward rollback.
- **Leave the install timing at its defaults** unless the user asks. For test builds only, `sync({ installMode: Patch.InstallMode.IMMEDIATE })` shortens the loop; revert it before shipping.

## 6. Native dependencies

- **CocoaPods:** install pods the way the project does. Look at its README and scripts. With a `Gemfile` that pins CocoaPods, that is `bundle install` and then `bundle exec pod install`; otherwise `cd ios && pod install`. Commit `Podfile.lock` and the tracked files `pod install` changed (`project.pbxproj`, the `.xcworkspace`) before the build. Generated files it writes (`ios/Pods`, `ios/build`, `ios/.xcode.env.local`) are outside the fingerprint.
- **SwiftPM mode:** never run `pod install`. Run `npx react-native spm` after installing the SDK and on every fresh checkout. Never opt the SDK out with `platforms: { ios: null }`; the app would build without Patch.
- **Expo generated projects:** `npx expo prebuild` (ask before `--clean`), then `pod install` unless the app uses Expo's Swift Package Manager experiment.
- **Android:** the SDK compiles C code, so the Android SDK needs the NDK and CMake. Its Gradle module finds `react-native` the way autolinking does, by running `node --print "require.resolve('react-native/package.json')"` in the app's `android/` directory, so hoisted monorepo layouts work as long as `node` is on Gradle's `PATH`. Where that cannot resolve, set `ext.reactNativeDir` in `android/build.gradle`.

## 7. Tests

Jest cannot load the native module. Keep the project's preset and add:

```js
setupFiles: ['@codemagic/react-native-patch/jest-setup'],
transformIgnorePatterns: ['node_modules/(?!((jest-)?react-native|@react-native(-community)?|@codemagic)/)'],
```

Merge `@codemagic` into the existing `transformIgnorePatterns` rather than replacing it. Details: [Testing your app](https://patch.codemagic.io/docs/setup/testing.md).

## 8. A plain-HTTP server for testing

A local evaluation server is `http://localhost`. Prefer an HTTPS server (an ngrok install) for anything beyond a simulator.

- **Android** release builds block cleartext. React Native templates set `android:usesCleartextTraffic="${usesCleartextTraffic}"` in `AndroidManifest.xml`, and the React Native Gradle plugin sets that placeholder to `false` for release after the app's own Gradle config has run. For a local test build, set the attribute to `"true"` in the manifest, tell the user, and list it as something to revert before any store build. The manifest is part of the Android fingerprint, so a test build with this edit and a store build without it fingerprint differently. Give the store build a new `versionName`, or test against an HTTPS server instead. Reach the host from an emulator or USB device with `adb reverse tcp:<api port> tcp:<api port>`, and the same for the storage port.
- **iOS:** the React Native template allows local networking (`NSAllowsLocalNetworking`), so a simulator reaches `localhost`. Other plain-HTTP hosts need an App Transport Security exception; do not ship one.

## 9. Staging and Production binaries

Each binary carries one deployment key. Produce Staging and Production builds by switching the key at build time (section 2), never by editing the file between builds by hand. See "Build for different deployments" in [Native setup](https://patch.codemagic.io/docs/setup/native-setup.md).

## 10. Check the integration

Read the diff back against [how-it-works.md](how-it-works.md):

- [ ] The SDK is in the app package's dependencies, and native dependencies are installed.
- [ ] Each binary variant resolves all three keys, with the iOS and Android keys from different apps.
- [ ] iOS release path: `CodemagicPatch.bundleURL()` first, embedded `main.jsbundle` second; `DEBUG` still on Metro.
- [ ] Android: `CodemagicPatch.getJSBundleFile(applicationContext)` is the bundle file; `Application` implements `ReactApplication`.
- [ ] Readiness: `Patch.wrap` on the root, or `sync()` after initialization.
- [ ] No other OTA system is active.

Then run `cmpatch doctor` from the app root (`--format json` to read it). When `ios/` and `android/` sit in the app root and there is one app target, `cmpatch wire --dry-run --diff` is a second opinion: it changes nothing and shows the edits it would make. If it proposes an edit you did not make, find out why before you continue.
