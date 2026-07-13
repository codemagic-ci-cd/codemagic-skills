---
name: codemagic-env-variables
description: Agentic skill for configuring environment variables and variable groups on Codemagic. Covers app-level and team-level variables, secure variables, binary file encoding, cross-step variable passing, and referencing variables in codemagic.yaml. For use by AI agents autonomously setting up environment configuration for a Codemagic pipeline.
---

# Environment Variables — Agentic Skill

## Purpose

Configure environment variables in a Codemagic pipeline so that credentials, secrets, and configuration values are securely available during builds. Execute all steps autonomously. Do not pause for confirmation unless a hard blocker is encountered.

---

## Preconditions

Before executing this skill, assert all of the following:

- [ ] A `codemagic.yaml` exists at the repo root with at least one workflow
- [ ] The variable names and values to be added are known
- [ ] It is clear whether variables should be app-level or team-level (see Scope Selection below)

---

## Scope Selection — App-level vs Team-level

Decide where to store the variables before proceeding:

| Condition | Use |
|---|---|
| Variables are specific to one app | **App-level** (App settings → Environment variables) |
| Variables are shared across multiple apps in the team | **Team-level** (Team settings → Global variables and secrets) |
| Credentials used for signing or publishing across multiple apps | **Team-level** |
| Unsure → default to | **App-level** |

---

## Key Rules (assert before every action)

1. **All variables must belong to a group** — there is no loose variable outside a group
2. **The group must be explicitly referenced in `codemagic.yaml`** — adding a variable in the UI does nothing until the group is listed under `environment.groups` in the workflow
3. **Secret variables cannot be recovered after saving** — the value is encrypted and cannot be viewed again from the UI. Confirm the value is stored elsewhere before marking as Secret
4. **Binary files (keystores, `.p12`, profiles) must be base64-encoded** before storing as a variable value — decode them in a build script step
5. **Multi-line variables must be wrapped in quotes** when referenced in scripts: `"$VARIABLE_NAME"`

---

## Step 1 — Add variables to Codemagic

**Which option to use:**
- If assisting a human user → use **Option A (UI)**. Simpler, no API token required.
- If operating autonomously with a `CM_API_TOKEN` available → use **Option B (API)**.
- If Option B fails (401/403) → fall back to Option A and instruct the user to add variables manually.

### Option A — Codemagic UI

**App-level:**
1. Open the app in Codemagic → **Environment variables** tab
2. Enter variable name, value, and group name (group is created automatically if it doesn't exist)
3. Tick **Secret** if the value is sensitive (credentials, keys, tokens)
4. Click **Add**
5. Repeat for each variable

**Team-level:**
1. Codemagic → **Team settings → Global variables and secrets**
2. Enter variable name, value, and group name
3. Tick **Secret** if sensitive
4. Set access: choose specific apps or **All applications**
5. Click **Add**

**Bulk import (many variables at once):**
- Click **Add variables** → import from a `.env` file
- Mark individual variables as Secret using the lock icon in the upload modal

### Option B — API

**App-level variable:**
```shell
curl -X POST https://api.codemagic.io/apps/<APP_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "<VARIABLE_NAME>",
    "value": "<VARIABLE_VALUE>",
    "group": "<GROUP_NAME>",
    "secure": true
  }'
```

**Team-level variable:**
```shell
curl -X POST https://api.codemagic.io/teams/<TEAM_ID>/variables \
  -H "x-auth-token: <CM_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "<VARIABLE_NAME>",
    "value": "<VARIABLE_VALUE>",
    "group": "<GROUP_NAME>",
    "secure": true
  }'
```

Assert: HTTP 200/201 returned. If 401 → token invalid. If 403 → insufficient permissions.

---

## Step 2 — Reference group in `codemagic.yaml`

Every group containing variables used by the workflow must be listed here — otherwise variables will not be available on the build machine:

```yaml
workflows:
  my-workflow:
    environment:
      groups:
        - group_name_one
        - group_name_two
```

**Multiple groups with the same variable name:** the last listed group wins.

---

## Step 3 — Reference variables in scripts

Use `$VARIABLE_NAME` syntax anywhere in the workflow:

```yaml
scripts:
  - name: Example
    script: echo $MY_VARIABLE
```

**Multi-line variables — always wrap in quotes:**
```yaml
script: echo "$MULTILINE_VAR"
```

**Pass variables across steps** using the `CM_ENV` file:
```yaml
scripts:
  - name: Set variable for later steps
    script: echo "MY_KEY=my_value" >> $CM_ENV
  - name: Use variable from previous step
    script: echo $MY_KEY
```

**Multi-line variable across steps:**
```yaml
scripts:
  - name: Set multiline variable
    script: |
      echo 'MY_VAR<<DELIMITER' >> $CM_ENV
      echo 'line_one' >> $CM_ENV
      echo 'line_two' >> $CM_ENV
      echo 'DELIMITER' >> $CM_ENV
```

---

## Handling binary files (keystores, certificates, profiles)

Binary files cannot be pasted directly as variable values — encode them first.

**Encode on macOS:**
```bash
cat your_file.p12 | base64 | pbcopy
```

**Encode on Linux:**
```bash
openssl base64 -in your_file.p12
```

**Encode on Windows (PowerShell):**
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("your_file.p12")) | Set-Clipboard
```

> Always include PEM header/footer tags when encoding keys or certificates — `-----BEGIN PRIVATE KEY-----` etc. must be part of the stored value.

**Decode in build script:**
```yaml
scripts:
  - name: Decode binary file
    script: echo $YOUR_VARIABLE | base64 --decode > /tmp/your_file.p12
