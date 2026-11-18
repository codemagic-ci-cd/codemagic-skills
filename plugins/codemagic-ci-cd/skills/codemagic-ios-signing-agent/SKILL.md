---
name: codemagic-ios-signing-agent
description: Agentic skill for setting up iOS code signing on Codemagic. Covers Code Signing Identities (UI upload), automatic signing via CLI, and manual signing via env vars. For use by AI agents autonomously configuring a Codemagic pipeline to produce signed iOS release artifacts (IPA).
---

# iOS Code Signing — Agentic Skill

## Purpose

Configure iOS code signing in a Codemagic pipeline so that the build produces a **signed IPA** ready for TestFlight or App Store submission. Execute all steps autonomously. Do not pause for confirmation unless a hard blocker is encountered.

---

## Preconditions

Before executing this skill, assert all of the following:

- [ ] A `codemagic.yaml` exists at the repo root with at least one working workflow
- [ ] A successful debug build has been produced (build infrastructure is confirmed working)
- [ ] Apple Developer Program membership is active ($99/year)
- [ ] Team admin role in Apple Developer account
- [ ] App record exists in App Store Connect (required for App Store / TestFlight distribution)

If preconditions are not met → resolve them first before proceeding.

---

## Method Selection

Choose **one** signing method per workflow — do not mix them.

| Method | Best for | Where files live |
|---|---|---|
| **Method 1 — Code Signing Identities** | Most cases; simplest setup | Uploaded to Codemagic UI |
| **Method 2 — Automatic (CLI)** | Automated pipelines; white-label; multi-bundle | Fetched from Apple at build time |
| **Method 3 — Manual (env vars)** | No UI access; scripted setups | Stored as base64 env vars |

Default to **Method 1** unless the customer has a specific reason for Method 2 or 3.

---

## Method 1 — Code Signing Identities (UI Upload)

Docs: https://docs.codemagic.io/yaml-code-signing/signing-ios/

### Step 1.1 — Collect required files

| File | Description |
|---|---|
| `.p12` certificate | Must include the private key — export from Keychain Access with private key included |
| `.mobileprovision` profile | Must match the distribution type (App Store, Ad Hoc, etc.) and bundle ID |

### Step 1.2 — Upload to Codemagic

**Which option to use:**
- If you are assisting a human user → use **Option A (UI)**. It is simpler and does not require an API token.
- If you have a `CM_API_TOKEN` and `TEAM_ID` available and are operating autonomously → use **Option B (API)**.
- If Option B fails (401/403) → fall back to Option A and instruct the user to upload manually.

**Option A — Codemagic UI:**
1. Team settings → Code signing identities
2. Under **iOS certificates** → upload `.p12` + enter password + give it a reference name
3. Under **iOS provisioning profiles** → upload `.mobileprovision` + give it a reference name

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

### Step 1.3 — Patch `codemagic.yaml`

**Option A — by distribution type + bundle ID** (Codemagic auto-matches):
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
      - <PROFILE_REFERENCE_NAME>
    certificates:
      - <CERT_REFERENCE_NAME>
```

> Cannot mix Option A and Option B in the same workflow.

### Step 1.4 — Add signing script

```yaml
scripts:
  - name: Set up code signing settings on Xcode project
    script: xcode-project use-profiles
```

### Step 1.5 — Add artifacts

```yaml
artifacts:
  - build/ios/ipa/*.ipa       # Flutter
  - build/**/*.ipa            # broader fallback for native / RN
