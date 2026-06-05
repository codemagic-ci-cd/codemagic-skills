# Native Android Stack (Phase 1+)

Published quickstart: https://docs.codemagic.io/yaml-quick-start/building-a-native-android-app/

## Phase 1 — Unsigned build

For projects with `android/` at repo root or standard Gradle layout:

```yaml
workflows:
  android-native-debug:
    name: Native Android debug
    instance_type: mac_mini_m2
    max_build_duration: 30
    environment:
      node: latest
    scripts:
      - name: Set Android SDK location
        script: echo "sdk.dir=$ANDROID_SDK_ROOT" > "$CM_BUILD_DIR/local.properties"
      - name: Build debug APK
        script: ./gradlew assembleDebug
    artifacts:
      - app/build/outputs/**/*.apk
```

If Gradle project is under `android/` subdirectory:

```yaml
      - name: Set Android SDK location
        script: echo "sdk.dir=$ANDROID_SDK_ROOT" > "$CM_BUILD_DIR/android/local.properties"
      - name: Build debug APK
        script: cd android && ./gradlew assembleDebug
    artifacts:
      - android/app/build/outputs/**/*.apk
```

## Phase 3+ — Signed release

Use [android-release-lane.md](android-release-lane.md) — `android_signing`, `./gradlew bundleRelease`.

## Notes

- Confirm `local.properties` path matches project layout (`local.properties` at Gradle root)
- `cd` must be in the same script block as Gradle

See [common-failures.md](common-failures.md).