```

---

## Precedence (highest to lowest)

| Source | Priority |
|---|---|
| Variables passed via API at build trigger time | 1 — highest |
| App-level variables | 2 |
| Team-level (global) variables | 3 — lowest |

Within the same level: last group listed in `codemagic.yaml` wins if two groups define the same variable name.

---

## Injecting variables into app code

**Android — Gradle:**
```groovy
defaultConfig {
    resValue "string", "maps_api_key", "$System.env.MAPS_API_KEY"
}
```

**Flutter — `--dart-define`:**
```yaml
script: flutter build ipa --release --dart-define=MAPS_API_KEY=$MAPS_API_KEY
```
In Dart:
```dart
final secret = String.fromEnvironment('MAPS_API_KEY');
```

**iOS — Info.plist + Swift:**
```xml
<key>MAPS_API_KEY</key>
<string>$(MAPS_API_KEY)</string>
```
```swift
Bundle.main.object(forInfoDictionaryKey: "MAPS_API_KEY") as? String ?? ""
```

---

## Useful built-in variables

These are set automatically by Codemagic on every build — do not override them unless intentional:

| Variable | Value |
|---|---|
| `CI` | `true` |
| `CM_BUILD_ID` | UUID of the current build |
| `CM_BRANCH` | Current branch name |
| `CM_COMMIT` | Current commit hash |
| `CM_TAG` | Tag being built (unset if not a tag build) |
| `CM_PULL_REQUEST` | `true` if building a PR |
| `CM_PULL_REQUEST_DEST` | Target branch of the PR |
| `CM_BUILD_DIR` | Absolute path to the cloned repo root |
| `CM_ARTIFACT_LINKS` | JSON list of build artifacts (available in publishing steps) |
| `CM_ENV` | Path to environment file for cross-step variable passing |
| `BUILD_NUMBER` | Build count for this workflow |
| `CM_TRIGGER_SOURCE` | `webhook`, `schedule`, or `api` |

---

## Error table

| Error | Root cause | Fix |
|---|---|---|
| Variable not available in build | Group not referenced in `codemagic.yaml` | Add the group name under `environment.groups` in the workflow |
| Variable available in one step but not the next | Variable set inside a script step — not persisted by default | Write to `CM_ENV`: `echo "KEY=value" >> $CM_ENV` |
| Binary file decode fails | Missing or corrupted base64 encoding | Re-encode the file; ensure no line breaks were introduced during copy/paste |
| Secret variable value lost | Marked as Secret before saving a backup | Cannot recover — delete and re-add with the correct value |
| Two groups define same variable — wrong value used | Last group in `codemagic.yaml` wins | Reorder groups so the intended group is listed last |
| API 401 | Invalid `CM_API_TOKEN` | Regenerate token from Codemagic → Integrations → API |
| API 403 | Insufficient permissions | Use a team admin token |
| PEM key rejected | Header/footer tags missing from encoded value | Re-encode including `-----BEGIN/END PRIVATE KEY-----` tags |

---

## Invariants

These must always hold. Assert before marking skill complete:

1. All variables are stored in a named group — no loose variables
2. Every group used by the workflow is listed under `environment.groups` in `codemagic.yaml`
3. Sensitive values (credentials, keys, tokens) are marked as Secret
4. Binary files are base64-encoded before storing
5. No sensitive values are committed to the repository

---

## Scope boundaries

This skill does NOT handle:

- iOS code signing (separate skill — variables for signing are set up there)
- Android code signing (separate skill)
- Publishing credentials (covered in publishing skill)
- Build triggers or webhook configuration
- Codemagic API token generation (must be done manually in Codemagic UI → Integrations → API)

---

## Reference documentation

| Topic | URL |
|---|---|
| Configuring environment variables | https://docs.codemagic.io/yaml-basic-configuration/configuring-environment-variables/ |
| Using environment variables | https://docs.codemagic.io/yaml-basic-configuration/using-environment-variables/ |
| Built-in environment variables | https://docs.codemagic.io/yaml-basic-configuration/environment-variables/ |
| Codemagic REST API — applications | https://docs.codemagic.io/rest-api/applications/ |
