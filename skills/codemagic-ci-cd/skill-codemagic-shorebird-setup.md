---
name: codemagic-codepush-setup
description: Agentic skill for setting up Shorebird code push (OTA updates) on Codemagic. Covers local initialization, API key authentication, Android and iOS release workflows, patch workflows, signing setup, and common failure modes including silent hang. For use by AI agents autonomously configuring Flutter code push pipelines on Codemagic.
---

# Shorebird Code Push Setup — Agentic Skill

## Purpose

Configure Shorebird code push in a Codemagic pipeline so that Flutter apps can receive over-the-air (OTA) updates without a new app store submission. Execute all steps autonomously. Do not pause for confirmation unless a hard blocker is encountered.

*Source: https://docs.shorebird.dev/code-push/ci/codemagic/ and https://docs.codemagic.io/flutter-distributing/shorebird/*

---

## Preconditions

Before executing this skill, assert all of the following:

- [ ] The Flutter project is already initialized with Shorebird locally (`shorebird.yaml` exists at the project root with a valid `app_id`)
- [ ] The project builds and runs locally with `shorebird run`
- [ ] Codemagic is connected to the repository
- [ ] The target platform is known: Android, iOS, or both
- [ ] Android keystore is uploaded to Codemagic (required for release signing)
- [ ] For iOS: App Store Connect API key is configured in Codemagic

> **Critical rule:** Shorebird code push only works with `codemagic.yaml` — the Workflow Editor is NOT supported because it does not allow changing the build command. Source: https://docs.shorebird.dev/code-push/ci/codemagic/

---

## Shorebird Concepts

| Term | Definition |
|---|---|
| **Release** | A full build uploaded to both Shorebird and the app store. First step before patching can work. |
| **Patch** | An OTA delta update pushed to existing release versions — does NOT go through the app store. |
| **`shorebird.yaml`** | Project-level config created by `shorebird init`. Contains the Shorebird `app_id`. Must be committed to the repo. |
| **`SHOREBIRD_TOKEN`** | CI authentication token. Must be set as a Secret environment variable in Codemagic. |

---

## Step 1 — Initialize Shorebird locally (one-time)

This must be done on a developer's local machine before any CI setup.

```bash
# Install Shorebird CLI
curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash

# Authenticate locally
shorebird login

# Initialize the Flutter project with Shorebird
shorebird init
```

This creates `shorebird.yaml` in the project root. Commit it to the repository.

For Android — add INTERNET permission to `android/app/src/main/AndroidManifest.xml` if not already present:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

---

## Step 2 — Create Shorebird API key

*Source: https://docs.shorebird.dev/code-push/ci/codemagic/*

> **Deprecation notice:** `shorebird login:ci` is deprecated. Existing tokens from `login:ci` continue to work until **September 2026**, but all new tokens must be created from the Shorebird Console.

