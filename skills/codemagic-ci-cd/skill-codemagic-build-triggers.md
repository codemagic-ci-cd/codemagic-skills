---
name: codemagic-build-triggers
description: Agentic skill for configuring build triggers on Codemagic. Covers push, pull request, tag, and label-based triggers; branch and tag patterns; webhook setup for GitHub, GitLab, Bitbucket, Azure DevOps, and AWS CodeCommit; scheduled builds; and conditional build logic. For use by AI agents autonomously configuring automated build triggering in a Codemagic pipeline.
---

# Build Triggers — Agentic Skill

## Purpose

Configure automated build triggers in a Codemagic pipeline so that builds start automatically on the right events, branches, and conditions. Execute all steps autonomously. Do not pause for confirmation unless a hard blocker is encountered.

---

## Preconditions

Before executing this skill, assert all of the following:

- [ ] A `codemagic.yaml` exists at the repo root with at least one working workflow
- [ ] The Git provider is known (GitHub, GitLab, Bitbucket, Azure DevOps, AWS CodeCommit)
- [ ] The trigger events and target branches/tags are known

> **Critical rule:** If no `triggering:` block is defined, builds can only be started manually.

---

## Step 1 — Identify trigger type

Determine which trigger(s) are needed and proceed to the corresponding section:

| Trigger | Section |
|---|---|
| Push to branch | Step 2A |
| Pull request opened or updated | Step 2B |
| Tag created | Step 2C |
| PR label added (GitHub only) | Step 2D |
| Scheduled builds | Step 2E |
| Conditional / file-based | Step 3 |

Multiple triggers can be combined in the same workflow.

---

## Step 2A — Push trigger

```yaml
triggering:
  events:
    - push
  branch_patterns:
    - pattern: 'main'
      include: true
      source: true
    - pattern: 'develop'
      include: true
      source: true
  cancel_previous_builds: false
```

**Common branch pattern examples:**

| Pattern | Matches |
|---|---|
| `*` | All branches |
| `main` | Exact match |
| `release/*` | Any branch starting with `release/` |
| `*-dev` | Any branch ending with `-dev` |
| `!(*-dev)` | Any branch NOT ending with `-dev` |
| `{main,develop}` | `main` or `develop` |
| `v+([0-9]).+([0-9]).+([0-9])` | Semantic version tags like `v1.0.0` |

**Pattern matching rule:** The first matching pattern applies. Each following pattern further limits the set.

**`cancel_previous_builds`:** Set to `true` to automatically cancel outdated builds when a newer push arrives for the same workflow. Default: `false`.

---

## Step 2B — Pull request trigger

```yaml
triggering:
  events:
    - pull_request
  branch_patterns:
    - pattern: 'main'
      include: true
      source: false   # false = pattern applies to TARGET (destination) branch
```

**`source` field:**
- `source: false` — pattern applies to the **target (destination)** branch of the PR
- `source: true` — pattern applies to the **source** branch of the PR

**Common PR patterns:**

Build when PR targets `main`:
```yaml
branch_patterns:
  - pattern: 'main'
    include: true
    source: false
```

Build when PR targets `main` OR `develop`:
```yaml
branch_patterns:
  - pattern: '{main,develop}'
    include: true
    source: false
```

Build on merge into `main` (after PR is merged):
```yaml
events:
  - push
  - pull_request
branch_patterns:
  - pattern: 'main'
    include: true
    source: true
```

**Skip draft PRs** (use `when:` condition — see Step 3B):
```yaml
when:
  condition: not event.pull_request.draft
```

---

## Step 2C — Tag trigger

```yaml
triggering:
  events:
    - tag
  tag_patterns:
    - pattern: '*'
      include: true
    - pattern: 'v+([0-9]).+([0-9]).+([0-9])'
      include: true
```

> Branch patterns do not apply to tag triggers — use `tag_patterns:` instead.

---

## Step 2D — PR label trigger (GitHub only)

```yaml
triggering:
  events:
    - pull_request_labeled
```

