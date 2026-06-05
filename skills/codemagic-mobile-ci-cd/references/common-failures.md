# Common Failures

Apply fixes inline and retry. Use [build-logs-api.md](build-logs-api.md) to fetch the first failed step log.

## General

| Symptom | Fix |
|---------|-----|
| Config not detected | `codemagic.yaml` at repo root, exact spelling; click **Check for configuration file** |
| Build never starts on push | Add `triggering:` **and** create webhook in app settings |
| Repo not connected | Connect Git provider; confirm Codemagic access |

## Flutter

| Symptom | Fix |
|---------|-----|
| `pubspec.yaml` missing | Not Flutter — re-detect stack |
| `flutter pub get` fails | Fix `pubspec.yaml` / lockfile |
| iOS codesign on debug | Use `--no-codesign` for phase 1 |

## React Native

| Symptom | Fix |
|---------|-----|
| `android/` missing (Expo) | `npx expo prebuild --platform android --non-interactive` |
| SDK not found | Generate `local.properties` with `$ANDROID_SDK_ROOT` |
| Gradle fails after separate `cd` | Merge: `cd android && ./gradlew ...` in one script |
| CocoaPods (iOS) | `cd ios && pod install`; for phase 1 validation, try Android first |

## Android (all stacks)

| Symptom | Fix |
|---------|-----|
| Gradle dependency errors | Fix versions; re-run `npm install` / `flutter pub get` |
| Release signing fails | Walk user through [Signing Android apps](https://docs.codemagic.io/yaml-code-signing/signing-android/); verify reference name and Gradle `CI=true` block |
| Version code rejected by Play | Increment with `google-play get-latest-build-number` |

## iOS (all stacks)

| Symptom | Fix |
|---------|-----|
| Provisioning profile mismatch | Check `distribution_type` + `bundle_identifier` |
| `build-ipa` fails | Ensure `xcode-project use-profiles` runs before build |
| Missing certs | Walk user through [Signing iOS apps](https://docs.codemagic.io/yaml-code-signing/signing-ios/); ask Android-first vs continue iOS |
| Flutter IPA export | Add `--export-options-plist=/Users/builder/export_options.plist` |

## Publishing

| Symptom | Fix |
|---------|-----|
| Play upload fails | See https://docs.codemagic.io/troubleshooting/common-google-play-errors/ |
| TestFlight stuck | Check build log post-processing; App Store Connect email |
| First app on store | Manual first upload / metadata often required |

## Deeper guides

- iOS — https://docs.codemagic.io/troubleshooting/common-ios-issues/
- Android — https://docs.codemagic.io/troubleshooting/common-android-issues/
- SSH debug — https://docs.codemagic.io/troubleshooting/accessing-builder-machine-via-ssh/
