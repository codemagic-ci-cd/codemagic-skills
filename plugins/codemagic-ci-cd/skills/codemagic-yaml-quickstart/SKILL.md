---
name: codemagic-yaml-quickstart
description: Agentic skill for generating a starter codemagic.yaml for new users. Covers all supported framework and platform combinations (Flutter, React Native, Native iOS, Native Android), iOS and Android distribution targets, machine selection, signing configuration, artifact paths, and publishing blocks. For use by AI agents autonomously producing a working first codemagic.yaml for a customer.
---

# YAML Quickstart Generator — Agentic Skill

## Purpose

Produce a complete, ready-to-commit `codemagic.yaml` for a new Codemagic user based on their framework, platform, distribution target, and machine preference. The output must be a working starting point — not pseudocode. Replace placeholder values with comments so the customer knows exactly what to fill in. Execute all steps autonomously.

---

## Preconditions

Before generating the YAML, assert all of the following are known:

- [ ] Framework: Flutter / React Native / Native iOS / Native Android
- [ ] Platform(s): iOS / Android / Both
- [ ] iOS distribution target (if applicable): App Store / TestFlight / Firebase / Build only
- [ ] Android distribution target (if applicable): Google Play / Firebase / Build only
- [ ] Machine type: `mac_mini_m2` or `mac_mini_m4`

If any of the above is unknown, ask for it before generating. Do not assume.

> **Note:** Native iOS apps are iOS-only by definition. Native Android apps are Android-only by definition. Only Flutter and React Native need a platform selection step.

---

## Step 1 — Select machine type

| Machine | Spec | Use when |
|---|---|---|
| `mac_mini_m2` | 8-core M2, 8GB RAM | Most projects — standard builds |
| `mac_mini_m4` | 10-core M4, 16GB RAM | Large projects, slow builds, heavy test suites |

> **Rule:** iOS and macOS builds always require a Mac instance. Android-only builds can use Linux (`linux_x2` — cheaper), but mac_mini is the default safe choice.

---

## Step 2 — Determine iOS signing configuration

The `distribution_type` under `ios_signing` must match the intended distribution target:

| Distribution target | `distribution_type` |
|---|---|
| App Store (production release) | `app_store` |
| TestFlight (beta) | `app_store` |
| Firebase App Distribution | `ad_hoc` |
| Build only (no distribution) | `development` |

> **Critical:** Using the wrong profile type is the most common first-build signing failure. App Store and TestFlight both use `app_store` — they differ only in the `publishing:` block, not in signing.

---

## Step 3 — Determine environment groups and credentials needed

| Distribution | Required group | Required variable |
|---|---|---|
| Google Play | `google_play_credentials` | `GCLOUD_SERVICE_ACCOUNT_CREDENTIALS` |
| Firebase (iOS or Android) | `firebase_credentials` | `FIREBASE_SERVICE_ACCOUNT` |
| App Store / TestFlight | App Store Connect API key (via Integrations, not env group) | Set up in Team Settings → Integrations → Developer Portal |
| Build only | None | None |

---

## Step 4 — Generate the YAML

Use the combinations below to assemble the correct workflow(s). For "Both" platform selections, generate two separate workflows (`ios-workflow` and `android-workflow`) in the same file.

---

### Script blocks by framework and platform

**Flutter — iOS:**
```yaml
scripts:
  - name: Get Flutter packages
    script: flutter pub get
  - name: Install CocoaPods
    script: cd ios && pod install
  - name: Set up code signing
    script: xcode-project use-profiles
  - name: Build IPA
    script: flutter build ipa --release
```

**Flutter — Android:**
```yaml
scripts:
  - name: Get Flutter packages
    script: flutter pub get
  - name: Build AAB
    script: flutter build appbundle --release
```

**React Native — iOS:**
```yaml
scripts:
  - name: Install dependencies
    script: npm install
  - name: Install CocoaPods
    script: cd ios && pod install
  - name: Set up code signing
    script: xcode-project use-profiles
  - name: Build IPA
    script: |
      xcode-project build-ipa \
        --workspace ios/YourApp.xcworkspace \
        --scheme YourApp
```