Filter by specific label using `when:` condition:
```yaml
when:
  condition: event.pull_request.labels[0].name == "build-ready"
```

---

## Step 2E — Scheduled builds

Scheduled builds are configured entirely in the **Codemagic UI** — there is no YAML syntax for scheduling.

1. Open app in Codemagic → **Scheduled builds** tab
2. Click **Add new schedule**
3. Select **Branch** and **Workflow**
4. Select days of the week and time (**UTC — no timezone selection**)
5. Click **Add schedule**

**Limitations:**
- All times are UTC
- Day-of-week + time only — no cron expressions, no "every N hours", no specific dates
- Build start may be delayed up to **15 minutes** during peak hours
- The `event` variable in `when: condition` is not available for scheduled builds
- No built-in retry on failure

---

## Step 3 — Conditional build logic

### Step 3A — File-based conditions (`changeset`)

Prevents unnecessary builds when only certain files changed:

```yaml
workflows:
  my-workflow:
    triggering:
      events:
        - push
    when:
      changeset:
        includes:
          - '.'           # watch everything
        excludes:
          - '**/*.md'     # except markdown files
```

**Monorepo — only build when relevant directory changes:**
```yaml
when:
  changeset:
    includes:
      - 'android/'
```

**Critical caveats:**
- `codemagic.yaml` is **always** included in the changeset — changes to it always trigger a build regardless of `excludes`
- After `changeset` is first added to `codemagic.yaml`, the **very next build will trigger regardless of the condition** — changeset logic applies from the second build onward

### Step 3B — Expression-based conditions (`condition`)

Evaluates environment variables or the webhook payload. Use `env.VAR_NAME` syntax — **not** `$VAR_NAME`.

**Skip draft PRs:**
```yaml
when:
  condition: not event.pull_request.draft
```

**Match specific branch:**
```yaml
when:
  condition: env.CM_BRANCH == "main"
```

**Filter by label:**
```yaml
when:
  condition: event.pull_request.labels[0].name == "build-ready"
```

**Combined logic:**
```yaml
when:
  condition: (not event.pull_request.draft) and (not event.pull_request.labels[0].name == "skip-build")
```

**Condition on individual steps:**
```yaml
scripts:
  - name: Run tests
    script: flutter test
    when:
      condition: env.RUN_TESTS == "true"
```

**Critical caveats:**
- Condition is evaluated **after** cloning the repository — builds always start; if condition is not met, build is marked **skipped** (not failed)
- The `event` variable is only available for webhook-triggered builds — not for manual or scheduled builds
- Shell-style `$VAR` syntax is **not supported** inside `when:` — use `env.VAR_NAME`

### Step 3C — Skip build via commit message

Include `[skip ci]` or `[ci skip]` in the commit message to prevent Codemagic from triggering a build for that commit.

**Custom keyword logic:**
```yaml
scripts:
  - name: Check commit message for build keyword
    script: |
      set -e
      COMMIT_MSG=$(git log -1 --pretty=%B)
      if [[ $COMMIT_MSG != *"buildcd"* ]]; then
        echo "Commit message does not include 'buildcd' — skipping build."
        exit 1
      fi
```

---

## Step 4 — Webhook setup

Automatic triggers require a webhook registered on the Git provider. Many OAuth-connected providers register webhooks automatically. For manual setups:

**Payload URL format:**
```
https://api.codemagic.io/hooks/<appId>
```

Find `appId` in the Codemagic browser URL: `codemagic.io/app/<appId>`

### GitHub

Path: Repository → **Settings → Webhooks → Add webhook**

- Payload URL: `https://api.codemagic.io/hooks/<appId>`
- Content type: `application/json`
- Events to select:
  - Branch or tag creation
  - Pull requests
  - Pushes

### GitLab

Path: Repository → **Settings → Webhooks**

- URL: `https://api.codemagic.io/hooks/<appId>`
- Trigger options to enable:
  - Push events
  - Tag push events
  - Merge request events
- Enable **SSL verification**

### Bitbucket

Path: Repository → **Settings → Webhooks → Add webhook**

