---
name: android-code-signing
description: Deep-dive reference for Android code signing on Codemagic — Code Signing Identities (UI upload), alternative manual signing via env vars, Gradle signing config, multiple keystores, common errors from real support tickets, and debugging checklist.
---

# Android Code Signing on Codemagic

## What You Need Before You Start

| Requirement | Notes |
|---|---|
| A `.jks` or `.keystore` file | Generated with `keytool` or exported from Android Studio |
| Keystore password | Set when the keystore was created |
| Key alias | The alias used when generating the key |
| Key password | May differ from the keystore password |

> **Critical:** The keystore cannot be downloaded from Codemagic after upload. Always keep an independent backup. Every Google Play release must use the **same keystore** — if it's lost, you cannot update the app on Google Play without going through account recovery.

---

## Two Signing Methods

Codemagic supports two approaches to Android signing. Choose one per workflow — do not mix them.

| Method | Where files live |
|---|---|
| **1. Code Signing Identities** | Uploaded to Codemagic UI |
| **2. Alternative Manual** | Stored as base64 env var |

---

## Method 1: Code Signing Identities (Codemagic UI Upload)

Docs: https://docs.codemagic.io/yaml-code-signing/signing-android/

Upload your keystore once to Codemagic's **Code Signing Identities** section. Codemagic sets default environment variables automatically and makes the keystore file available on the build machine.

### Upload the keystore

1. Go to **Team settings → Code signing identities**
2. Under **Android keystores** → upload your `.jks`/`.keystore` file
3. Enter keystore password, key alias, key password
4. Give it a reference name

### Reference in codemagic.yaml

```yaml
environment:
  android_signing:
    - MyKeystore   # reference name from Code Signing Identities
```

### Default environment variables set by Codemagic

Once referenced in `android_signing`, these variables are automatically available in all build steps:

| Variable | Value |
|---|---|
| `CM_KEYSTORE_PATH` | Absolute path to the keystore file on disk |
| `CM_KEYSTORE_PASSWORD` | Keystore password |
| `CM_KEY_ALIAS` | Key alias |
| `CM_KEY_PASSWORD` | Key password |

### Gradle signing config

Use these env vars in your Gradle signing config. The `CI` check prevents the config from breaking local builds where the variables aren't set.