**React Native — Android:**
```yaml
scripts:
  - name: Install dependencies
    script: npm install
  - name: Build AAB
    script: cd android && ./gradlew bundleRelease
```

**Native iOS:**
```yaml
scripts:
  - name: Set up code signing
    script: xcode-project use-profiles
  - name: Build IPA
    script: |
      xcode-project build-ipa \
        --workspace YourApp.xcworkspace \
        --scheme YourApp
```

**Native Android:**
```yaml
scripts:
  - name: Build AAB
    script: ./gradlew bundleRelease
```

---

### Artifact paths by framework and platform

| Framework | Platform | Artifact path |
|---|---|---|
| Flutter | iOS | `build/ios/ipa/*.ipa` |
| Flutter | Android | `build/**/outputs/**/*.aab` |
| React Native | iOS | `build/ios/ipa/*.ipa` |
| React Native | Android | `android/app/build/outputs/**/*.aab` |
| Native iOS | iOS | `build/ios/ipa/*.ipa` |
| Native Android | Android | `app/build/outputs/**/*.aab` |

---

### Environment blocks by framework

**Flutter iOS:**
```yaml
environment:
  ios_signing:
    distribution_type: app_store  # or ad_hoc / development — see Step 2
    bundle_identifier: com.example.app  # replace with your bundle ID
  xcode: latest
  flutter: stable
```

**Flutter Android:**
```yaml
environment:
  android_signing:
    - your_keystore_reference  # replace with keystore name from Code Signing Identities
  groups:
    - google_play_credentials  # or firebase_credentials
  flutter: stable
```

**React Native iOS:**
```yaml
environment:
  ios_signing:
    distribution_type: app_store
    bundle_identifier: com.example.app
  xcode: latest
  node: latest
```

**React Native Android:**
```yaml
environment:
  android_signing:
    - your_keystore_reference
  groups:
    - google_play_credentials  # or firebase_credentials
  node: latest
```

**Native iOS:**
```yaml
environment:
  ios_signing:
    distribution_type: app_store
    bundle_identifier: com.example.app
  xcode: latest
```

**Native Android:**
```yaml
environment:
  android_signing:
    - your_keystore_reference
  groups:
    - google_play_credentials  # or firebase_credentials
```

---

### Publishing blocks by distribution target

**App Store:**
```yaml
publishing:
  app_store_connect:
    auth: integration
    submit_to_testflight: false
    submit_to_app_store: true
    release_type: AFTER_APPROVAL
  email:
    recipients:
      - your@email.com
    notify:
      success: true
      failure: true
```

**TestFlight:**
```yaml
publishing:
  app_store_connect:
    auth: integration
    submit_to_testflight: true
    submit_to_app_store: false
  email:
    recipients:
      - your@email.com
    notify:
      success: true
      failure: true
```

**Firebase — iOS:**
```yaml
publishing:
  firebase:
    firebase_service_account: $FIREBASE_SERVICE_ACCOUNT
    ios:
      app_id: x:xxxxxxxxxxxx:ios:xxxxxxxxxxxxxxxxxxxxxx  # replace with Firebase App ID
      groups:
        - ios-testers
  email:
    recipients:
      - your@email.com
    notify:
      success: true
      failure: true
```

**Firebase — Android:**
```yaml
publishing:
  firebase:
    firebase_service_account: $FIREBASE_SERVICE_ACCOUNT
    android:
      app_id: x:xxxxxxxxxxxx:android:xxxxxxxxxxxxxxxxxxxxxx  # replace with Firebase App ID
      groups:
        - android-testers
      artifact_type: 'aab'
  email:
    recipients:
      - your@email.com
    notify:
      success: true
      failure: true
```

