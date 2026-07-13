---
name: ios-code-signing
description: Deep-dive reference for iOS code signing on Codemagic — code signing identities (UI upload), automatic signing via CLI, manual signing via env vars, App Store Connect API keys, certificates, provisioning profiles, TestFlight distribution, and every common error seen in real support tickets.
---

# iOS Code Signing on Codemagic

## What You Need Before You Start

| Requirements                                     |  Where to get it             |
|--------------------------------------------------|------------------------------|
| Apple Developer Program membership ($99/year)    | developer.apple.com          |
| Team admin role in Apple Developer               | Apple Developer → People     |
| App record created in App Store Connect          | App Store Connect → Apps → + |

---

## Three Signing Methods

Codemagic supports three distinct approaches to iOS code signing. Choose one per workflow — do not mix them.

| Method | Where files live |
|---|---|
| **1. Code Signing Identities** | Uploaded to Codemagic UI |
| **2. Automatic (alternative)** | Fetched from Apple at build time via CLI |
| **3. Manual (alternative)** | Stored as base64 env vars |
 
---

## Method 1: Code Signing Identities (Codemagic UI Upload)

Docs: https://docs.codemagic.io/yaml-code-signing/signing-ios/

You upload your `.p12` certificate and `.mobileprovision` profile once to Codemagic's **Code Signing Identities** section, then reference them by name in YAML. Codemagic handles the keychain setup automatically — no manual keychain scripts needed.

### Upload files

**Which option to use:**
- If assisting a human user → use **Option A (UI)**. Simpler, no API token required.
- If operating autonomously with a `CM_API_TOKEN` available → use **Option B (API)**.
- If Option B fails (401/403) → fall back to Option A and instruct the user to upload manually.

**Option A — Codemagic UI:**
1. Go to **Team settings → Code signing identities**
2. Under **iOS certificates** → upload your `.p12` (must include the private key) + enter the password + give it a reference name
3. Under **iOS provisioning profiles** → upload your `.mobileprovision` + give it a reference name

Certificates can also be **generated** (Codemagic creates a new one in Apple Developer using your API key) or **fetched** (pull an existing certificate from Apple Developer that Codemagic previously created).

**Option B — API:**
```shell
# Upload certificate
curl -X POST https://api.codemagic.io/teams/<TEAM_ID>/code-signing/ios-certificates \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -F "certificate=@<CERT.p12>" \
  -F "password=<CERT_PASSWORD>" \
  -F "referenceName=<CERT_REFERENCE_NAME>"

# Upload provisioning profile
curl -X POST https://api.codemagic.io/teams/<TEAM_ID>/code-signing/provisioning-profiles \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -F "provisioningProfile=@<PROFILE.mobileprovision>" \
  -F "referenceName=<PROFILE_REFERENCE_NAME>"
```

Assert: HTTP 200/201 returned. If 401 → token invalid. If 403 → not team admin.

### Reference in codemagic.yaml

**Option A — by distribution type + bundle ID** (Codemagic auto-matches the right uploaded file):
```yaml
environment:
  ios_signing:
    distribution_type: app_store   # app_store | ad_hoc | development | enterprise
    bundle_identifier: com.example.app
```

**Option B — by specific reference name** (explicit control):
```yaml
environment:
  ios_signing:
    provisioning_profiles:
      - MyAppStore_Profile
    certificates:
      - MyDistribution_Cert
```

**Option B with environment variable** (exposes file path on disk):
```yaml
environment:
  ios_signing:
    provisioning_profiles:
      - profile: MyAppStore_Profile
        environment_variable: PROFILE_PATH
    certificates:
      - certificate: MyDistribution_Cert
        environment_variable: CERT_PATH
```

> **Cannot mix Option A and Option B** in the same workflow — use one or the other.

### Build scripts

With Code Signing Identities, Codemagic sets up the keychain automatically. You only need one script step:

```yaml
scripts:
  - name: Set up code signing settings on Xcode project
    script: xcode-project use-profiles
```

For TestFlight **internal testing only** (bypasses external review, no beta submission required):
```yaml
scripts:
  - name: Set up code signing settings on Xcode project
    script: xcode-project use-profiles --custom-export-options='{"testFlightInternalTestingOnly": true}'
```

### App extensions need separate profiles