**build.gradle (Groovy):**
```groovy
android {
    signingConfigs {
        release {
            if (System.getenv()["CI"]) {
                storeFile file(System.getenv()["CM_KEYSTORE_PATH"])
                storePassword System.getenv()["CM_KEYSTORE_PASSWORD"]
                keyAlias System.getenv()["CM_KEY_ALIAS"]
                keyPassword System.getenv()["CM_KEY_PASSWORD"]
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

**build.gradle.kts (Kotlin DSL):**
```kotlin
android {
    signingConfigs {
        create("release") {
            if (System.getenv("CI") == "true") {
                storeFile = file(System.getenv("CM_KEYSTORE_PATH"))
                storePassword = System.getenv("CM_KEYSTORE_PASSWORD")
                keyAlias = System.getenv("CM_KEY_ALIAS")
                keyPassword = System.getenv("CM_KEY_PASSWORD")
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

### Multiple keystores (e.g. per flavor)

When using multiple keystores, the default `CM_KEYSTORE_PATH` etc. variables are ambiguous — you must explicitly name the env vars for each keystore:

```yaml
environment:
  android_signing:
    - keystore: ProductionKeystore
      keystore_environment_variable: PROD_KEYSTORE_PATH
      keystore_password_environment_variable: PROD_KEYSTORE_PASSWORD
      key_alias_environment_variable: PROD_KEY_ALIAS
      key_password_environment_variable: PROD_KEY_PASSWORD
    - keystore: StagingKeystore
      keystore_environment_variable: STAGING_KEYSTORE_PATH
      keystore_password_environment_variable: STAGING_KEYSTORE_PASSWORD
      key_alias_environment_variable: STAGING_KEY_ALIAS
      key_password_environment_variable: STAGING_KEY_PASSWORD
```

Reference these custom variable names in your Gradle signing config instead of the defaults.

---

## Method 2: Alternative Manual (base64 env var)

Docs: https://docs.codemagic.io/yaml-code-signing/alternative-code-signing-methods/

No Codemagic UI involvement. The keystore is base64-encoded and stored as an environment variable. Decoded and installed in the build script.

### Encode your keystore

**macOS:**
```bash
cat your-release-key.keystore | base64 | pbcopy
```

### Required environment variables

Store in a group (e.g. `keystore_credentials`). See [how to configure environment variable groups](https://docs.codemagic.io/yaml-basic-configuration/configuring-environment-variables/):

| Variable | Value |
|---|---|
| `CM_KEYSTORE` | Base64-encoded keystore file |
| `CM_KEYSTORE_PASSWORD` | Keystore password |
| `CM_KEY_ALIAS` | Key alias |
| `CM_KEY_PASSWORD` | Key password |

### codemagic.yaml scripts

```yaml
environment:
  groups:
    - keystore_credentials
  vars:
    CM_KEYSTORE_PATH: /tmp/keystore.keystore

scripts:
  - name: Set up keystore
    script: |
      echo $CM_KEYSTORE | base64 --decode > $CM_KEYSTORE_PATH
```

Then use the same Gradle signing config as Method 1 (it reads from the same env var names).

---

## Common Errors and Fixes

### "Unrecognized keystore format"

The keystore file is invalid, corrupted, or in an unsupported format.

| Cause | Fix |
|---|---|
| Keystore was already in use with an existing app and has an unexpected format | Verify with `keytool -list -keystore your.keystore` — if it errors locally, the file is corrupt |
| Wrong file uploaded (e.g. `.p12` instead of `.jks`) | Re-export from Android Studio or regenerate with `keytool` |
| Base64 encoding stripped or corrupted (Method 2) | Re-encode: `cat your.keystore \| base64` — ensure no line breaks were introduced |

---

### Build signed but APK/AAB not found

1. Check that the `artifacts` path matches actual output:
   ```yaml
   artifacts:
     - app/build/outputs/**/*.aab   # AAB
     - app/build/outputs/**/*.apk   # APK
   ```
2. Signing config not applied to `release` build type — confirm `signingConfig signingConfigs.release` is set under `buildTypes { release { ... } }`

---

### Gradle/Java version incompatibility

Seen in real tickets: Gradle 8.2 cannot process bytecode compiled with Java 21 (e.g. `bcprov` library version 1.79).

```
Execution failed for task ':app:minifyReleaseWithR8'.
Unsupported class file major version 65
```

**Fix options:**
1. Upgrade Gradle wrapper to 8.5+ (supports Java 21 bytecode)
2. Downgrade the affected library to a version compiled with Java 17 or lower

---

### `variables.gradle` not found (Capacitor / React Native projects)

Capacitor-generated Android projects reference a `variables.gradle` file in the root `android/` directory. If it's missing from the repo, the build fails with a file not found error.

**Fix:** Add the `variables.gradle` file to the repo. It defines version variables used across Gradle files:
```groovy
ext {
    minSdkVersion = 22
    compileSdkVersion = 34
    targetSdkVersion = 34
    // ... other vars
}
```

---

### "I lost my keystore — can Codemagic give it back?"

No. Keystores are encrypted at rest and cannot be retrieved or downloaded from Codemagic, even by support staff.

**If the app is already enrolled in Google Play's App Signing:**
Request a signing key reset directly from Google:
https://support.google.com/googleplay/android-developer/contact/key

Once approved, the App Signing page in Play Console is updated — the old key is removed and a new signing configuration can be set up.

**If the app has never been published to Google Play:**
The keystore is effectively gone. Generate a new one and use it going forward. The app on Play Store (if published) will need the key reset process above first.

> Always keep an independent backup of the keystore file — Codemagic explicitly warns that it cannot be downloaded after upload.

---

### Keystore not referenced — APK is unsigned

If the `android_signing` block is missing from `codemagic.yaml`, or the keystore was not uploaded, the build produces a debug-signed or unsigned APK that cannot be uploaded to Google Play.

**Fix:** Upload the keystore to Code Signing Identities and add the `android_signing` block to your workflow's `environment` section.

---

## Debugging Checklist

1. **What is the exact error?** — Gradle error, Xcode-style message, or upload rejection from Google Play?
2. **Which signing method?** — Code Signing Identities or manual base64?
3. **Is `signingConfig` set on `release` build type?** — Not just defined in `signingConfigs`, but also assigned under `buildTypes`
4. **Is the `CI` guard present in Gradle?** — Without it, local builds may fail and the CI variable check may behave unexpectedly
5. **Is the keystore valid?** — Run `keytool -list -keystore your.keystore` locally to confirm
6. **Artifacts path** — Does it match the actual output location for APK/AAB?
7. **Multiple keystores** — Are all referenced by the correct names in `android_signing`?
8. **Capacitor/RN project** — Is `variables.gradle` committed to the repo?
9. **Gradle + dependency Java compatibility** — Are any dependencies compiled with Java 21 bytecode while using Gradle < 8.5?

---

## Key Documentation Links

- Code Signing Identities (Method 1): https://docs.codemagic.io/yaml-code-signing/signing-android/
- Alternative signing (Method 2): https://docs.codemagic.io/yaml-code-signing/alternative-code-signing-methods/
- Environment variable groups: https://docs.codemagic.io/yaml-basic-configuration/configuring-environment-variables/
- Sample projects: https://github.com/codemagic-ci-cd/codemagic-sample-projects
