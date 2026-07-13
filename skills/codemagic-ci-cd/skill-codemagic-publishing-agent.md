---
name: codemagic-publishing-agent
description: Agentic skill for configuring app publishing on Codemagic. Covers App Store Connect, TestFlight, Google Play, Firebase App Distribution, and Huawei AppGallery. For use by AI agents autonomously configuring a Codemagic pipeline to publish release artifacts.
---

# App Publishing — Agentic Skill

## Purpose

Configure app publishing in a Codemagic pipeline so that a successfully built and signed artifact is automatically published to the target distribution platform. Execute all steps autonomously. Do not pause for confirmation unless a hard blocker is encountered.

---

## Preconditions

Before executing this skill, assert all of the following:

- [ ] A `codemagic.yaml` exists at the repo root with at least one working workflow
- [ ] A signed release artifact is produced by the workflow (APK, AAB, or IPA)
- [ ] The target app exists in the destination platform (App Store Connect, Google Play Console, Firebase, etc.)
- [ ] Required credentials are available (API keys, service accounts, tokens)

If preconditions are not met → resolve them first before proceeding.

---

## Platform Detection

Identify the target platform(s) from the user's request and execute the corresponding section(s) below. Multiple platforms can be configured in the same workflow.

| Platform | Artifact | Section |
|---|---|---|
| App Store Connect (production) | IPA | Step A |
| TestFlight (beta) | IPA | Step A |
| Google Play | AAB / APK | Step B |
| Firebase App Distribution | APK / AAB / IPA | Step C |
| Huawei AppGallery | APK / AAB | Step D |

---

## Step A — App Store Connect / TestFlight

### A.1 — Precondition

The app record must already exist in App Store Connect before the first publish. Create it manually if it does not exist.

### A.2 — Generate App Store Connect API key

Navigate to App Store Connect → Users and Access → Integrations → App Store Connect API → Generate API key with **App Manager** role minimum. Download the `.p8` file — it can only be downloaded once.

### A.3 — Add API key to Codemagic UI

Codemagic UI → Teams → Integrations → App Store Connect → Add API key. Provide the Key ID, Issuer ID, and upload the `.p8` file.

### A.4 — Patch `codemagic.yaml`

```yaml
workflows:
  ios-workflow:
    integrations:
      app_store_connect: <API key name>   # name given in Codemagic UI

    publishing:
      app_store_connect:
        auth: integration
        submit_to_testflight: true              # distribute to TestFlight
        expire_build_submitted_for_review: true # auto-expire previous builds
        beta_groups:                            # TestFlight tester groups — remove if not needed
          - internal_testers
        submit_to_app_store: false             # set true to submit for App Store review
        cancel_previous_submissions: true      # cancel any pending App Store submissions
        release_type: AFTER_APPROVAL           # AFTER_APPROVAL | MANUAL | SCHEDULED
        phased_release: true                   # stagger rollout over 7 days (production only)
        copyright: "2024 Company Name"         # optional
```

### A.5 — TestFlight vs App Store

| Goal | Configuration |
|---|---|
| TestFlight — internal testers only | `submit_to_testflight: true`, `submit_to_app_store: false` |
| TestFlight — external tester groups | `submit_to_testflight: true` + `beta_groups` list |
| App Store — release after review | `submit_to_app_store: true`, `release_type: AFTER_APPROVAL` |
| App Store — manual release | `submit_to_app_store: true`, `release_type: MANUAL` |
| App Store — scheduled release | `submit_to_app_store: true`, `release_type: SCHEDULED`, `earliest_release_date: 2024-12-01T14:00:00+00:00` |
| Both TestFlight and App Store | `submit_to_testflight: true`, `submit_to_app_store: true` |

### A.6 — White-label apps

For white-label builds where different App Store Connect credentials are required per client, use environment variables instead of the UI integration. Add credentials as secure env vars per group:

```yaml
environment:
  groups:
    - app_store_credentials   # contains APP_STORE_CONNECT_PRIVATE_KEY,
                              # APP_STORE_CONNECT_KEY_IDENTIFIER,
                              # APP_STORE_CONNECT_ISSUER_ID

publishing:
  app_store_connect:
    api_key: $APP_STORE_CONNECT_PRIVATE_KEY
    key_id: $APP_STORE_CONNECT_KEY_IDENTIFIER
    issuer_id: $APP_STORE_CONNECT_ISSUER_ID
    # same publishing options as above apply
```

### A.7 — Assert

- Build completes with `status == "finished"`
- App appears in App Store Connect under the correct bundle ID
- TestFlight build visible to testers (if `submit_to_testflight: true`)
- App Store submission visible in App Store Connect → App → App Store tab (if `submit_to_app_store: true`)

---

## Step B — Google Play

### B.1 — Inputs required