```

### Step 1.6 — Assert

- Build completes with `status == "finished"`
- IPA file present in artifacts
- No `exportArchive` errors in Xcode build step log

---

## Method 2 — Automatic Signing (CLI-based)

Docs: https://docs.codemagic.io/yaml-code-signing/alternative-code-signing-methods/

Codemagic fetches or creates certificates and profiles from Apple at build time using the `app-store-connect` CLI tool. No files uploaded to UI.

### Step 2.1 — Generate App Store Connect API key

1. App Store Connect → Users and Access → Integrations → App Store Connect API
2. Create key with **App Manager** role minimum
3. Download `.p8` file (one-time download only) — note Key ID and Issuer ID
4. Add to Codemagic: Team settings → Integrations → Developer Portal → Add key

### Step 2.2 — Generate certificate private key

```bash
ssh-keygen -t rsa -b 2048 -m PEM -f ~/Desktop/ios_distribution_private_key -q -N ""
```

Copy full contents of `ios_distribution_private_key` (not `.pub`) including `-----BEGIN RSA PRIVATE KEY-----` header/footer.

### Step 2.3 — Add environment variables to Codemagic

**Which option to use:**
- If you are assisting a human user → use **Option A (UI)**.
- If you have a `CM_API_TOKEN` and are operating autonomously → use **Option B (API)**.
- If Option B fails → fall back to Option A and instruct the user to add variables manually.

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

Repeat for: `APP_STORE_CONNECT_PRIVATE_KEY`, `APP_STORE_CONNECT_KEY_IDENTIFIER`, `APP_STORE_CONNECT_ISSUER_ID`.

| Variable | Value |
|---|---|
| `APP_STORE_CONNECT_PRIVATE_KEY` | Full contents of `.p8` API key file |
| `APP_STORE_CONNECT_KEY_IDENTIFIER` | Key ID (e.g. `ABC123DEF4`) |
| `APP_STORE_CONNECT_ISSUER_ID` | Issuer ID (UUID format) |
| `CERTIFICATE_PRIVATE_KEY` | RSA private key generated in Step 2.2 |

### Step 2.4 — Patch `codemagic.yaml`

```yaml
workflows:
  ios-automatic:
    integrations:
      app_store_connect: <API_KEY_NAME>   # name given in Codemagic UI
    environment:
      groups:
        - code_signing
      vars:
        BUNDLE_ID: com.example.app
    scripts:
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
```

**Profile type values for `--type`:**

| `--type` | Distribution |
|---|---|
| `IOS_APP_STORE` | App Store / TestFlight |
| `IOS_APP_ADHOC` | Ad Hoc |
| `IOS_APP_DEVELOPMENT` | Development |
| `IOS_APP_INHOUSE` | Enterprise |

**Auto-detect bundle ID:**
```bash
app-store-connect fetch-signing-files "$(xcode-project detect-bundle-id)" \
  --type IOS_APP_STORE \
  --create
```

---

## Method 3 — Manual Signing (env vars only)

Docs: https://docs.codemagic.io/yaml-code-signing/alternative-code-signing-methods/

No Codemagic UI involvement. Certificate and profile are base64-encoded and stored as env vars.

### Step 3.1 — Encode files

```bash
cat ios_distribution_certificate.p12 | base64 | pbcopy   # certificate
cat MyApp.mobileprovision | base64 | pbcopy               # profile
```

### Step 3.2 — Add environment variables to Codemagic

**Which option to use:**
- If you are assisting a human user → use **Option A (UI)**.
- If you have a `CM_API_TOKEN` and are operating autonomously → use **Option B (API)**.
- If Option B fails → fall back to Option A and instruct the user to add variables manually.

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

Repeat for: `CM_CERTIFICATE_PASSWORD`, `CM_PROVISIONING_PROFILE`.

| Variable | Value |
|---|---|
| `CM_CERTIFICATE` | Base64-encoded `.p12` file |
| `CM_CERTIFICATE_PASSWORD` | Certificate password (if set) |
| `CM_PROVISIONING_PROFILE` | Base64-encoded `.mobileprovision` file |

### Step 3.3 — Patch `codemagic.yaml`

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

---

## App extensions — all methods

Each extension target needs its own provisioning profile (Notification Service, Widget, Share Extension, etc.).

**Method 1 — Code Signing Identities:**
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

**Method 3 — Manual:**
Use naming convention `CM_PROVISIONING_PROFILE_BASE`, `CM_PROVISIONING_PROFILE_NOTIFICATIONSERVICE`, then loop:
```yaml
scripts:
  - name: Set up provisioning profiles
    script: |
      PROFILES_HOME="$HOME/Library/MobileDevice/Provisioning Profiles"
      mkdir -p "$PROFILES_HOME"
      for profile in "${!CM_PROVISIONING_PROFILE_@}"; do
        PROFILE_PATH="$(mktemp "$PROFILES_HOME"/ios_$(uuidgen).mobileprovision)"
        echo ${!profile} | base64 --decode > "$PROFILE_PATH"
      done
