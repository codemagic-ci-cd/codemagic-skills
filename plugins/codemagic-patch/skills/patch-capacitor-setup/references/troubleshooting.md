# Troubleshooting Capacitor and Ionic

Read with: [Troubleshooting and debugging](https://patch.codemagic.io/docs/troubleshooting.md) (headings named below are on that page) and [Verify a test release](https://patch.codemagic.io/docs/using-patch/verify-test-release.md).

## Diagnose first

From the app root:

| Check | Command |
| --- | --- |
| Setup, each configured platform | `cmpatch-capacitor doctor` |
| One platform and deployment | `cmpatch-capacitor doctor --platform ios --app <app> --deployment Staging --verbose` |
| After publishing | `cmpatch-capacitor doctor --verify-delivery` (add `--target-binary-version` to check a specific build) |
| Release state | `cmpatch-capacitor release inspect --app <app> --deployment Staging --label <label> --wait` |

Doctor reads the project, the synced configs, the server and the published files. Among other things, it reports:

- missing or mismatched values, and native overrides;
- a shared key;
- a stale or missing synced config, and a missing plugin registration;
- web sources that never call `start()`;
- an unbuilt `webDir`.

It cannot see the installed build, and it exits `1` when a check fails. Values that only the build decides (a Gradle `resValue`, a flavor, an `Info.plist` `$(SETTING)`) are reported as unresolved. Pass `--platform`, `--app` and `--deployment` to check them against a deployment.

When `sync()` only says `"error"`, call the step APIs (`checkForUpdate`, `downloadUpdate`) once from a debug screen or the console. They throw `CodemagicPatchError` with a `code`.

## Symptoms

| Symptom | Likely cause | Check, then read |
| --- | --- | --- |
| The release is published, nothing ever changes on the device | The installed binary was built before the plugin was added, or before `cap sync` carried the config | Sync and rebuild. "Native binary without the SDK" |
| No update offered, no error | The release's target version differs from the installed `CFBundleShortVersionString` / `versionName` | Compare them. "Wrong binary version targeting" |
| No update offered, no error | A wrong deployment key (another platform, deployment, or a leftover native override), or a `downloadBaseUrl` that is not the server's `PUBLIC_BASE_URL` | Read both synced configs and the native overrides. "Incorrect deployment key", "Release published but app finds no update" |
| `CONFIGURATION_INVALID` | A platform block is incomplete, sits directly under `CodemagicPatch`, or `cap sync` did not run | Fix the config and sync |
| Updates never apply | `server.url` is set, live reload is on, or another plugin or custom shell sets the server path | Remove it from release builds. [how-it-works.md](how-it-works.md) |
| The update downloads but the old UI stays | The default install mode applies on the next cold start | Close and reopen the app |
| Blank screen after an update | `index.html` was not at the top level of `--bundle-path` | Publish the `webDir` itself. "App shows a blank screen after an update" |
| The update installs, then rolls back | `start()` / `sync()` never ran in the new bundle, or the app crashed before it | Check where `start()` is called |
| Old files keep showing after an update | A service worker serves its cache | Turn it off in native builds |
| One launch shows the embedded bundle after installing a build with only a new `versionCode` (Android) | Capacitor ignores its persisted path on the first launch of a new binary | Relaunch. Bump `versionName` for native releases |
| Signed releases are rejected on iOS only | The `Info.plist` public-key override contains a literal `\n` | Use real line breaks, or set `publicKey` in the config |
| Android build fails in the plugin's native code | NDK `27.0.12077973`, CMake or JDK 21 is missing, or `minSdkVersion` is below 24 | Install them through the SDK Manager, or raise minSdk |
| Network errors on a phone | The device cannot reach a `localhost` or plain-HTTP server | Use HTTPS (ngrok), or the cleartext and `adb reverse` steps in [integration.md](integration.md) |
| The release stays processing | The server's release worker failed or is slow | "Release stuck processing" |

## Device logs

- Android: `adb logcat -s CodemagicPatch`. Release builds log only with `loggingBehavior: "production"` at the top level of the Capacitor config.
- iOS: the Xcode console. The same `loggingBehavior` rule applies.
