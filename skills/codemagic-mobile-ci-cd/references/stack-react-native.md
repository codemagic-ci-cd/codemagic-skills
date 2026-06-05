# React Native Stack (Phase 1+)

Published quickstart: https://docs.codemagic.io/yaml-quick-start/building-a-react-native-app/

## Variant selection

| Variant | When | Path |
|---------|------|------|
| **RN CLI** (default) | `ios/` and/or `android/` in repo | Standard Gradle / Xcode scripts below |
| **Expo with prebuild** | `app.json` / `app.config.*`, no committed `android/` | Add `npx expo prebuild` before native build |
| **Expo without prebuild** (escape hatch) | User cannot commit native folders | See quickstart "Using Expo without prebuild" — `support-files/build.gradle` pattern |

Default to **Expo prebuild-in-CI** when Expo is detected unless the user explicitly uses the without-prebuild pattern.

## Phase 1 — RN CLI Android (default validation)

```yaml
workflows:
  rn-android-debug:
    name: React Native Android debug
    instance_type: mac_mini_m2
    max_build_duration: 30
    environment:
      node: latest
    scripts:
      - name: Install dependencies
        script: npm install
      - name: Set Android SDK location
        script: echo "sdk.dir=$ANDROID_SDK_ROOT" > "$CM_BUILD_DIR/android/local.properties"
      - name: Build debug APK
        script: cd android && ./gradlew assembleDebug
    artifacts:
      - android/app/build/outputs/**/*.apk
```

## Phase 1 — Expo Android

Add prebuild before Gradle:

```yaml
      - name: Run Expo prebuild
        script: npx expo prebuild --platform android --non-interactive
```

## Phase 1 — RN iOS unsigned

```yaml
  rn-ios-unsigned:
    name: React Native iOS unsigned
    instance_type: mac_mini_m2
    max_build_duration: 60
    environment:
      node: latest
      xcode: latest
      cocoapods: default
      vars:
        XCODE_WORKSPACE: "YourApp.xcworkspace"
        XCODE_SCHEME: "YourApp"
    scripts:
      - name: Install dependencies
        script: npm install
      - name: Install CocoaPods
        script: cd ios && pod install
      - name: Build without signing
        script: |
          xcodebuild \
            -workspace "$CM_BUILD_DIR/ios/$XCODE_WORKSPACE" \
            -scheme "$XCODE_SCHEME" \
            -configuration Debug \
            -destination 'generic/platform=iOS' \
            CODE_SIGNING_ALLOWED=NO \
            build
    artifacts:
      - /tmp/xcodebuild_logs/*.log
```

Expo iOS: run `npx expo prebuild --platform ios --non-interactive` before CocoaPods.

## Phase 3+ — Signed release

Use [ios-release-lane.md](ios-release-lane.md) or [android-release-lane.md](android-release-lane.md). RN signed iOS typically uses `xcode-project build-ipa` after `xcode-project use-profiles`.

## Stack-specific failures

- Missing `android/` (Expo) → run prebuild
- `local.properties` missing → generate with `$ANDROID_SDK_ROOT`
- `cd` not persisting → merge into one script block

See [common-failures.md](common-failures.md).
