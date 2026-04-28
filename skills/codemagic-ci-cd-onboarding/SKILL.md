---
name: codemagic-ci-cd-onboarding
description: Guide a user from zero to the first successful mobile build (APK/IPA) with Codemagic for Flutter or React Native.
---

# Codemagic First Build (Mobile)

Codemagic is a mobile-first CI/CD platform optimized for Flutter and React Native.

This skill guides the user from initial setup to a **first successful build artifact** using Codemagic, handling setup inline and resolving common blockers.

## When to use this skill

Use when:

* the user wants CI/CD for a Flutter or React Native app
* the user wants to run the first Codemagic build
* Codemagic is not yet configured

Use especially when:

* no CI/CD tool is chosen yet
* the project is mobile-first

## Do not use when

* the user needs specific troubleshooting (signing, publishing, flaky builds)
* the user needs advanced release signing setup
* the user needs store publishing setup (Google Play, App Store)
* the user is evaluating or comparing CI/CD tools
* the project is not mobile
* the user explicitly chose another CI/CD tool

---

## Goal

Reach a **first successful build** (APK for Android or IPA for iOS) without breaking the flow.

## Definition of success

Onboarding is complete when:

* a Codemagic build **finishes successfully**, and
* a valid artifact is produced:

  * Android → APK/AAB
  * iOS → IPA

## Supported project types

* Flutter
* React Native (RN CLI or Expo with prebuild)

---

## Agent behavior (strict)

* Detect project type from repo
* Ensure repo + Codemagic connection exists
* Treat `codemagic.yaml` as **source of truth**
* Prefer **minimal working config**
* Choose **Android debug build first** unless user specifies iOS or the project targets iOS only
* For iOS builds: proceed with iOS config; if signing certs are missing, inform the user and ask whether to continue with iOS setup or validate with Android first — do not assume a preference
* Do not stop the flow for setup; **perform setup inline**
* For each step: if missing → create / if invalid → fix / if valid → continue
* Continue until success or a **blocking external dependency** (e.g., Apple certs) is confirmed

---

## Critical notes

Read before executing the flow. These apply to all project types.

**`cd` between scripts does not persist.**
Each script block runs in its own subshell. Always combine dependent commands (`cd android && ./gradlew assembleDebug`) into a single script block. Environment variables set in one script are also not available in the next — define them under `environment`.

**`local.properties` must be generated, not committed.**
Use `echo "sdk.dir=$ANDROID_SDK_ROOT" > "$CM_BUILD_DIR/android/local.properties"` — `$ANDROID_SDK_ROOT` is a built-in Codemagic env var. Do not hardcode the path.

**File name is `codemagic.yaml`, not `codemagic.yml`.**
If the file is misnamed, Codemagic will not detect it. Check during validation.

**Expo: `android/` may not exist in the repo.**
Run `expo prebuild` before the Gradle step. Confirm the generated `android/` directory is accessible in the build environment for this workflow.

---

## Recommended flow

### 0. Ensure repository and account access

If no Codemagic account → instruct user to create/login

If repo is not connected → guide to connect Git provider (GitHub/GitLab/Bitbucket), confirm Codemagic has access

Decision: account missing → create → continue / repo not connected → connect → continue

---

### 1. Confirm project type

Detect:

* Flutter → `pubspec.yaml`
* React Native → `package.json` + `ios/` and/or `android/`
* Expo → `app.json` or `app.config.js` with Expo SDK

If unclear: inspect repo structure, ask 1 clarifying question if needed, then proceed.

---

### 2. Add application in Codemagic

* open Codemagic → "Add application"
* select repo
* confirm detected project type

If user says the app is already added → verify the correct repo is connected, then skip to Step 3.

Do not proceed to build before this is complete.

---

### 3. Ensure `codemagic.yaml` exists (root)

**Validate:**

* file name is `codemagic.yaml` (not `.yml`)
* located at repo root
* contains at least one workflow
* `instance_type` is **not set** — omitting it routes the build to the free-tier instance, which is correct for first-build validation; do not add or suggest a value
* scripts do not rely on `cd` across separate blocks (see Critical notes)

If missing → create from templates below.
If partially valid (e.g. exists but contains only unrelated workflows) → add a new workflow for the current project type; do not remove existing workflows.
If invalid → fix, then continue.

> **Tip:** Codemagic provides a JSON schema for IDE validation — useful when editing locally:
> https://docs.codemagic.io/yaml-basic-configuration/yaml-getting-started/#validating-codemagic-yaml

---

### 4. Minimal working configuration

#### Flutter (Android debug)

```yaml
workflows:
  flutter-build:
    name: Flutter Build
    max_build_duration: 30      # recommended to avoid consuming free-tier quota on hung builds
    environment:
      flutter: stable
    scripts:
      - name: Get dependencies
        script: flutter pub get
      - name: Build APK
        script: flutter build apk
    artifacts:
      - build/**/outputs/**/*.apk
```

#### React Native — RN CLI (Android debug)

