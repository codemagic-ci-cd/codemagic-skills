---
name: codemagic-notifications
description: Agentic skill for configuring build notifications on Codemagic. Covers email, Slack, Microsoft Teams, and Firebase App Distribution notifications. For use by AI agents autonomously setting up notification and distribution configuration in a Codemagic pipeline.
---

# Notifications — Agentic Skill

## Purpose

Configure build notifications in a Codemagic pipeline so that the right people are informed of build outcomes via the right channel. Execute all steps autonomously. Do not pause for confirmation unless a hard blocker is encountered.

---

## Preconditions

Before executing this skill, assert all of the following:

- [ ] A `codemagic.yaml` exists at the repo root with at least one working workflow
- [ ] The target notification channel(s) are known (email, Slack, Teams, Firebase)
- [ ] Required credentials or webhook URLs are available

---

## Channel Selection

Identify the target channel(s) from the user's request and execute the corresponding section(s). Multiple channels can be configured in the same workflow.

| Channel | Type | Section |
|---|---|---|
| Email | Native integration | Step A |
| Slack | Native integration | Step B |
| Microsoft Teams | Script-based (webhook) | Step C |
| Firebase App Distribution | Native integration | Step D |

---

## Step A — Email

*Source: docs.codemagic.io/yaml-notification/email/*

### A.1 — YAML configuration

```yaml
publishing:
  email:
    recipients:
      - name@example.com
      - another@example.com
    notify:
      success: true    # set to false to suppress success emails
      failure: true    # set to false to suppress failure emails
```

### A.2 — Fields

| Field | Required | Default | Description |
|---|---|---|---|
| `recipients` | Yes | — | List of email addresses to notify |
| `notify.success` | No | `true` | Send email on successful build |
| `notify.failure` | No | `true` | Send email on failed build |

### A.3 — Behaviour

- **Success:** sends release notes (if provided) + artifact download links
- **Failure:** sends a link to the build logs
- Artifact download links expire after **24 hours** by default — configurable in Account/Team Settings → Artifact download links

### A.4 — Critical caveat

Email is only sent when artifacts are available for Codemagic to collect. If build scripts run cleanup steps (e.g. `flutter clean`) **before** Codemagic collects artifacts, the binaries are deleted and no email is sent — even if the build succeeded. Cleanup steps must run **after** artifact collection.

### A.5 — Assert

- Build completes and email is received by all recipients
- If no email received on success → check artifact path in `artifacts` section; check for premature cleanup scripts

---

## Step B — Slack

*Source: docs.codemagic.io/yaml-notification/slack/*

### B.1 — Connect Slack workspace (one-time setup)

This must be done in the Codemagic UI before the YAML config will work:

1. Codemagic → Account or Team **Settings → Integrations**
2. Click **Connect** next to Slack
3. Authorize the Codemagic app on the Slack permission screen
4. Click **Allow**

This is a workspace-level connection — done once, applies to all apps in the account/team.

### B.2 — YAML configuration

```yaml
publishing:
  slack:
    channel: '#channel-name'
    notify_on_build_start: true    # optional
    notify:
      success: true    # set to false to suppress success notifications
      failure: true    # set to false to suppress failure notifications
```

### B.3 — Fields

| Field | Required | Default | Description |
|---|---|---|---|
| `channel` | Yes | — | Target Slack channel — must include `#` |
| `notify_on_build_start` | No | `false` | Send notification when build starts |
| `notify.success` | No | `true` | Post to channel on successful build |
| `notify.failure` | No | `true` | Post to channel on failed build |

### B.4 — Behaviour

- **Success:** posts release notes and artifact links to the channel
- **Failure:** posts a link to the build logs
- **Build start:** optional notification when `notify_on_build_start: true`
- Artifact download links expire after **24 hours** by default

### B.5 — Private channels

The Codemagic Slack app must be invited to private channels before it can post:
1. Open the private channel in Slack
2. Type `@codemagic` and send
3. Confirm the invite when prompted
4. If workspace restrictions exist, a Slack admin may need to approve

### B.6 — Assert

- Build notification appears in the target channel
- If not → confirm Slack workspace is connected in Settings → Integrations; confirm channel name is correct including `#`; confirm Codemagic app is invited if channel is private

---

## Step C — Microsoft Teams

*Source: docs.codemagic.io/integrations/ms-teams-integration/*

Teams notifications are **not a native integration** — they are implemented via a custom script using an Incoming Webhook URL.

### C.1 — Create Incoming Webhook in Teams

1. In the target Teams channel → click **More options (⋮)**
2. Select **Connectors → Edit**
3. Add **Incoming Webhook** connector
4. Give it a name → click **Create**
5. Copy the generated webhook URL

### C.2 — Store webhook URL in Codemagic

**Which option to use:**
- If assisting a human user → use **Option A (UI)**.
- If operating autonomously with a `CM_API_TOKEN` → use **Option B (API)**.
- If Option B fails → fall back to Option A.

**Option A — Codemagic UI:**
- App settings → Environment variables → add `TEAMS_WEBHOOK_URL` as Secret, assign to group `teams_credentials`

**Option B — API:**
```shell
curl -X POST https://api.codemagic.io/apps/<APP_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "TEAMS_WEBHOOK_URL",
    "value": "<WEBHOOK_URL>",
    "group": "teams_credentials",
    "secure": true
  }'
```

### C.3 — YAML configuration

```yaml
environment:
  groups:
    - teams_credentials

publishing:
  scripts:
    - name: Send Microsoft Teams notification
      script: |
        # Fetch artifact URL (adjust endswith filter for .apk if needed)
        ARTIFACT_LINK=$(echo $CM_ARTIFACT_LINKS | jq -r '.[] | select(.name | endswith(".ipa")) | .url')

        # Get short commit hash
        COMMIT=$(echo "${CM_COMMIT}" | sed 's/^\(........\).*/\1/;q')

        # Get commit message and author
        COMMIT_MESSAGE=$(git log --format=%B -n 1 $CM_COMMIT)
        AUTHOR=$(git show -s --format='%ae' $CM_COMMIT)

        curl -H "Content-Type: application/json" \
          -d "{
                \"title\": \"New Codemagic Build\",
                \"text\": \"**Branch:** $CM_BRANCH <br>**Commit:** \`$COMMIT\` <br>**Author:** $AUTHOR <br>**Message:** $COMMIT_MESSAGE <br>**Artifact:** <a href='$ARTIFACT_LINK'>Download</a>\"
              }" \
          $TEAMS_WEBHOOK_URL
