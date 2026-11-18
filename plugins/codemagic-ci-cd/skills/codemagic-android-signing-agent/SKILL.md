---
name: codemagic-android-signing-agent
description: Agentic skill for setting up Android code signing on Codemagic. For use by AI agents autonomously configuring a Codemagic pipeline to produce signed Android release artifacts.
---

# Android Code Signing — Agentic Skill

## Purpose

Configure Android code signing in a Codemagic pipeline so that the build produces a **signed release APK or AAB**. Execute all steps autonomously. Do not pause for confirmation unless a hard blocker is encountered.

---

## Preconditions

Before executing this skill, assert all of the following:

- [ ] A `codemagic.yaml` exists at the repo root with at least one working workflow
- [ ] A successful debug build has been produced (build infrastructure is confirmed working)
- [ ] A keystore file is available (`.jks` or `.keystore`) — generate one if not

If preconditions are not met → resolve them first before proceeding.

---

## Inputs required

Collect or locate these before starting:

| Input | Description |
|---|---|
| `KEYSTORE_FILE` | Path to the `.jks` or `.keystore` file |
| `KEYSTORE_PASSWORD` | Password for the keystore |
| `KEY_ALIAS` | Alias of the signing key inside the keystore |
| `KEY_PASSWORD` | Password for the key (may differ from keystore password) |
| `KEYSTORE_REFERENCE_NAME` | A unique name to reference the keystore in `codemagic.yaml` (e.g. `my_app_release`) |

If keystore does not exist → generate one:

```shell
keytool -genkey -v -keystore codemagic.keystore -storetype JKS \
        -keyalg RSA -keysize 2048 -validity 10000 -alias codemagic \
        -storepass <KEYSTORE_PASSWORD> -keypass <KEY_PASSWORD> \
        -dname "CN=Agent, OU=CI, O=Org, L=City, S=State, C=US"
```

Store the generated file securely. It cannot be recovered if lost.

---

## Execution path

### Step 1 — Detect Gradle DSL

Inspect `android/app/build.gradle` and `android/app/build.gradle.kts`:

- `build.gradle` exists → DSL = **Groovy**
- `build.gradle.kts` exists → DSL = **Kotlin**
- Both exist → use `build.gradle.kts` (Kotlin takes precedence in modern projects)

---

### Step 2 — Patch Gradle signing config

Inject the signing config into the detected Gradle file. Use the `CI` guard so local builds are not broken.

#### Groovy (`build.gradle`)

```groovy
android {
    signingConfigs {
        release {
            if (System.getenv()["CI"]) {
                storeFile file(System.getenv()["CM_KEYSTORE_PATH"])
                storePassword System.getenv()["CM_KEYSTORE_PASSWORD"]
                keyAlias System.getenv()["CM_KEY_ALIAS"]
                keyPassword System.getenv()["CM_KEY_PASSWORD"]
            } else {
                keyAlias keystoreProperties['keyAlias']
                keyPassword keystoreProperties['keyPassword']
                storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
                storePassword keystoreProperties['storePassword']
            }
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
        }
    }
}
```

#### Kotlin DSL (`build.gradle.kts`)

```kotlin
import java.io.File
import java.util.*

val keystoreProperties = Properties().apply {
    val file = File("key.properties")
    if (file.exists()) load(file.reader())
}

android {
    signingConfigs {
        create("release") {
            if (System.getenv()["CI"].toBoolean()) {
                storeFile = file(System.getenv()["CM_KEYSTORE_PATH"])
                storePassword = System.getenv()["CM_KEYSTORE_PASSWORD"]
                keyAlias = System.getenv()["CM_KEY_ALIAS"]
                keyPassword = System.getenv()["CM_KEY_PASSWORD"]
            } else {
                storeFile = file(keystoreProperties.getProperty("storeFile") ?: "")
                storePassword = keystoreProperties.getProperty("storePassword")
                keyAlias = keystoreProperties.getProperty("keyAlias")
                keyPassword = keystoreProperties.getProperty("keyPassword")
            }
        }
    }
    buildTypes {
        getByName("release") {
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

Assert: `signingConfig signingConfigs.release` (Groovy) or `signingConfig = signingConfigs.getByName("release")` (Kotlin) is present in the release build type. If missing → inject it.

---

### Step 3 — Upload keystore to Codemagic

Use the Codemagic REST API to upload the keystore to Code Signing Identities:

```shell
curl -X POST https://api.codemagic.io/teams/<TEAM_ID>/code-signing/android-keystores \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -F "keystore=@<KEYSTORE_FILE>" \
  -F "keystorePassword=<KEYSTORE_PASSWORD>" \
  -F "keyAlias=<KEY_ALIAS>" \
  -F "keyPassword=<KEY_PASSWORD>" \
  -F "referenceName=<KEYSTORE_REFERENCE_NAME>"
```

Assert: HTTP 200/201 returned. If 401 → token invalid. If 403 → insufficient permissions (team admin required).

If API upload is not available → fall back to **Method B** (env vars, Step 3B).

---

### Step 3B — Fallback: environment variables

If API upload fails or is unavailable, store keystore as base64-encoded secret env var.

Encode keystore:

```shell
# macOS / Linux
base64 -i <KEYSTORE_FILE> -o keystore_b64.txt

# Linux alternative
openssl base64 -in <KEYSTORE_FILE> -out keystore_b64.txt
```

Add via Codemagic API:

```shell
curl -X POST https://api.codemagic.io/apps/<APP_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "CM_KEYSTORE",
    "value": "<BASE64_CONTENT>",
    "group": "keystore_credentials",
    "secure": true
  }'