Each extension target (Notification Service, Widget, Share Extension, etc.) needs its own profile:

```yaml
environment:
  ios_signing:
    provisioning_profiles:
      - MainApp_Profile            # com.example.app
      - NotificationExt_Profile    # com.example.app.NotificationService
      - Widget_Profile             # com.example.app.Widget
    certificates:
      - MyDistribution_Cert
```

### File locations on the build machine
- Provisioning profiles: `~/Library/MobileDevice/Provisioning Profiles`
- Certificates: `~/Library/MobileDevice/Certificates`

---

## Method 2: Automatic Code Signing (Alternative — CLI-based)

Docs: https://docs.codemagic.io/yaml-code-signing/alternative-code-signing-methods/

No files uploaded to Codemagic UI. Codemagic fetches or creates certificates and profiles from Apple at build time using the `app-store-connect` CLI tool. Requires an App Store Connect API key and a `CERTIFICATE_PRIVATE_KEY`.

### Required environment variables

**Which option to use:**
- If assisting a human user → use **Option A (UI)**. Simpler, no API token required.
- If operating autonomously with a `CM_API_TOKEN` available → use **Option B (API)**.
- If Option B fails → fall back to Option A and instruct the user to add variables manually.

| Variable | Value |
|---|---|
| `APP_STORE_CONNECT_PRIVATE_KEY` | Full contents of your `.p8` API key file |
| `APP_STORE_CONNECT_KEY_IDENTIFIER` | Key ID (e.g. `ABC123DEF4`) |
| `APP_STORE_CONNECT_ISSUER_ID` | Issuer ID (UUID format) |
| `CERTIFICATE_PRIVATE_KEY` | RSA private key (see generation below) |

**Option A — Codemagic UI:**
- App-level: App settings → Environment variables → add variables as secure, assign to group `code_signing`
- Team-level (shared across apps): Team settings → Global environment variables → same steps

**Option B — API:**
```shell
curl -X POST https://api.codemagic.io/apps/<APP_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "CERTIFICATE_PRIVATE_KEY",
    "value": "<RSA_PRIVATE_KEY_CONTENTS>",
    "group": "code_signing",
    "secure": true
  }'
```
Repeat for `APP_STORE_CONNECT_PRIVATE_KEY`, `APP_STORE_CONNECT_KEY_IDENTIFIER`, `APP_STORE_CONNECT_ISSUER_ID`.