```

---

## Error table

| Error | Root cause | Fix |
|---|---|---|
| `No matching certificate found for provisioning profile` | Certificate not uploaded, exported without private key, or expired | Re-export `.p12` from Keychain with private key included; re-upload to Codemagic |
| `No signing certificate 'iOS Distribution' found` / `exportArchive No Accounts` | Missing `keychain add-certificates` step, or development cert used instead of distribution | Add `keychain add-certificates` step; verify certificate type is Distribution |
| `--export-options-plist` conflict | Manually passing export options plist — Codemagic generates this automatically | Remove `--export-options-plist` flag from build command |
| `Build succeeds but no IPA generated` | `CODE_SIGNING_ALLOWED=NO` in Xcode settings, or `artifacts` path wrong | Remove `CODE_SIGNING_ALLOWED=NO`; check artifacts glob matches actual output |
| `AuthenticationError: invalid API key` | Wrong `.p8` key type (APNs key used instead of App Store Connect API key) | Regenerate key in ASC → Users and Access → Integrations (not in Developer portal) |
| `You already have a current Distribution certificate` | Hit Apple's 3-certificate limit | Revoke an old cert in Apple Developer, or export existing cert as `.p12` instead of generating new |
| Profile becomes invalid after capability change | Apple invalidates profile when capabilities change | Regenerate profile via `--create` (Method 2) or re-upload manually (Method 1) |
| `.p8 file picker doesn't open` on iPadOS Safari | `accept=".p8"` has no MIME type registered in Safari | Use a desktop browser instead |
| `API 401` on upload | Invalid `CM_API_TOKEN` | Regenerate token from Codemagic → Integrations → API |
| `API 403` on upload | Not team admin | Use team admin token or grant admin access |

---

## Invariants

These must always hold. Assert before marking skill complete:

1. Certificate and profile are NOT committed to the repository
2. Certificate type matches profile type (Distribution cert with App Store profile, etc.)
3. Every app extension target has a matching provisioning profile
4. `xcode-project use-profiles` is present in the build scripts
5. `artifacts` section captures the IPA output path
6. `--export-options-plist` is NOT manually passed in the build command
7. `CODE_SIGNING_ALLOWED=NO` is NOT present in Xcode build settings

---

## Scope boundaries

This skill does NOT handle:

- Android code signing (separate skill)
- Publishing to App Store Connect or TestFlight (separate skill)
- App record creation in App Store Connect (must be done manually first)
- Push notification certificates (APNs — separate from signing)
- Enterprise distribution provisioning
- Xcode project configuration (bundle ID changes, capability enabling)

---

## Reference documentation

| Topic | URL |
|---|---|
| Code Signing Identities (Method 1) | https://docs.codemagic.io/yaml-code-signing/signing-ios/ |
| Alternative signing methods (Methods 2 & 3) | https://docs.codemagic.io/yaml-code-signing/alternative-code-signing-methods/ |
| Environment variables and groups | https://docs.codemagic.io/yaml-basic-configuration/configuring-environment-variables/ |
| Codemagic REST API — builds | https://docs.codemagic.io/rest-api/builds/ |
| App Store Connect API | https://developer.apple.com/documentation/appstoreconnectapi |
| CLI tools — app-store-connect | https://github.com/codemagic-ci-cd/cli-tools/blob/master/docs/app-store-connect/ |
| Sample projects | https://github.com/codemagic-ci-cd/codemagic-sample-projects |