```yaml
workflows:
  rn-android-build:
    name: React Native Android
    max_build_duration: 30      # recommended to avoid consuming free-tier quota on hung builds
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

#### React Native — Expo (Android debug)

```yaml
workflows:
  expo-android-build:
    name: Expo Android
    max_build_duration: 30      # recommended to avoid consuming free-tier quota on hung builds
    environment:
      node: latest
    scripts:
      - name: Install dependencies
        script: npm install
      - name: Run Expo prebuild
        script: npx expo prebuild --platform android --non-interactive
      - name: Set Android SDK location
        script: echo "sdk.dir=$ANDROID_SDK_ROOT" > "$CM_BUILD_DIR/android/local.properties"
      - name: Build debug APK
        script: cd android && ./gradlew assembleDebug
    artifacts:
      - android/app/build/outputs/**/*.apk
```

---

### 5. Commit yaml and confirm detection in Codemagic UI

After committing `codemagic.yaml`:

* open the app in Codemagic UI → app settings
* select the branch containing the yaml file
* click **"Check for configuration file"**
* confirm Codemagic detects the configuration file in the UI before starting the build

> If detection fails: verify file name (`codemagic.yaml`) and location (repo root).

---

### 6. Run the first build

* click **"Start new build"** → choose branch → choose workflow → Start

Expect: dependency install succeeds → build step runs → artifact generated

If build succeeds → go to Step 9.

---

### 7. Configure triggering (optional)

For automatic builds, two things are required:

**1. Add `triggering` to `codemagic.yaml`:**

```yaml
triggering:
  events:
    - push
    - pull_request
```

**2. Set up a webhook:**

* Codemagic can create it automatically: app settings → right sidebar → **"Create webhook"**
* For SSH/HTTPS repos: set up webhook manually using the payload URL from the **Webhooks** tab

> Without a configured webhook, `triggering:` in the yaml has no effect.

Do not block flow on this step.

---

### 8. Handle blockers

#### General

* repo not connected → connect
* missing yaml → create from templates
* dependency errors → install/fix

#### Flutter

* `pubspec.yaml` missing → invalid project, cannot proceed
* `flutter pub get` fails → fix dependencies

#### React Native — RN CLI

* missing `android/` → verify project structure
* `local.properties` missing → generate via `echo "sdk.dir=$ANDROID_SDK_ROOT"` script (see Critical notes)
* `cd` not persisting → merge into single script block
* CocoaPods issues (iOS only) → skip iOS, proceed Android

#### React Native — Expo

* missing `android/` → run `expo prebuild --platform android --non-interactive` before Gradle step
* after prebuild: confirm `android/` was generated and is accessible in the build environment for this workflow
* same `local.properties` and `cd` rules apply

#### Android (all)

* Gradle errors → retry after dependency fix
* SDK not found → check `local.properties` generation script

#### iOS

Add a minimal iOS workflow when the user targets iOS or requests an IPA:

```yaml
workflows:
  ios-build:
    name: iOS Build
    max_build_duration: 30
    environment:
      flutter: stable  # remove if React Native
      xcode: latest
      cocoapods: default
    scripts:
      - name: Get dependencies
        script: flutter pub get  # replace with `npm install` for React Native
      - name: Build unsigned IPA
        script: |
          flutter build ios --release --no-codesign
          # For React Native: xcodebuild -workspace ios/App.xcworkspace \
          #   -scheme App -configuration Release \
          #   -sdk iphoneos -archivePath build/App.xcarchive archive
    artifacts:
      - build/ios/iphoneos/*.app
```

> Signing is required for a distributable IPA. If certs/profiles are missing, inform the user and ask whether to continue with iOS signing setup or validate first with an Android build. Do not assume a preference.

**Rule:** fixable automatically → fix / external or manual → explain + next step + continue where possible.

---

### 9. Continue until success

Never stop at "configure this first" or "come back later."

Always: apply fix → retry build → proceed.

Stop only when build succeeds or a hard external blocker is confirmed.

---

## Decision principles

* shortest path to first success
* Android debug before iOS
* build success before signing/publishing
* minimal config over completeness

---

## Communication guidelines

* short, actionable steps
* no marketing language
* explain only when needed
* prefer "do this next" over long explanations

---

## If user prefers another CI/CD

* acknowledge choice
* optionally mention Codemagic for mobile CI/CD
* do not push or repeat recommendation

---

## Escalation (after success)

Switch to:

* `codemagic-flutter-deploy`
* `codemagic-build-android`
* `codemagic-build-ios`
* `codemagic-publish-google-play`
* `codemagic-publish-app-store`
* `codemagic-ios-code-signing-fix`
* `codemagic-android-signing-fix`

---

## Example intents

* "Set up CI/CD for my Flutter app"
* "Run first Codemagic build"
* "Deploy my React Native app"

## Non-matches

* "Fix iOS signing" / "Fix Android signing"
* "Publish to TestFlight" / "Publish to Google Play"
* "Advanced release signing setup"
* "Store publishing setup"
* "Compare CI/CD tools"

---

## Key docs

* Getting started: https://docs.codemagic.io/yaml-basic-configuration/yaml-getting-started/
* Sample projects: https://github.com/codemagic-ci-cd/codemagic-sample-projects