See [how to configure environment variable groups](https://docs.codemagic.io/yaml-basic-configuration/configuring-environment-variables/).

### Generate the certificate private key

**New key:**
```bash
ssh-keygen -t rsa -b 2048 -m PEM -f ~/Desktop/ios_distribution_private_key -q -N ""
```
Copy the contents of `ios_distribution_private_key` (not `.pub`) including the `-----BEGIN RSA PRIVATE KEY-----` header/footer into `CERTIFICATE_PRIVATE_KEY`.

**Or export from an existing certificate in Keychain:**
```bash
openssl pkcs12 -in IOS_DISTRIBUTION.p12 -nodes -nocerts | openssl rsa -out ios_distribution_private_key
```

### codemagic.yaml

```yaml
workflows:
  ios-automatic:
    name: iOS Automatic Signing
    instance_type: mac_mini_m2
    integrations:
      app_store_connect: MyASCIntegration   # API key configured in Team settings
    environment:
      groups:
        - code_signing    # contains CERTIFICATE_PRIVATE_KEY + ASC vars
      vars:
        BUNDLE_ID: com.example.app
      flutter: stable
      xcode: latest
    scripts:
      - name: Get Flutter packages
        script: flutter pub get
      - name: Install CocoaPods
        script: cd ios && pod install
      - name: Set up keychain
        script: keychain initialize
      - name: Fetch signing files from Apple
        script: |
          app-store-connect fetch-signing-files "$BUNDLE_ID" \
            --type IOS_APP_STORE \
            --create
      - name: Add certificates to keychain
        script: keychain add-certificates
      - name: Set up code signing settings on Xcode project
        script: xcode-project use-profiles
      - name: Build IPA
        script: flutter build ipa --release
    artifacts:
      - build/ios/ipa/*.ipa
    publishing:
      app_store_connect:
        auth: integration
        submit_to_testflight: true
```

**Profile type values for `--type`:**

| `--type` | Distribution |
|---|---|
| `IOS_APP_STORE` | App Store / TestFlight |
| `IOS_APP_ADHOC` | Ad Hoc (Firebase App Distribution etc.) |
| `IOS_APP_DEVELOPMENT` | Development |
| `IOS_APP_INHOUSE` | Enterprise |

**Auto-detect bundle ID** instead of hardcoding:
```bash
app-store-connect fetch-signing-files "$(xcode-project detect-bundle-id)" \
  --type IOS_APP_STORE \
  --create
```

---

## Method 3: Manual Code Signing (Alternative — env vars only)

Docs: https://docs.codemagic.io/yaml-code-signing/alternative-code-signing-methods/

No Codemagic UI involvement. Certificate and provisioning profile are base64-encoded and stored as environment variables. Decoded and installed in build scripts.

### Encode your files

**macOS:**
```bash
cat ios_distribution_certificate.p12 | base64 | pbcopy   # certificate
cat MyApp.mobileprovision | base64 | pbcopy               # profile
```

### Required environment variables

**Which option to use:**
- If assisting a human user → use **Option A (UI)**. Simpler, no API token required.
- If operating autonomously with a `CM_API_TOKEN` available → use **Option B (API)**.
- If Option B fails → fall back to Option A and instruct the user to add variables manually.

| Variable | Value |
|---|---|
| `CM_CERTIFICATE` | Base64-encoded `.p12` file |
| `CM_CERTIFICATE_PASSWORD` | Certificate password (if set) |
| `CM_PROVISIONING_PROFILE` | Base64-encoded `.mobileprovision` file |

**Option A — Codemagic UI:**
- App-level: App settings → Environment variables → add variables as secure, assign to group `manual_signing`
- Team-level (shared across apps): Team settings → Global environment variables → same steps

**Option B — API:**
```shell
curl -X POST https://api.codemagic.io/apps/<APP_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "CM_CERTIFICATE",
    "value": "<BASE64_P12_CONTENT>",
    "group": "manual_signing",
    "secure": true
  }'
```
Repeat for `CM_CERTIFICATE_PASSWORD` and `CM_PROVISIONING_PROFILE`.

See [how to configure environment variable groups](https://docs.codemagic.io/yaml-basic-configuration/configuring-environment-variables/).

### codemagic.yaml scripts

```yaml
environment:
  groups:
    - manual_signing

scripts:
  - name: Set up keychain
    script: keychain initialize
  - name: Set up provisioning profile
    script: |
      PROFILES_HOME="$HOME/Library/MobileDevice/Provisioning Profiles"
      mkdir -p "$PROFILES_HOME"
      PROFILE_PATH="$(mktemp "$PROFILES_HOME"/$(uuidgen).mobileprovision)"
      echo ${CM_PROVISIONING_PROFILE} | base64 --decode > "$PROFILE_PATH"
      echo "Saved provisioning profile $PROFILE_PATH"
  - name: Set up signing certificate
    script: |
      echo $CM_CERTIFICATE | base64 --decode > /tmp/certificate.p12
      if [ -z ${CM_CERTIFICATE_PASSWORD+x} ]; then
        keychain add-certificates --certificate /tmp/certificate.p12
      else
        keychain add-certificates --certificate /tmp/certificate.p12 \
          --certificate-password $CM_CERTIFICATE_PASSWORD
      fi
  - name: Set up code signing settings on Xcode project
    script: xcode-project use-profiles
```

### Multiple provisioning profiles (app extensions)

Use a naming convention like `CM_PROVISIONING_PROFILE_BASE`, `CM_PROVISIONING_PROFILE_NOTIFICATIONSERVICE`, then loop:

```yaml
scripts:
  - name: Set up provisioning profiles
    script: |
      PROFILES_HOME="$HOME/Library/MobileDevice/Provisioning Profiles"
      mkdir -p "$PROFILES_HOME"
      for profile in "${!CM_PROVISIONING_PROFILE_@}"; do
        PROFILE_PATH="$(mktemp "$PROFILES_HOME"/ios_$(uuidgen).mobileprovision)"
        echo ${!profile} | base64 --decode > "$PROFILE_PATH"
        echo "Saved provisioning profile $PROFILE_PATH"
      done
```

---

## App Store Connect API Key

Required for: Method 2 (automatic signing), publishing to TestFlight/App Store, and generating/fetching certificates via Codemagic UI.

### Setup in Team settings (recommended)
1. ASC → Users and Access → Integrations → App Store Connect API → create key with **App Manager** role
2. Download `.p8` (one-time download only), note Key ID and Issuer ID
3. Codemagic → Team settings → Integrations → Developer Portal → Add key

### Setup as environment variables
```yaml
publishing:
  app_store_connect:
    api_key: $APP_STORE_CONNECT_PRIVATE_KEY
    key_id: $APP_STORE_CONNECT_KEY_IDENTIFIER
    issuer_id: $APP_STORE_CONNECT_ISSUER_ID
```

### .p8 key type confusion — #1 cause of "invalid key" errors

Three different Apple services use `.p8` files. They look identical but are not interchangeable:

| Key type | Purpose | Where created |
|---|---|---|
| **App Store Connect API** | CI/CD, certificate/profile automation | ASC → Users and Access → Integrations |
| **APNs** | Push notifications | Developer → Certificates, IDs & Profiles → Keys |
| **DeviceCheck** | Fraud detection | Developer → Certificates, IDs & Profiles → Keys |

Codemagic requires the **App Store Connect API** key. If a customer says the key "doesn't work" or authentication fails, always ask which type of key they're using and where they created it.

---

## Provisioning Profile Management

### Profile types

| Type | Used for |
|---|---|
| Development | Testing on registered devices |
| Ad Hoc | Distributing to up to 100 registered devices |
| App Store | TestFlight and App Store submission |
| Enterprise | In-house distribution (Enterprise Program only) |

### When profiles become invalid

A profile must be regenerated in Apple Developer (and re-uploaded to Codemagic) when:
- The associated certificate expires or is revoked
- A capability is added or removed (Push Notifications, Sign in with Apple, etc.)
- The bundle ID changes
- Device UDIDs change (Ad Hoc profiles)

**Update workflow:** Delete the old profile in Codemagic → re-upload from scratch. In-place editing of profiles in Codemagic does not reliably sync the updated version. (Confirmed in ticket #17694.)

### Adding capabilities to a bundle ID via CLI

Instead of adding capabilities manually in the Apple Developer portal, the `app-store-connect` CLI tool can enable them programmatically. This is the automated equivalent of going to Apple Developer → App IDs → your app → Edit capabilities.

```bash
app-store-connect bundle-ids enable-capabilities \
  --bundle-id-resource-id YOUR_BUNDLE_ID_RESOURCE_ID \
  PUSH_NOTIFICATIONS \
  SIGN_IN_WITH_APPLE
```

Full usage reference: https://github.com/codemagic-ci-cd/cli-tools/blob/master/docs/app-store-connect/bundle-ids/enable-capabilities.md#usage

> **Important:** After enabling capabilities via the CLI, the provisioning profile linked to that bundle ID becomes invalid (Apple invalidates it automatically when capabilities change). You must regenerate the profile — either via `app-store-connect fetch-signing-files --create` (Method 2) or manually in Apple Developer and re-upload to Codemagic (Method 1). For **Code Signing Identities** (Method 1), the updated profile must be explicitly re-uploaded in the Codemagic UI — the CLI change alone is not enough.

### Certificate limit

Apple allows a maximum of **3 Distribution certificates** per team. Exceeding this gives:
```
You already have a current Distribution certificate or a pending certificate request.
```
**Fix:** Revoke an old certificate in Apple Developer → Certificates, or export an existing one as `.p12` and upload it directly instead of generating a new one.

---

## Common Errors and Fixes

### "Cannot save Signing Certificates without certificate private key"

**ONLY applies to this exact error message** — do not apply to "No matching profiles found" or any other signing error.

**Root cause:** `CERTIFICATE_PRIVATE_KEY` environment variable is missing. Codemagic needs an RSA private key to generate a CSR and create the distribution certificate via the App Store Connect API. This is unrelated to keychain setup, stale certs in Apple Developer Portal, or Codemagic Code Signing Identities.

**Fix:** Generate the key on any machine (no Mac required):
```bash
ssh-keygen -t rsa -b 2048 -m PEM -f ~/Desktop/ios_distribution_private_key -q -N ""
```
Copy the full contents of `ios_distribution_private_key` (not `.pub`) — including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` — and add it as the `CERTIFICATE_PRIVATE_KEY` environment variable in Codemagic.

---

### "No matching profiles found for bundle identifier X and distribution type app_store"

**ONLY applies to this exact error** — do not suggest `CERTIFICATE_PRIVATE_KEY` for this error.

**Root cause:** The App ID (bundle ID) has not been registered in Apple Developer Portal. Codemagic automatic signing can create certificates and provisioning profiles, but it **cannot create App IDs**. The bundle ID must exist under Identifiers before automatic signing can bootstrap.

**Fix:**
1. Go to [developer.apple.com](https://developer.apple.com) → Certificates, Identifiers & Profiles → Identifiers
2. Register a new App ID with the exact bundle ID shown in the error
3. Re-run the build — automatic signing will now be able to create the certificate and profile

---

### "No matching certificate found for provisioning profile"

The provisioning profile references a certificate Codemagic can't find on the build machine.

| Cause | Fix |
|---|---|
| Certificate not uploaded to Codemagic | Upload the `.p12` that was used to create the profile |
| Certificate exported without private key | Re-export from Keychain Access — right-click cert → Export → include private key |
| Wrong certificate type | Profile is App Store but certificate is Development; regenerate profile with correct cert |
| Certificate expired | Create new cert in Apple Developer, regenerate profile, re-upload both to Codemagic |
| Profile out of sync after Apple Developer update | Delete profile in Codemagic, re-add from scratch |

---

### "No signing certificate 'iOS Distribution' found" / `exportArchive No Accounts`

Xcode can't find a distribution certificate during IPA export.

**Causes:**
- Missing `keychain add-certificates` step (Methods 2 and 3)
- Development certificate uploaded instead of Distribution
- `--export-options-plist` manually passed in the build command — Codemagic already generates this file automatically; passing it again causes a conflict

```yaml
# WRONG — conflicts with Codemagic's auto-generated export options
script: flutter build ipa --export-options-plist=ios/ExportOptions.plist

# CORRECT
script: flutter build ipa --release
```

---

### "The .p8 file picker doesn't open" (iPadOS Safari)

The `accept=".p8"` attribute on the file input has no registered MIME type in Safari on iPadOS, causing the picker to silently not open.

**Fix:** Use a desktop browser. Known platform bug, tracked internally. (Confirmed in ticket #17446 — resolved after switching to desktop browser.)

---

### Build succeeds but no IPA generated

1. `CODE_SIGNING_ALLOWED=NO` in Xcode build settings — disables signing entirely, no IPA produced. Remove it.
2. Artifacts path doesn't match actual output:
   ```yaml
   artifacts:
     - build/ios/ipa/*.ipa       # Flutter
     - build/**/*.ipa             # broader fallback
   ```
3. Archive succeeded but export failed — look for `exportArchive` errors in the Xcode step log

---

## Debugging Checklist

1. **What is the exact error message?** — check the Xcode build step log
2. **Which signing method is being used?** — Code Signing Identities, automatic, or manual? Are they mixing?
3. **Certificate type** — Development vs Distribution; does it match the profile type?
4. **API key type** — App Store Connect API key (created in ASC) vs APNs key (created in Developer portal)?
5. **Profile validity** — was a capability added recently? Was the profile regenerated in Apple Developer but not re-uploaded?
6. **`--export-options-plist` flag** — manually specified? Remove it.
7. **`CODE_SIGNING_ALLOWED=NO`** — in Xcode project or build script?
8. **Certificate count** — hit the 3-cert limit?
9. **App extensions** — does every extension target have a matching profile?

---

## Key Documentation Links

- Code signing identities (Method 1): https://docs.codemagic.io/yaml-code-signing/signing-ios/
- Alternative signing methods (Methods 2 & 3): https://docs.codemagic.io/yaml-code-signing/alternative-code-signing-methods/
- CLI tools — enable capabilities: https://github.com/codemagic-ci-cd/cli-tools/blob/master/docs/app-store-connect/bundle-ids/enable-capabilities.md#usage
- Sample projects: https://github.com/codemagic-ci-cd/codemagic-sample-projects