| Input | Description |
|---|---|
| `GCLOUD_SERVICE_ACCOUNT_CREDENTIALS` | JSON key for Google Play service account |
| `PACKAGE_NAME` | App package name (e.g. `com.example.app`) |
| `TRACK` | Target track: `internal`, `alpha`, `beta`, or `production` |
| `ROLLOUT_FRACTION` | Rollout percentage for production (0.0–1.0); omit for full rollout |

### B.2 — Create service account

1. Google Play Console → Setup → API access → Link to Google Cloud project
2. Google Cloud Console → IAM & Admin → Service Accounts → Create service account
3. Grant role: **Service Account User**
4. Create JSON key → download
5. In Google Play Console → Users and permissions → Invite new user → add service account email → grant **Release manager** permission

### B.3 — Upload credentials to Codemagic

**Option 1 — Codemagic UI (recommended):**
- App-level: App settings → Environment variables → add `GCLOUD_SERVICE_ACCOUNT_CREDENTIALS` as secure, assign to group `google_play_credentials`
- Team-level (shared across apps): Team settings → Global environment variables → same steps

**Option 2 — API:**
```shell
curl -X POST https://api.codemagic.io/teams/<TEAM_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "GCLOUD_SERVICE_ACCOUNT_CREDENTIALS",
    "value": "<JSON_KEY_CONTENTS>",
    "group": "google_play_credentials",
    "secure": true
  }'
```

### B.4 — Patch `codemagic.yaml`

```yaml
environment:
  groups:
    - google_play_credentials

publishing:
  google_play:
    credentials: $GCLOUD_SERVICE_ACCOUNT_CREDENTIALS
    track: internal                     # internal | alpha | beta | production
    submit_as_draft: false
    rollout_fraction: 0.1               # remove for full rollout; only for production track
```

### B.5 — Assert

- Build completes with `status == "finished"`
- Release visible in Google Play Console under the target track
- If `submit_as_draft: true` — release appears as draft requiring manual promotion

---

## Step C — Firebase App Distribution

### C.1 — Inputs required

| Input | Description |
|---|---|
| `FIREBASE_TOKEN` | Firebase CLI token or service account JSON |
| `FIREBASE_APP_ID` | Firebase App ID (found in Firebase Console → Project Settings → General) |
| `TESTER_GROUPS` | Comma-separated tester group aliases (optional) |

### C.2 — Generate Firebase token

```shell
# Interactive login — run locally, copy token to Codemagic
firebase login:ci
```

Or use a Google service account with Firebase App Distribution Admin role.

### C.3 — Upload credentials to Codemagic

**Option 1 — Codemagic UI (recommended):**
- App-level: App settings → Environment variables → add `FIREBASE_TOKEN` and `FIREBASE_APP_ID` as secure, assign to group `firebase_credentials`
- Team-level (shared across apps): Team settings → Global environment variables → same steps

**Option 2 — API:**
```shell
curl -X POST https://api.codemagic.io/teams/<TEAM_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "FIREBASE_TOKEN",
    "value": "<TOKEN>",
    "group": "firebase_credentials",
    "secure": true
  }'
```

Repeat for `FIREBASE_APP_ID`.

### C.4 — Patch `codemagic.yaml`

```yaml
environment:
  groups:
    - firebase_credentials

publishing:
  firebase:
    firebase_token: $FIREBASE_TOKEN
    android:
      app_id: $FIREBASE_APP_ID
      groups:
        - testers                       # tester group alias in Firebase Console
    ios:
      app_id: $FIREBASE_IOS_APP_ID      # separate App ID for iOS if needed
      groups:
        - testers
```

### C.5 — Assert

- Build completes with `status == "finished"`
- Release visible in Firebase Console → App Distribution
- Testers receive email notification if groups are configured

---

## Step D — Huawei AppGallery

### D.1 — Inputs required

| Input | Description |
|---|---|
| `HUAWEI_CLIENT_ID` | Client ID from AppGallery Connect API key |
| `HUAWEI_CLIENT_SECRET` | Client secret from AppGallery Connect API key |
| `HUAWEI_APP_ID` | App ID from AppGallery Connect |

### D.2 — Generate API credentials

AppGallery Connect → Users and permissions → API key → Create → download Client ID and Client Secret.

### D.3 — Upload credentials to Codemagic

**Option 1 — Codemagic UI (recommended):**
- App-level: App settings → Environment variables → add `HUAWEI_CLIENT_SECRET`, `HUAWEI_CLIENT_ID`, and `HUAWEI_APP_ID` as secure, assign to group `huawei_credentials`
- Team-level (shared across apps): Team settings → Global environment variables → same steps

**Option 2 — API:**
```shell
curl -X POST https://api.codemagic.io/teams/<TEAM_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "HUAWEI_CLIENT_SECRET",
    "value": "<CLIENT_SECRET>",
    "group": "huawei_credentials",
    "secure": true
  }'
```