**Google Play:**
```yaml
publishing:
  google_play:
    credentials: $GCLOUD_SERVICE_ACCOUNT_CREDENTIALS
    track: internal  # internal | alpha | beta | production
  email:
    recipients:
      - your@email.com
    notify:
      success: true
      failure: true
```

**Build only:**
```yaml
publishing:
  email:
    recipients:
      - your@email.com
    notify:
      success: true
      failure: true
```

---

### Integrations block (App Store and TestFlight only)

Add this above `environment:` when distributing to App Store or TestFlight:

```yaml
integrations:
  app_store_connect: YOUR_API_KEY_NAME  # set up in Team Settings → Integrations → Developer Portal
```

---

## Step 5 — Assemble the full YAML

**Template structure:**

```yaml
workflows:
  ios-workflow:                        # rename if desired
    name: <descriptive name>
    instance_type: mac_mini_m2         # or mac_mini_m4
    integrations:                      # App Store / TestFlight only
      app_store_connect: YOUR_KEY_NAME
    environment:
      <environment block from Step 4>
    scripts:
      <script block from Step 4>
    artifacts:
      - <artifact path from Step 4>
    publishing:
      <publishing block from Step 4>

  android-workflow:                    # include only if building Android
    name: <descriptive name>
    instance_type: mac_mini_m2
    environment:
      <environment block from Step 4>
    scripts:
      <script block from Step 4>
    artifacts:
      - <artifact path from Step 4>
    publishing:
      <publishing block from Step 4>
```

---

## Full worked examples

### Flutter — Both platforms — TestFlight + Google Play — Mac Mini M2

```yaml
workflows:
  ios-workflow:
    name: iOS TestFlight
    instance_type: mac_mini_m2
    integrations:
      app_store_connect: YOUR_API_KEY_NAME  # set up in Team Settings → Integrations → Developer Portal
    environment:
      ios_signing:
        distribution_type: app_store
        bundle_identifier: com.example.app  # replace with your bundle ID
      xcode: latest
      flutter: stable
    scripts:
      - name: Get Flutter packages
        script: flutter pub get
      - name: Install CocoaPods
        script: cd ios && pod install
      - name: Set up code signing
        script: xcode-project use-profiles
      - name: Build IPA
        script: flutter build ipa --release
    artifacts:
      - build/ios/ipa/*.ipa
    publishing:
      app_store_connect:
        auth: integration
        submit_to_testflight: true
        submit_to_app_store: false
      email:
        recipients:
          - your@email.com
        notify:
          success: true
          failure: true

  android-workflow:
    name: Android Google Play
    instance_type: mac_mini_m2
    environment:
      android_signing:
        - your_keystore_reference  # replace with keystore name from Code Signing Identities
      groups:
        - google_play_credentials
      flutter: stable
    scripts:
      - name: Get Flutter packages
        script: flutter pub get
      - name: Build AAB
        script: flutter build appbundle --release
    artifacts:
      - build/**/outputs/**/*.aab
    publishing:
      google_play:
        credentials: $GCLOUD_SERVICE_ACCOUNT_CREDENTIALS
        track: internal
      email:
        recipients:
          - your@email.com
        notify:
          success: true
          failure: true
```

---

### React Native — iOS only — Firebase — Mac Mini M2

```yaml
workflows:
  ios-workflow:
    name: iOS Firebase
    instance_type: mac_mini_m2
    environment:
      ios_signing:
        distribution_type: ad_hoc
        bundle_identifier: com.example.app  # replace with your bundle ID
      xcode: latest
      node: latest
      groups:
        - firebase_credentials
    scripts:
      - name: Install dependencies
        script: npm install
      - name: Install CocoaPods
        script: cd ios && pod install
      - name: Set up code signing
        script: xcode-project use-profiles
      - name: Build IPA
        script: |
          xcode-project build-ipa \
            --workspace ios/YourApp.xcworkspace \
            --scheme YourApp
    artifacts:
      - build/ios/ipa/*.ipa
    publishing:
      firebase:
        firebase_service_account: $FIREBASE_SERVICE_ACCOUNT
        ios:
          app_id: x:xxxxxxxxxxxx:ios:xxxxxxxxxxxxxxxxxxxxxx  # replace with Firebase App ID
          groups:
            - ios-testers
      email:
        recipients:
          - your@email.com
        notify:
          success: true
          failure: true
```

