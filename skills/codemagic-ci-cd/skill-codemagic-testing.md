---
name: codemagic-testing
description: Support skill for configuring and troubleshooting automated testing on Codemagic. Covers unit tests, integration tests, and UI tests for Flutter, React Native, Native Android, and Native iOS. Includes test result publishing, emulator/simulator setup, and common test failures.
---

# Testing on Codemagic

## Overview

Tests run as script steps in `codemagic.yaml`, the same as any other build step. Codemagic does not have a dedicated "test" section — you run tests via shell commands and optionally publish results using the `test_report` artifacts key.

Docs: https://docs.codemagic.io/yaml-testing/testing/

---

## Flutter Testing

### Unit and widget tests
```yaml
scripts:
  - name: Run Flutter tests
    script: flutter test
```

### With test coverage
```yaml
scripts:
  - name: Run Flutter tests with coverage
    script: flutter test --coverage
artifacts:
  - coverage/lcov.info
```

### Integration tests (on emulator/simulator)
```yaml
scripts:
  - name: Run Flutter integration tests
    script: |
      flutter test integration_test/app_test.dart \
        -d emulator-5554
```

Requires an Android emulator or iOS simulator to be running. See emulator/simulator setup below.

### Publishing Flutter test results
```yaml
artifacts:
  - flutter_drive.log
```

There is no native JUnit XML output from `flutter test` by default. Use the `flutter_test_reporter` package or `--machine` flag with a custom parser to get structured results.

---

## Android / Native Testing

### Unit tests
```yaml
scripts:
  - name: Run Android unit tests
    script: ./gradlew test
artifacts:
  - app/build/reports/tests/**/*.xml
  - app/build/test-results/**/*.xml
```

### Instrumented tests (on emulator)
```yaml
scripts:
  - name: Run Android instrumented tests
    script: ./gradlew connectedAndroidTest
artifacts:
  - app/build/reports/androidTests/**
```

Requires an Android emulator. See emulator setup below.

### Publishing test results (JUnit XML)
```yaml
artifacts:
  - app/build/test-results/**/*.xml

publishing:
  email:
    recipients:
      - you@example.com
```

Codemagic parses JUnit XML files from the `artifacts` section and displays test results in the build UI.

---

## iOS / Native Testing

### Unit tests
```yaml
scripts:
  - name: Run iOS unit tests
    script: |
      xcodebuild test \
        -project Runner.xcodeproj \
        -scheme Runner \
        -destination 'platform=iOS Simulator,name=iPhone 15,OS=latest' \
        -resultBundlePath TestResults
artifacts:
  - TestResults.xcresult
```

### Publishing iOS test results
```yaml
artifacts:
  - TestResults.xcresult
```

Codemagic displays `.xcresult` bundles in the build UI with pass/fail breakdown.

---

## React Native Testing

### Jest unit tests
```yaml
scripts:
  - name: Run Jest tests
    script: yarn test --ci --reporters=default --reporters=jest-junit
artifacts:
  - junit.xml
```

Requires `jest-junit` package for JUnit XML output:
```bash
yarn add --dev jest-junit
```

---

## Android Emulator Setup

To run instrumented or integration tests, start an emulator before the test step:

```yaml
scripts:
  - name: Launch Android emulator
    script: |
      cd $ANDROID_HOME/tools
      echo "no" | ./android create avd --force -n test -t "system-images;android-30;google_apis;x86_64" --abi google_apis/x86_64
      nohup $ANDROID_HOME/emulator/emulator -avd test -no-audio -no-window &
      adb wait-for-device
      adb shell input keyevent 82
```

Pre-installed emulator images vary by machine — check docs.codemagic.io/specs/ for available system images on Linux machines.

---

## iOS Simulator Setup

iOS simulators are available on macOS machines. Boot one before running tests:

```yaml
scripts:
  - name: Boot iOS simulator
    script: |
      UDID=$(xcrun simctl list devices available | grep "iPhone 15" | head -1 | awk -F'[()]' '{print $2}')
      xcrun simctl boot "$UDID"
```

To list available simulators on the machine:
```bash
xcrun simctl list devices available
```

For a specific iOS version that is not pre-installed, it can be downloaded with:
```bash
xcodebuild -downloadPlatform iOS -buildVersion <version>
```
This adds approximately 5–10 minutes to the build. Only suggest this after confirming the version is actually available — very new iOS releases may not be published by Apple yet.

---

## Test Result Publishing

Codemagic reads test results from artifacts and displays them in the build UI:

| Format | Platforms | Artifact path |
|---|---|---|
| JUnit XML | Android, React Native, any | `**/test-results/**/*.xml` |
| `.xcresult` | iOS/macOS | `*.xcresult` |
| HTML reports | Any | `**/reports/**/*.html` |

```yaml
artifacts:
  - app/build/test-results/**/*.xml   # Android JUnit
  - TestResults.xcresult              # iOS
  - coverage/lcov.info                # coverage report
```

---

## Common Test Failures

### Tests pass locally but fail on Codemagic
- Check tool versions — Node, Java, Flutter version may differ between local and CI
- Check that test dependencies are installed (e.g. `flutter pub get`, `npm install`)
- Check that required environment variables are available in the test step

### Emulator/simulator not found
- List available devices: `xcrun simctl list devices` (iOS) or `adb devices` (Android)
- Ensure the emulator is fully booted before running tests — add a wait step

### Tests timing out
- UI/integration tests can be slow — increase the step timeout:
  ```yaml
  scripts:
    - name: Run integration tests
      script: flutter test integration_test/
      timeout: 30  # minutes
  ```

### "No tests were found"
- Wrong path to test files
- Test files not matching the expected naming convention (e.g. `*_test.dart` for Flutter, `*.test.js` for Jest)

### Android instrumented tests failing on CI only
- Screen lock active — unlock the emulator before running tests:
  ```bash
  adb shell input keyevent 82
  ```
- Animation scale causing flakiness — disable animations:
  ```bash
  adb shell settings put global window_animation_scale 0
  adb shell settings put global transition_animation_scale 0
  adb shell settings put global animator_duration_scale 0
  ```

---

## Debugging Checklist

1. Are test dependencies installed before the test step runs?
2. Is the emulator/simulator booted and ready before tests start?
3. Are test result artifact paths correct?
4. Do the tool versions match what the tests expect?
5. Are required environment variables available in the test step?
6. Is the test step timeout long enough for UI/integration tests?

---

## Key Documentation Links

- Testing overview: https://docs.codemagic.io/yaml-testing/testing/
- Android testing: https://docs.codemagic.io/yaml-testing/android-instrumented-testing/
- iOS testing: https://docs.codemagic.io/yaml-testing/ios-testing/
- Flutter integration tests: https://docs.codemagic.io/yaml-testing/flutter-driver-testing/
- Machine specs (Linux/Android): https://docs.codemagic.io/specs/
- Machine specs (macOS/iOS): https://docs.codemagic.io/specs-macos/
