# Troubleshooting React Native and Expo

Read with: [Troubleshooting and debugging](https://patch.codemagic.io/docs/troubleshooting.md) (the headings named below are on that page unless another page is linked) and [Verify a test release](https://patch.codemagic.io/docs/using-patch/verify-test-release.md).

## Diagnose first

Run the read-only checks from the app root before you change anything:

| Check | Command |
| --- | --- |
| Setup, both platforms | `cmpatch doctor` |
| One platform and deployment | `cmpatch doctor --platform ios --app <app> --deployment Staging --verbose` |
| After publishing | `cmpatch doctor --verify-delivery` |
| Release state | `cmpatch release inspect --app <app> --deployment Staging --label <label> --wait` |
| Resolved server, team, project root and the server's SDK configuration | `cmpatch context --remote` |
| Server health | `cmpatch server status` |

Use `--format json` to read the results. Doctor exits `1` on a confirmed failure, and `0` can still leave checks unresolved, so read the report rather than only the exit code. Doctor reads the project, the server and the published releases; it cannot see the installed build. When native target selection is ambiguous, pass `--platform ios --plist-file <path>`, or `--platform android` with `--android-strings-file` / `--gradle-file`.

## Symptoms

| Symptom | Likely cause | Check, then read |
| --- | --- | --- |
| The release is published, nothing ever changes on the device | The installed binary was built before the SDK was added, or without the right key | Rebuild and reinstall. "Native binary without the SDK" |
| No update is offered, and no error either | The binary's version differs from the release's target binary version | Compare the installed `CFBundleShortVersionString` / `versionName` with the release in `cmpatch release list --format json`. "Wrong binary version targeting" |
| No update is offered, and no error either | The deployment key in the binary is for another deployment or platform, or has a typo. The server answers 404, which the SDK treats as "no update" | Compare the packaged key with `cmpatch deployment list`. "Incorrect deployment key" |
| No update is offered, and no error either | `CodemagicPatchDownloadBaseUrl` is not the server's exact `PUBLIC_BASE_URL` | Compare with `cmpatch context --remote`. "Release published but app finds no update" |
| The update downloads but the old UI stays | The default install mode applies a regular update on the next cold start | Force-quit and reopen the app. "Update downloaded but not visible yet" |
| Nothing happens in a debug build | Debug builds load JavaScript from Metro | Test a release build |
| Network errors on a phone | The binary points at `localhost`, or Android blocks plain HTTP | Use the HTTPS URLs of an ngrok or real install. For the local stack on Android: `adb reverse` and cleartext, in [Local quickstart](https://patch.codemagic.io/docs/local-quickstart.md) |
| The update installs, then rolls back | The readiness call never ran on the new bundle (`Patch.wrap`, `sync()` or `notifyAppReady()`), or the app crashed before it | Find where readiness is confirmed. "Missing `notifyAppReady()` with manual flows" |
| `release-react` cannot find the project, the entry file or the version | It ran outside the app root, the monorepo's Metro setup cannot bundle from there, or the version is set by flavors or CI | Run from the app root; pass `--bundler`, `--entry-file`, `--target-binary-version`, `--xcode-target-name`. "`cmpatch release-react` run outside the project root" |
| The app crashes at start with `'NativeCodemagicPatch' could not be found` | The native module is not linked: the SDK is not in the app package's dependencies, pods or SwiftPM packages were not installed, or the binary predates the SDK | Check the app's `package.json`, run `pod install` / `npx react-native spm`, rebuild |
| Signed releases are all rejected on iOS, but Android accepts them | The `Info.plist` public key contains a literal `\n` | Put the PEM in `Info.plist` with real line breaks |
| `IMMEDIATE` install or `restartApp()` fails on Android | The `Application` does not implement `ReactApplication` | Implement it, or let the update apply on the next cold start |
| A new store build still runs an older OTA update | Only the build number changed; Patch keys stored updates to the marketing version / `versionName` | Bump the marketing version with each native release |
| `doctor` warns on the download connectivity check | The download base is a loopback host (`localhost`), or the server answered 5xx / 429 / a login redirect. A 403 alone passes: it proves reachability only | Expected for a local server. Run `doctor --verify-delivery` after publishing to check the real files |
| `release-react` says "Could not find hermesc under project root", or the iOS dry run shows no Hermes step | `react-native` / `hermes-compiler` are hoisted out of the app's `node_modules` in a monorepo | See "Hermes" in [server-and-release.md](server-and-release.md); tell the user before publishing with `--hermes false` |
| The release stays `uploaded` or `processing` | The server's release worker failed or is slow | `release inspect --wait`, then the server logs. "Release stuck processing" |
| `npx react-native spm add` stops on mixed Swift and Objective-C sources | The SDK is older than 0.7.0 | Upgrade the SDK; do not set `platforms: { ios: null }`. "`npx react-native spm add` reports mixed Swift and Objective-C sources" |
| Jest fails with `'NativeCodemagicPatch' could not be found`, or on `export` in the SDK's source | The SDK's Jest setup file, or `@codemagic` in `transformIgnorePatterns`, is missing | [Testing your app](https://patch.codemagic.io/docs/setup/testing.md) |
| A release is refused for a fingerprint disagreement | The native state on disk differs from the first release for that deployment and binary version: a changed native file, a `Podfile.lock` missing on one machine or CI checkout, or (Expo) different app-config environment variables | Find what changed. Publish from the commit the binary was built from. [Fingerprinting](https://patch.codemagic.io/docs/using-patch/fingerprinting.md) |
| Android release build has no `CodemagicPatch*` strings | Resource shrinking on SDKs older than 0.5.2 | Upgrade the SDK. "Release published but app finds no update" |
| CI cannot publish | The runner cannot reach a `localhost` server, or has no token | Use a public server URL and `CODEMAGIC_PATCH_TOKEN`. [CI integration](https://patch.codemagic.io/docs/using-patch/ci-integration.md) |
| Sign-in, server boot or certificate problems | Server configuration | "Server issues" |

## Device logs

- Android: `adb logcat`. The SDK warns once under the `CodemagicPatchModule` tag when the deployment key is missing or blank.
- iOS: the Xcode console for the simulator or device.

The lifecycle to trace is: check for update, download, install, restart.