Repeat for `HUAWEI_CLIENT_ID` and `HUAWEI_APP_ID`.

### D.4 — Patch `codemagic.yaml`

```yaml
environment:
  groups:
    - huawei_credentials

publishing:
  huawei:
    client_id: $HUAWEI_CLIENT_ID
    client_secret: $HUAWEI_CLIENT_SECRET
    app_id: $HUAWEI_APP_ID
    file_extension: aab                 # aab | apk
```

### D.5 — Assert

- Build completes with `status == "finished"`
- Release visible in AppGallery Connect under the correct app

---

## Error table

| Error | Root cause | Fix |
|---|---|---|
| `AuthenticationError: invalid API key` | App Store Connect `.p8` key invalid or expired | Regenerate API key in App Store Connect → Users and Access → Integrations |
| `No suitable application found` | Bundle ID mismatch or app record not created in App Store Connect | Verify bundle ID matches exactly; create app record manually in App Store Connect first |
| `The provided entity includes an attribute with a value that has already been used` | Duplicate build number — already uploaded to App Store Connect | Increment build number (`CFBundleVersion`) before triggering next build |
| `IPA not found / no artifacts` | `CODE_SIGNING_ALLOWED=NO` set in Xcode build settings, or `artifacts` section missing in `codemagic.yaml` | Check Xcode build settings; add IPA path to `artifacts` in `codemagic.yaml` |
| Build succeeds but nothing published | `artifacts` section missing or wrong path — IPA not captured | Add correct IPA glob to `artifacts` (e.g. `build/ios/ipa/*.ipa`) |
| Confusion between upload, submit, and release | `submit_to_testflight` uploads to TestFlight; `submit_to_app_store` submits for review; release is separate manual step in App Store Connect | Set flags explicitly per goal — see A.5 table |
| `Google Play: 403 forbidden` | Service account lacks Release Manager permission in Google Play Console | Re-grant permission: Google Play Console → Users and permissions → invite service account email with Release Manager role |
| `apkUploadFailed: Package not found` | App not yet created in Google Play Console, or package name mismatch | Create app manually in Google Play Console first; verify package name matches exactly |
| `Invalid track` | Track name not valid | Use exactly: `internal`, `alpha`, `beta`, or `production` |
| First upload to Google Play fails | Google Play requires first upload to be done manually via the Console | Upload first AAB/APK manually in Play Console; subsequent builds can use API |
| `Firebase: app not found` | Wrong `FIREBASE_APP_ID` | Copy App ID from Firebase Console → Project Settings → General → Your apps |
| `Firebase: permission denied` | Token expired or service account lacks Firebase App Distribution Admin role | Re-run `firebase login:ci` locally; or check service account IAM role in GCP |
| `Huawei: 403` | Client ID or secret invalid or expired | Regenerate API credentials in AppGallery Connect → Users and permissions → API key |
| `Huawei: app_id not found` | Wrong App ID | Verify App ID in AppGallery Connect → My Apps → App information |

---

## Invariants

These must always hold. Assert before marking skill complete:

1. Credentials stored as secure environment variables — never committed to the repository
2. The correct artifact path is captured in `artifacts` section of `codemagic.yaml`
3. Code signing is configured and working before publishing is attempted
4. The app exists in the target platform before the first publish attempt
5. Build number is unique for each App Store Connect submission
6. Service account or API key has the minimum required permissions for the target platform

---

## Scope boundaries

This skill does NOT handle:

- iOS code signing (separate skill)
- Android code signing (separate skill)
- App creation in App Store Connect or Google Play Console (must be done manually first)
- App Store review responses or rejection handling
- Google Play rollout promotion (internal → alpha → beta → production)
- In-app purchases, subscriptions, or metadata management
- Crash reporting or analytics configuration

---

## Reference documentation

| Topic | URL |
|---|---|
| App Store Connect publishing (codemagic.yaml) | https://docs.codemagic.io/yaml-publishing/app-store-connect/ |
| TestFlight publishing | https://docs.codemagic.io/yaml-publishing/testflight/ |
| Google Play publishing | https://docs.codemagic.io/yaml-publishing/google-play/ |
| Firebase App Distribution | https://docs.codemagic.io/yaml-publishing/firebase-app-distribution/ |
| Huawei AppGallery | https://docs.codemagic.io/yaml-publishing/huawei-appgallery/ |
| App Store Connect API | https://developer.apple.com/documentation/appstoreconnectapi |
| Google Play Developer API | https://developers.google.com/android-publisher |
| Firebase CLI reference | https://firebase.google.com/docs/cli |
| Codemagic environment variables | https://docs.codemagic.io/yaml-basic-configuration/configuring-environment-variables/ |
| Codemagic REST API — builds | https://docs.codemagic.io/rest-api/builds/ |
