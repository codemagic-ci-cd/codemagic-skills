---
name: rollbar-auto-fix
description: Complete workflow to find, investigate, and fix Rollbar errors. Checks requirements, lists issues for selection, creates a branch, and generates a Pull Request. Use when users want to solve an error reported on Rollbar for a specific project.
user-invocable: true
triggers:
  - "rollbar"
  - "rollbar production error"
  - "fix rollbar"
allowed-tools: Bash(curl *, git *, gh *), Read(*), Write(*)
---


# Rollbar Autofix Workflow

This skill follows a structured process to solve production errors reported in Rollbar.

## Step 1: Verification

Check for all required tools and credentials. Load secrets from `.env` (always use the absolute path). Then detect the default branch and ensure it is checked out and up to date.

```bash
ROOT="$(git rev-parse --show-toplevel)"
source "$ROOT/.env"

# Check required tools
curl --version > /dev/null 2>&1 || echo "ERROR: curl not found"
gh --version > /dev/null 2>&1   || echo "ERROR: gh CLI not found — install it and run 'gh auth login'"

# Check required tokens
[ -z "$ROLLBAR_ACCESS_TOKEN" ] && echo "ERROR: ROLLBAR_ACCESS_TOKEN missing in .env"
[ -z "$GITHUB_TOKEN" ]         && echo "ERROR: GITHUB_TOKEN missing in .env"
```

If any check fails, stop and tell the user what is missing before proceeding.

Required secrets in `.env`:
- `ROLLBAR_ACCESS_TOKEN` — Rollbar read token. If is missed instruct the user to go to Rollbar -> Projects -> Project Access Tokens, and create a Read token.
- `GITHUB_TOKEN` — GitHub personal access token with `repo` scope (used by `gh`)

After the checks pass, detect the default branch and ensure it is current:

```bash
DEFAULT_BRANCH="$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')"

git checkout "$DEFAULT_BRANCH" && git pull
```

If `git checkout` or `git pull` fails, stop and tell the user to resolve the issue manually (e.g. uncommitted changes, conflicts) before continuing.

## Step 2: List Issues

Fetch active errors and show **ID**, **Title** and **Occurrences** as a table. Don't summarize — show the raw list.

```bash
curl -s "https://api.rollbar.com/api/1/items/?status=active&level=error" \
  -H "X-Rollbar-Access-Token: $ROLLBAR_ACCESS_TOKEN" | \
  python3 -c "import sys, json; data = json.load(sys.stdin); [print(f'[{i[\"id\"]}] {i[\"title\"]} ({i[\"total_occurrences\"]} occurrences)') for i in data['result']['items']]"
```

## Step 3: Investigation & Fix

Once the user selects an ID, create the fix branch and investigate:

```bash
git checkout -b rollbar/issue-autofix-<issue-id>
```

1. Get full context of the issue from Rollbar API `/api/1/item/<id>/instances/`.
2. Analyze code, find the bug, and apply the fix.
3. Verify the fix running tests and linting code, don't continue to next step until this step is successful.

## Step 4: Pull Request

Commit the fix, push the branch, and create the PR using `gh`:

```bash
DEFAULT_BRANCH="$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')"

# Commit
git add <changed files>
git commit -m "<concise description of the fix>"

# Push
git push -u origin rollbar/issue-autofix-<issue-id>

# Create PR (GITHUB_TOKEN must be set)
GITHUB_TOKEN=$GITHUB_TOKEN gh pr create \
  --title "<title>" \
  --body "<body>" \
  --base "$DEFAULT_BRANCH" \
  --head rollbar/issue-autofix-<issue-id>
```

The PR body must follow this template:

```
## Summary

- Fixes [Rollbar #<issue-id>](https://rollbar.com/item/<issue-id>/) (<N> occurrences)
- <Root cause explanation>
- <Description of the fix>

## Test plan

- [ ] <test item>
- [ ] <test item>
```