---

## What to tell the customer after generating

Always include these next steps:

1. **Place the file** at the repository root as `codemagic.yaml`
2. **Replace all placeholder values** — everything marked with a `#` comment
3. **Set up credentials** before the first build:
   - iOS: App Store Connect API key in Team Settings → Integrations → Developer Portal
   - Android keystore: Teams → Code Signing Identities → Android keystores
   - Google Play: `GCLOUD_SERVICE_ACCOUNT_CREDENTIALS` in environment variables group `google_play_credentials`
   - Firebase: `FIREBASE_SERVICE_ACCOUNT` in environment variables group `firebase_credentials`
4. **Commit and push** — the build will appear in the Codemagic dashboard

---

## Common first-build failures and fixes

| Error | Root cause | Fix |
|---|---|---|
| No matching provisioning profile | `bundle_identifier` placeholder not replaced | Update `bundle_identifier` to the actual app bundle ID |
| No certificate found | App Store Connect API key not set up | Set up key in Team Settings → Integrations → Developer Portal |
| Keystore not found | Keystore reference name doesn't match uploaded name | Check Code Signing Identities → Android keystores for the exact reference name |
| `flutter` command not found | `flutter:` not set in environment | Add `flutter: stable` under `environment:` |
| `pod install` fails | CocoaPods not up to date or Podfile lock conflict | Add `pod repo update` before `pod install` |
| AAB not collected | Wrong artifact path | Check actual output path by adding `find build -name "*.aab"` as a debug script step |
| Firebase: `app not found` | Firebase App ID placeholder not replaced | Copy App ID from Firebase Console → Project Settings → General → Your apps |
| Google Play: permission denied | Service account missing release permissions | Assign 'Release manager' role to the service account in Google Play Console |

---

## Invariants

Assert before marking skill complete:

1. The `distribution_type` matches the intended distribution target — App Store and TestFlight both use `app_store`; Firebase uses `ad_hoc`
2. Every environment group referenced in `environment.groups` has its credentials added in Codemagic
3. The artifact path pattern matches the framework's actual build output location
4. App Store / TestFlight workflows include the `integrations.app_store_connect` block
5. Placeholder values (`com.example.app`, `YOUR_API_KEY_NAME`, `your_keystore_reference`, Firebase App IDs) are clearly marked with `#` comments
6. The file is named exactly `codemagic.yaml` and placed at the repository root

---

## Scope boundaries

This skill does NOT handle:

- Detailed signing setup (certificate creation, keystore generation) — see iOS/Android signing skills
- Build triggers and branching strategy — see build triggers skill
- Caching, test configuration, or build optimisation
- Shorebird / code push workflows — see CodePush skill
- Monorepo multi-app routing

---

## Reference documentation

| Topic | URL |
|---|---|
| YAML structure overview | https://docs.codemagic.io/yaml-basic-configuration/yaml-getting-started/ |
| Flutter builds | https://docs.codemagic.io/yaml-quick-start/building-a-flutter-app/ |
| React Native builds | https://docs.codemagic.io/yaml-quick-start/building-a-react-native-app/ |
| iOS code signing | https://docs.codemagic.io/yaml-code-signing/ios-code-signing/ |
| Android code signing | https://docs.codemagic.io/yaml-code-signing/android-code-signing/ |
| App Store Connect publishing | https://docs.codemagic.io/yaml-publishing/app-store-connect/ |
| Google Play publishing | https://docs.codemagic.io/yaml-publishing/google-play/ |
| Firebase App Distribution | https://docs.codemagic.io/yaml-distributing/firebase-app-distribution/ |
| Machine types | https://docs.codemagic.io/specs-macos/ |