```

Repeat for: `CM_KEYSTORE_PASSWORD`, `CM_KEY_ALIAS`, `CM_KEY_PASSWORD`, `CM_KEYSTORE_PATH` (value: `$CM_BUILD_DIR/codemagic.keystore`).

Add decode script to `codemagic.yaml` **before** the build step:

```yaml
- name: Set up keystore
  script: echo $CM_KEYSTORE | base64 --decode > $CM_KEYSTORE_PATH
```

---

### Step 4 — Patch `codemagic.yaml`

#### Method A (Code Signing Identities)

Add `android_signing` under `environment`:

```yaml
environment:
  android_signing:
    - <KEYSTORE_REFERENCE_NAME>
```

Update build script to use release target:

```yaml
# Flutter
- name: Build signed release AAB
  script: flutter build appbundle --release

# React Native / Native Android — APK
- name: Build signed release APK
  script: cd android && ./gradlew assembleRelease

# React Native / Native Android — AAB
- name: Build signed release AAB
  script: cd android && ./gradlew bundleRelease
```

Update artifacts to capture release output:

```yaml
artifacts:
  # AAB
  - build/**/outputs/**/*.aab        # Flutter
  - android/app/build/outputs/**/*.aab  # RN / Native

  # APK
  - build/**/outputs/**/*.apk        # Flutter
  - android/app/build/outputs/**/*.apk  # RN / Native
```

#### Method B (env vars)

Add group reference and decode script:

```yaml
environment:
  groups:
    - keystore_credentials
scripts:
  - name: Set up keystore
    script: echo $CM_KEYSTORE | base64 --decode > $CM_KEYSTORE_PATH
  # ... followed by build step
```

---

### Step 5 — Trigger build and assert

Trigger a build via API:

```shell
curl -X POST https://api.codemagic.io/builds \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "appId": "<APP_ID>",
    "workflowId": "<WORKFLOW_ID>",
    "branch": "<BRANCH>"
  }'
```

Poll build status until terminal state (`finished` or `failed`):

```shell
curl https://api.codemagic.io/builds/<BUILD_ID> \
  -H "x-auth-token: <CM_API_TOKEN>"
```

Assert on completion:
- `status == "finished"` → proceed to artifact verification
- `status == "failed"` → fetch logs, apply fix from error table below, retry once

---

### Step 6 — Verify signing

Download the artifact and verify:

```shell
# APK
apksigner verify --verbose <artifact>.apk

# AAB (requires bundletool)
bundletool validate --bundle=<artifact>.aab
```

Assert: `Verified using <key>: true` in apksigner output.

If unsigned → signing config not applied. Check Gradle `buildTypes.release.signingConfig` — it must reference the release signingConfig, not be null or omitted.

---

## Error table

| Error | Root cause | Fix |
|---|---|---|
| `Keystore file not found` | `CM_KEYSTORE_PATH` not set or decode script missing | Add Method B decode script; or verify Method A reference name matches upload |
| `UnrecoverableKeyException: wrong password` | `CM_KEY_PASSWORD` incorrect | Key password ≠ keystore password — verify both separately |
| `Keystore was tampered with` | `CM_KEYSTORE_PASSWORD` incorrect | Verify keystore password |
| `No value for storeFile` | `key.properties` not generated | Add key.properties script before build step |
| `signingConfig is null` | `buildTypes.release` missing `signingConfig` line | Inject `signingConfig signingConfigs.release` into release buildType |
| `Build type release not signed` | Gradle config present but `CI` guard not triggered | Confirm `CI=true` is set — it is always set by Codemagic; check guard syntax |
| `NullPointerException` on `.toBoolean()` | Kotlin DSL: `System.getenv()["CI"]` is null locally | Safe: `System.getenv()["CI"].toBoolean()` handles null in Kotlin — no fix needed for Codemagic builds |
| API 401 on keystore upload | Invalid `CM_API_TOKEN` | Regenerate token from Codemagic → Integrations → API |
| API 403 on keystore upload | Not team admin | Use team admin token or grant admin access |

---

## Invariants

These must always hold. Assert before marking skill complete:

1. Keystore is stored outside the repository (never committed)
2. `signingConfig signingConfigs.release` is present in `buildTypes.release`
3. The `CI` guard is present in Gradle so local builds are not broken
4. `codemagic.yaml` references the correct keystore reference name or env var group
5. Build artifact is verified as signed (`apksigner verify` passes)
6. The same keystore reference will be used for all future release builds of this app

---

## Scope boundaries

This skill does NOT handle:

- Google Play App Signing enrollment (separate process — keystore becomes upload key only)
- iOS code signing
- Publishing to Google Play (separate skill)
- Keystore rotation or migration
- Flavor-specific signing configurations (multiple keystores per build variant)

---

## Reference documentation

| Topic | URL |
|---|---|
| Signing Android apps (codemagic.yaml) | https://docs.codemagic.io/yaml-code-signing/signing-android/ |
| Signing Android apps (Flutter Workflow Editor) | https://docs.codemagic.io/flutter-code-signing/android-code-signing/ |
| Environment variables and groups | https://docs.codemagic.io/yaml-basic-configuration/configuring-environment-variables/ |
| Codemagic REST API — builds | https://docs.codemagic.io/rest-api/builds/ |
| Codemagic REST API — apps | https://docs.codemagic.io/rest-api/apps/ |
| Flutter deployment — Android signing | https://docs.flutter.dev/deployment/android#signing-the-app |
| Android apksigner | https://developer.android.com/tools/apksigner |
| Android bundletool | https://developer.android.com/tools/bundletool |