- URL: `https://api.codemagic.io/hooks/<appId>`
- Triggers to select:
  - Repository: **Push**
  - Pull Request: **Created**, **Updated**, **Merged**

### Azure DevOps

Path: **Project Settings → Service Hooks → Create subscription → Web Hooks**

- Each event type requires its **own separate webhook**
- Create one webhook per event: Code pushed, Pull request created, Pull request updated
- Configure repository filters on each webhook as needed

### AWS CodeCommit

1. Create a **Standard SNS topic**
2. Create an **HTTPS subscription** using `https://api.codemagic.io/hooks/<appId>`
3. **Disable raw message delivery** on the subscription
4. In the repository → create a **notification rule** with events:
   - Pull requests: Source updated, Created
   - Branches/tags: Created, Updated

### Webhook auto-setup vs manual

| Connection method | Webhook setup |
|---|---|
| GitHub App / OAuth App | Usually automatic |
| GitLab OAuth | Usually automatic |
| Bitbucket OAuth | Usually automatic |
| HTTP / HTTPS repository | Manual — follow provider steps above |
| SSH repository | Manual — follow provider steps above |
| AWS CodeCommit | Manual via SNS |
| Azure DevOps | Manual — one webhook per event type |

**Stale webhooks:** If a webhook stops working after app migration, a team admin can update it via the **"Update webhook"** button in Codemagic app settings.

---

## Error table

| Error | Root cause | Fix |
|---|---|---|
| Build not triggering on push | No `triggering:` block or `push` not in `events` | Add `triggering: events: - push` to the workflow |
| Build triggers on wrong branch | Branch pattern too broad or `source` flag incorrect | Tighten `branch_patterns`; verify `source: true/false` for PR targets |
| PR build not triggering | `pull_request` not in events, or webhook not registered | Add `pull_request` to events; verify webhook is set up on the Git provider |
| Tag build not triggering | Using `branch_patterns` instead of `tag_patterns` for tags | Replace `branch_patterns` with `tag_patterns` for tag events |
| `condition` not working | Using `$VAR` syntax instead of `env.VAR` | Replace `$CM_BRANCH` with `env.CM_BRANCH` in `when:` expressions |
| Build triggered despite `changeset` exclude | `codemagic.yaml` itself was changed | Expected — YAML changes always trigger a build regardless of changeset config |
| First build after adding `changeset` ignores condition | Known behaviour — first build always runs | Expected — changeset logic applies from second build onward |
| `event` variable not available | Build triggered manually or by schedule | `event` is only available for webhook-triggered builds |
| Azure DevOps: only one event type triggering | Each event type needs its own webhook | Create separate webhook subscriptions for each event in Azure DevOps |
| Scheduled build delayed | Peak hours | Expected — up to 15 minutes delay during peak; no fix needed |

---

## Invariants

These must always hold. Assert before marking skill complete:

1. At least one event is listed under `triggering.events` — without it, builds are manual only
2. For tag builds: `tag_patterns:` is used, not `branch_patterns:`
3. For PR target-branch filtering: `source: false` is set on the relevant pattern
4. Webhook is registered on the Git provider for automatic triggering to work
5. `when: condition` uses `env.VAR_NAME` syntax, not `$VAR_NAME`
6. Azure DevOps has one webhook per event type

---

## Scope boundaries

This skill does NOT handle:

- Build configuration (scripts, artifacts, instance type)
- Code signing (separate skills)
- Publishing or notifications (separate skills)
- Codemagic API-triggered builds (separate API usage)
- Monorepo workflow routing beyond `changeset` conditions

---

## Reference documentation

| Topic | URL |
|---|---|
| Starting builds automatically | https://docs.codemagic.io/yaml-running-builds/starting-builds-automatically/ |
| Webhooks | https://docs.codemagic.io/yaml-running-builds/webhooks/ |
| Scheduling builds | https://docs.codemagic.io/yaml-running-builds/scheduling-builds/ |
| Built-in environment variables | https://docs.codemagic.io/yaml-basic-configuration/environment-variables/ |