```

### C.4 — Conditional success/failure notifications

There is no native `notify.success` / `notify.failure` toggle — use shell logic:

```yaml
publishing:
  scripts:
    - name: Send Teams notification
      script: |
        if [ "$CM_BUILD_STEP_STATUS" = "failed" ]; then
          MESSAGE="Build failed on branch $CM_BRANCH"
        else
          ARTIFACT_LINK=$(echo $CM_ARTIFACT_LINKS | jq -r '.[] | select(.name | endswith(".ipa")) | .url')
          MESSAGE="Build succeeded. <a href='$ARTIFACT_LINK'>Download artifact</a>"
        fi
        curl -H "Content-Type: application/json" \
          -d "{\"text\": \"$MESSAGE\"}" \
          $TEAMS_WEBHOOK_URL
```

### C.5 — Assert

- Teams message appears in the target channel after build completes
- If not → verify webhook URL is correct and stored as Secret; confirm group is referenced in `environment.groups`; confirm `jq` is available (pre-installed on Codemagic macOS machines)

---

## Step D — Firebase App Distribution

*Source: docs.codemagic.io/yaml-distributing/firebase-app-distribution/*

Firebase App Distribution distributes builds directly to tester groups — it is a distribution channel, not just a notification.

### D.1 — Authentication setup (service account — recommended)

1. Firebase Console → **Project Settings → Service Accounts → Manage service account permissions**
2. Google Cloud Console → IAM & Admin → Service Accounts → Create service account
3. Assign role: **Firebase App Distribution Admin**
4. Generate JSON key → download

> **Note:** Firebase token (`firebase_token`) is deprecated. Use service account.

### D.2 — Store credentials in Codemagic

**Which option to use:**
- If assisting a human user → use **Option A (UI)**.
- If operating autonomously with a `CM_API_TOKEN` → use **Option B (API)**.
- If Option B fails → fall back to Option A.

**Option A — Codemagic UI:**
- App settings → Environment variables → add `FIREBASE_SERVICE_ACCOUNT` (paste full JSON key contents) as Secret, assign to group `firebase_credentials`

**Option B — API:**
```shell
curl -X POST https://api.codemagic.io/apps/<APP_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "FIREBASE_SERVICE_ACCOUNT",
    "value": "<JSON_KEY_CONTENTS>",
    "group": "firebase_credentials",
    "secure": true
  }'
```

### D.3 — YAML configuration

```yaml
environment:
  groups:
    - firebase_credentials

