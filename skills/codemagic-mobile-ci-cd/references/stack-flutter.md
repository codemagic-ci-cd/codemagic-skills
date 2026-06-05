# Flutter Stack (Phase 1+)

Published quickstart: https://docs.codemagic.io/yaml-quick-start/building-a-flutter-app/

## Phase 1 — Unsigned build

**Android (default validation path):**

```yaml
workflows:
  flutter-android-debug:
    name: Flutter Android debug
    instance_type: mac_mini_m2
    max_build_duration: 30
    environment:
      flutter: stable
    scripts:
      - name: Get dependencies
        script: flutter pub get
      - name: Build debug APK
        script: flutter build apk --debug
    artifacts:
      - build/**/outputs/**/*.apk
```

**iOS (when user targets iOS):**

```yaml
  flutter-ios-unsigned:
    name: Flutter iOS unsigned
    instance_type: mac_mini_m2
    max_build_duration: 60
    environment:
      flutter: stable
      xcode: latest
      cocoapods: default
    scripts:
      - name: Get dependencies
        script: flutter pub get
      - name: Install CocoaPods
        script: find . -name "Podfile" -execdir pod install \;
      - name: Build iOS without signing
        script: flutter build ios --debug --no-codesign
    artifacts:
      - build/ios/iphoneos/**/*.app
      - /tmp/xcodebuild_logs/*.log
```

## Phase 3+ — Signed release

Use [ios-release-lane.md](ios-release-lane.md) or [android-release-lane.md](android-release-lane.md) with Flutter-specific build commands:

- Android: `flutter build appbundle` (after signing configured)
- iOS: `flutter build ipa` after `xcode-project use-profiles`; may need `--export-options-plist=/Users/builder/export_options.plist`

## Stack-specific failures

- `pubspec.yaml` missing → not a Flutter project
- `flutter pub get` fails → fix dependencies in `pubspec.yaml`
- CocoaPods errors on iOS → run `pod install` in script; see common-failures

See [common-failures.md](common-failures.md).