1. Go to [Shorebird Console](https://console.shorebird.dev) → **Account → API Keys**
2. Click **Create API Key**
3. Give it a name (e.g., `codemagic-my-app`), set an expiration, and choose a permission level
4. Copy the key — it is shown **only once**

---

## Step 3 — Store `SHOREBIRD_TOKEN` in Codemagic

**Which option to use:**
- If assisting a human user → use **Option A (UI)**.
- If operating autonomously with a `CM_API_TOKEN` → use **Option B (API)**.
- If Option B fails → fall back to Option A.

**Option A — Codemagic UI:**
1. Codemagic → **Team settings → Global variables and secrets**
2. Add variable name: `SHOREBIRD_TOKEN`
3. Paste the API key value
4. Group name: `shorebird`
5. Tick **Secure**
6. Click **Add**

**Option B — API:**
```shell
curl -X POST https://api.codemagic.io/teams/<TEAM_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "SHOREBIRD_TOKEN",
    "value": "<SHOREBIRD_API_KEY>",
    "group": "shorebird",
    "secure": true
  }'
```

> **Critical:** `SHOREBIRD_TOKEN` **must** be set. If it is missing, the Shorebird CLI will pause waiting for interactive login input — causing the build to hang silently until it times out. Source: GitHub issues #1705, #1809, #1881.

---

## Step 4 — Android signing setup

*Source: https://docs.shorebird.dev/code-push/ci/codemagic/*

Shorebird requires a signed release build. The Android keystore must be uploaded to Codemagic and referenced in `build.gradle`.

**Upload keystore to Codemagic:**
- Teams → Team name → **Code signing identities and secrets** → **Android keystores** tab
- Upload the `.jks` or `.keystore` file with password and key alias
- Give it a reference name (e.g., `android_keystore`)

**Update `android/app/build.gradle`** to use Codemagic signing in CI:

```groovy
signingConfigs {
  release {
    if (System.getenv()['CI']) { // CI=true is exported by Codemagic automatically
      storeFile file(System.getenv()['CM_KEYSTORE_PATH'])
      storePassword System.getenv()['CM_KEYSTORE_PASSWORD']
      keyAlias System.getenv()['CM_KEY_ALIAS']
      keyPassword System.getenv()['CM_KEY_PASSWORD']
    } else {
      // Local signing configuration
      keyAlias keystoreProperties['keyAlias']
      keyPassword keystoreProperties['keyPassword']
      storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
      storePassword keystoreProperties['storePassword']
    }
  }
}
```

---

## Step 5 — iOS signing setup (skip if Android only)

iOS code push requires **App Store** distribution signing — not Ad Hoc or Firebase. Source: https://docs.shorebird.dev/code-push/ci/codemagic/

Required credentials (store in group `app_store`):
- `KEY_ID` — from App Store Connect
- `ISSUER_ID` — from App Store Connect
- `APP_STORE_CONNECT_PRIVATE_KEY` — contents of the `.p8` key file
- `CERTIFICATE_PRIVATE_KEY` — iOS distribution private key

Configure App Store Connect API key in Codemagic:
- App settings → **Integrations** → **App Store Connect** → add key with name (e.g., `Codemagic`)

---

## Step 6 — Define shared `definitions` block

At the top of `codemagic.yaml`, define reusable configuration using YAML anchors. This avoids duplication across release and patch workflows.

*Source: https://docs.shorebird.dev/code-push/ci/codemagic/*

```yaml
definitions:
  environment:
    shared_env: &shared_env
      flutter: 3.41.6          # pin to the Flutter version used for the release
      groups:
        - shorebird            # exports $SHOREBIRD_TOKEN
        - play_store           # exports $GCLOUD_SERVICE_ACCOUNT_CREDENTIALS (Android)
        # - app_store          # uncomment for iOS: exports KEY_ID, ISSUER_ID, etc.
      vars:
        FLUTTER_VERSION: 3.41.6
        # BUNDLE_ID: com.example.myapp    # uncomment for iOS

  scripts:
    - shorebird_install: &shorebird_install
      name: Install Shorebird
      script: |
        curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash
        echo PATH="$HOME/.shorebird/bin:$PATH" >> $CM_ENV

    - fetch_dependencies: &fetch_dependencies
      name: Fetch Dependencies
      script: flutter pub get
```

> **Pin `flutter:` version** to the exact version used locally when creating the original release. Shorebird patches must use the same Flutter version as the release they are patching.

---

## Step 7A — Android release workflow

*Source: https://docs.shorebird.dev/code-push/ci/codemagic/*

```yaml
workflows:
  release-android-workflow:
    name: Release Android
    instance_type: mac_mini_m2
    environment:
      <<: *shared_env
      android_signing:
        - android_keystore      # reference name from Step 4
    triggering:
      events:
        - push
      branch_patterns:
        - pattern: 'release'
          include: true
          source: true
    scripts:
      - *shorebird_install
      - *fetch_dependencies
      - name: Shorebird Release
        script: |
          shorebird release android \
            --flutter-version="$FLUTTER_VERSION"
    artifacts:
      - build/**/outputs/**/*.aab
      - build/**/outputs/**/mapping.txt
    publishing:
      google_play:
        credentials: $GCLOUD_SERVICE_ACCOUNT_CREDENTIALS
        track: internal
```

---

## Step 7B — Android patch workflow

```yaml
  patch-android-workflow:
    name: Patch Android
    instance_type: mac_mini_m2
    environment:
      <<: *shared_env
      android_signing:
        - android_keystore
    inputs:
      release_version:
        description: The release version to patch (e.g. 1.0.0+5), or "latest"
    triggering:
      events:
        - push
      branch_patterns:
        - pattern: 'patch'
          include: true
          source: true
    scripts:
      - *shorebird_install
      - *fetch_dependencies
      - name: Shorebird Patch
        script: |
          shorebird patch android \
            --release-version=${{ inputs.release_version }}
```

> **`inputs.release_version`:** Codemagic workflow inputs allow specifying the target release version at build trigger time. Use `latest` to target the most recently published release. Source: https://docs.shorebird.dev/code-push/ci/codemagic/

---

## Step 8A — iOS release workflow (skip if Android only)

```yaml
  release-ios-workflow:
    name: Release iOS
    instance_type: mac_mini_m2
    integrations:
      app_store_connect: Codemagic    # name of the App Store Connect API key in Codemagic
    environment:
      <<: *shared_env
      ios_signing:
        distribution_type: app_store
        bundle_identifier: "$BUNDLE_ID"
    scripts:
      - *shorebird_install
      - *fetch_dependencies
      - name: Fetch signing files
        script: |
          app-store-connect fetch-signing-files "$BUNDLE_ID" \
            --type IOS_APP_STORE --create \
            --issuer-id "$ISSUER_ID" \
            --key-id "$KEY_ID" \
            --private-key="$APP_STORE_CONNECT_PRIVATE_KEY"
      - name: Set up keychain
        script: keychain initialize
      - name: Add certs to keychain
        script: keychain add-certificates
      - name: Use profiles
        script: xcode-project use-profiles --custom-export-options='{"manageAppVersionAndBuildNumber":false}'
      - name: Shorebird Release iOS
        script: |
          shorebird release ios \
            --flutter-version="$FLUTTER_VERSION" \
            --export-options-plist=/Users/builder/export_options.plist
    artifacts:
      - build/ios/ipa/*.ipa
    publishing:
      app_store_connect:
        auth: integration
        submit_to_testflight: true
```

---

## Step 8B — iOS patch workflow (skip if Android only)

```yaml
  patch-ios-workflow:
    name: Patch iOS
    instance_type: mac_mini_m2
    environment:
      <<: *shared_env
      ios_signing:
        distribution_type: app_store
        bundle_identifier: "$BUNDLE_ID"
    inputs:
      release_version:
        description: The release version to patch (e.g. 1.0.0+5), or "latest"
    scripts:
      - *shorebird_install
      - *fetch_dependencies
      - name: Fetch signing files
        script: |
          app-store-connect fetch-signing-files "$BUNDLE_ID" \
            --type IOS_APP_STORE --create \
            --issuer-id "$ISSUER_ID" \
            --key-id "$KEY_ID" \
            --private-key="$APP_STORE_CONNECT_PRIVATE_KEY"
      - name: Set up keychain
        script: keychain initialize
      - name: Add certs to keychain
        script: keychain add-certificates
      - name: Use profiles
        script: xcode-project use-profiles --custom-export-options='{"manageAppVersionAndBuildNumber":false}'
      - name: Shorebird Patch iOS
        script: |
          shorebird patch ios \
            --release-version=${{ inputs.release_version }} \
            --export-options-plist=/Users/builder/export_options.plist
```

---

## Workflow overview (what runs when)

| Event | Workflow triggered | What it does |
|---|---|---|
| Push to `release` branch | `release-android-workflow` | Builds signed AAB, uploads to Shorebird, publishes to Google Play internal track |
| Push to `release` branch | `release-ios-workflow` | Builds signed IPA, uploads to Shorebird, submits to TestFlight |
| Manual trigger (with version input) | `patch-android-workflow` | Pushes OTA patch to the specified release version |
| Manual trigger (with version input) | `patch-ios-workflow` | Pushes OTA patch to the specified iOS release version |

---

## Dart/Flutter arguments passthrough

To pass `--dart-define` or other Flutter build arguments through Shorebird:

```yaml
script: |
  shorebird release android \
    --flutter-version="$FLUTTER_VERSION" \
    -- --dart-define="ENV=production"
```

Arguments after `--` are forwarded to the underlying `flutter build` call. Source: https://docs.shorebird.dev/code-push/patch/

---

## Flavor support

```yaml
script: |
  shorebird patch android \
    --flavor production \
    --target lib/main_production.dart \
    -- --dart-define="ENV=production"
```

Source: https://docs.shorebird.dev/code-push/patch/

---

## Error table

| Error | Root cause | Fix |
|---|---|---|
| Build hangs silently until timeout | `SHOREBIRD_TOKEN` not set — CLI waits for interactive login | Set `SHOREBIRD_TOKEN` as a Secure variable in group `shorebird`; reference the group in `environment.groups` |
| Build hangs on "Would you like to continue?" | `CI=true` not set — CLI prompts for confirmation | `CI=true` is exported automatically by Codemagic — if not present, add `CI: 'true'` to `environment.vars` |
| `--force` flag not recognized | `--force` was removed from Shorebird CLI | Remove `--force`; it is no longer a valid flag |
| `shorebird login:ci` outputs deprecation warning | Token generation method deprecated | Create tokens from Shorebird Console → Account → API Keys instead. Existing `login:ci` tokens work until Sep 2026. |
| `shorebird release` fails with auth error | `SHOREBIRD_TOKEN` is set but incorrect or expired | Generate a new API key from Shorebird Console; re-add as Secure variable |
| YAML anchor (`*shorebird_install`) not found | `definitions:` block is missing or misindented | Ensure `definitions:` is at the root level of `codemagic.yaml`, not nested inside a workflow |
| Android build signs with debug key | `build.gradle` not updated to use Codemagic signing env vars | Update `signingConfigs.release` to read `CM_KEYSTORE_PATH`, `CM_KEYSTORE_PASSWORD`, `CM_KEY_ALIAS`, `CM_KEY_PASSWORD` |
| iOS patch fails — "no matching release" | Release version input doesn't match a published Shorebird release | Check available releases in Shorebird Console; use `--release-version latest` for the latest |
| iOS distribution fails with Shorebird | Using Ad Hoc or Firebase distribution profile | iOS Shorebird release requires **App Store** distribution profile only |
| Patch applies but update not received by users | App was not built from a Shorebird release (e.g. built with plain `flutter build`) | A patch can only be applied to apps installed from a Shorebird release — rebuild and redistribute the release first |
| `shorebird.yaml` not found | File not committed to repo | Run `shorebird init` locally, commit `shorebird.yaml`, push |

---

## Invariants

These must always hold. Assert before marking skill complete:

1. `shorebird.yaml` is committed to the repository with a valid `app_id`
2. `SHOREBIRD_TOKEN` is stored as a Secret variable in group `shorebird` — never hardcoded in YAML
3. The `shorebird` group is referenced under `environment.groups` in every workflow that calls a Shorebird command
4. The Flutter version pinned in `codemagic.yaml` matches the version used when the original release was created
5. Android `build.gradle` reads signing config from `CM_KEYSTORE_PATH` / `CM_KEYSTORE_PASSWORD` / `CM_KEY_ALIAS` / `CM_KEY_PASSWORD` in CI
6. iOS Shorebird workflows use `app_store` distribution type — not `ad_hoc` or `development`
7. `shorebird login:ci` is NOT used for new token generation — Shorebird Console API Keys only

---

## Scope boundaries

This skill does NOT handle:

- Initial `shorebird init` on the developer's local machine (must be done before this skill runs)
- Android code signing setup beyond keystore upload reference (see Android signing skill)
- iOS code signing setup beyond what is required for Shorebird (see iOS signing skill)
- Publishing to Google Play or App Store beyond what Shorebird release workflows require
- React Native or other framework code push solutions

---

## Reference documentation

| Topic | URL |
|---|---|
| Shorebird Codemagic integration (YAML) | https://docs.shorebird.dev/code-push/ci/codemagic/ |
| Codemagic docs — Shorebird (Workflow Editor) | https://docs.codemagic.io/flutter-distributing/shorebird/ |
| Shorebird patch command reference | https://docs.shorebird.dev/code-push/patch/ |
| Shorebird Console (create API keys) | https://console.shorebird.dev |
| Codemagic blog — Shorebird + codemagic.yaml | https://blog.codemagic.io/how-to-set-up-flutter-code-push-with-shorebird-and-codemagic/ |
| Shorebird reference repo | https://github.com/shorebirdtech/codemagic_demo |