publishing:
  firebase:
    firebase_service_account: $FIREBASE_SERVICE_ACCOUNT
    android:
      app_id: x:xxxxxxxxxxxx:android:xxxxxxxxxxxxxxxxxxxxxx   # from Firebase console
      groups:
        - androidTesters
      artifact_type: 'apk'    # 'apk' or 'aab' — defaults to 'aab'
    ios:
      app_id: x:xxxxxxxxxxxx:ios:xxxxxxxxxxxxxxxxxxxxxx       # from Firebase console
      groups:
        - iosTesters
```

### D.4 — Fields

| Field | Platform | Required | Description |
|---|---|---|---|
| `firebase_service_account` | Both | Yes | Service account JSON env var |
| `android.app_id` | Android | Yes | App ID from Firebase console |
| `android.groups` | Android | Yes | Tester group names from Firebase console |
| `android.artifact_type` | Android | No | `'apk'` or `'aab'` — defaults to `'aab'` |
| `ios.app_id` | iOS | Yes | App ID from Firebase console |
| `ios.groups` | iOS | Yes | Tester group names from Firebase console |

### D.5 — Release notes

Create a `release_notes.txt` file in the project root — Codemagic picks it up automatically and includes it with the distribution.

### D.6 — iOS signing requirement

iOS distribution via Firebase requires a **development, Ad Hoc, or Enterprise** provisioning profile. App Store profiles will not work.

### D.7 — Android AAB requirement

The Firebase project must be linked to a Google Play account for AAB distribution to work. If not linked, use `artifact_type: 'apk'` instead.

### D.8 — Assert

- Build completes with `status == "finished"`
- Testers receive email from Firebase with download link
- Release appears in Firebase Console → App Distribution under the correct app

---

## Comparison table

| Feature | Email | Slack | MS Teams | Firebase |
|---|---|---|---|---|
| Native integration | Yes | Yes | No (script) | Yes |
| `notify.success` toggle | Yes | Yes | Manual (shell) | No |
| `notify.failure` toggle | Yes | Yes | Manual (shell) | No |
| `notify_on_build_start` | No | Yes | Manual | No |
| UI auth setup required | No | Yes (OAuth) | No | No |
| Artifact links in message | Yes | Yes | Yes (via jq) | N/A |
| Link expiration | 24h | 24h | Codemagic default | N/A |

---

## Error table

| Error | Root cause | Fix |
|---|---|---|
| Email not received on success | Cleanup script runs before artifact collection | Move cleanup steps after artifact collection phase |
| Slack notification not posted | Workspace not connected, wrong channel name, or app not invited | Connect workspace in Settings → Integrations; verify `#channel-name`; invite `@codemagic` to private channels |
| Teams notification not sent | Wrong webhook URL, group not referenced, or `jq` not available | Verify `TEAMS_WEBHOOK_URL` value; add group to `environment.groups`; confirm `jq` is installed |
| Firebase: `permission denied` | Service account missing Firebase App Distribution Admin role | Re-assign role in Google Cloud IAM; re-download and re-upload JSON key |
| Firebase: `app not found` | Wrong `app_id` | Copy App ID from Firebase Console → Project Settings → General → Your apps |
| Firebase iOS: build not distributed | App Store provisioning profile used | Switch to development, Ad Hoc, or Enterprise profile for Firebase distribution |
| Firebase Android AAB fails | Firebase project not linked to Google Play | Link in Firebase Console, or switch to `artifact_type: 'apk'` |
| `firebase_token` auth stopped working | Token-based auth deprecated by Firebase | Migrate to service account authentication |

---

## Invariants

These must always hold. Assert before marking skill complete:

1. Credentials and webhook URLs are stored as Secret environment variables — never hardcoded in `codemagic.yaml`
2. Variable groups are referenced under `environment.groups` in the workflow
3. For Slack: workspace is connected in Settings → Integrations
4. For Firebase iOS: distribution profile is development, Ad Hoc, or Enterprise — not App Store
5. For Firebase: service account is used, not deprecated `firebase_token`

---

## Scope boundaries

This skill does NOT handle:

- Publishing to App Store Connect or Google Play (separate publishing skill)
- iOS or Android code signing (separate skills)
- Webhook-based build triggers (separate configuration)
- PagerDuty, OpsGenie, or other alerting integrations

---

## Reference documentation

| Topic | URL |
|---|---|
| Email notifications | https://docs.codemagic.io/yaml-notification/email/ |
| Slack notifications | https://docs.codemagic.io/yaml-notification/slack/ |
| Microsoft Teams integration | https://docs.codemagic.io/integrations/ms-teams-integration/ |
| Firebase App Distribution | https://docs.codemagic.io/yaml-distributing/firebase-app-distribution/ |
| Built-in environment variables | https://docs.codemagic.io/yaml-basic-configuration/environment-variables/ |
