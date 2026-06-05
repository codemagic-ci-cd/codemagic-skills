# Build Logs API (Debugging)

**Note:** Step log endpoints are **not documented** in official Codemagic docs but work in practice via `api.codemagic.io`. Documented API: https://docs.codemagic.io/rest-api/codemagic-rest-api/

## Authentication

**User:** Create an API token at [Codemagic settings](https://codemagic.io/settings) and make it available to the agent as `CM_API_TOKEN` (shell env var). Do not commit the token or paste it into yaml.

**Agent:** Use `x-auth-token: $CM_API_TOKEN` in API requests. If no token is available, use [Fallbacks](#fallbacks) (UI logs first).

Never echo, log, or commit real tokens.

## Workflow

When a build fails:

1. Resolve `build_id` from user paste (URL or ID), or `GET /apps` → `lastBuildId`, or `GET /builds?appId=`
2. `GET /builds/{build_id}` → inspect `buildActions[]` for first `failed` action
3. `GET /builds/{build_id}/step/{step_id}` → read raw log text
4. Grep log for `error`, `FAILED`, `exit code` if large
5. Map error to [common-failures.md](common-failures.md) or OS release lane
6. Fix yaml/signing → retry build

## Endpoints

Base URL: `https://api.codemagic.io`

| Method | Endpoint | Returns |
|--------|----------|---------|
| `GET` | `/apps` | Applications list, `lastBuildId` |
| `GET` | `/builds?appId={id}` | Builds with `_id`, `status`, `branch` |
| `GET` | `/builds/{build_id}` | `build.status`, `buildActions[]` with `_id`, `name`, `status` |
| `GET` | `/builds/{build_id}/step/{step_id}` | Raw step log (plain text) |

### Examples

List recent builds:

```bash
curl -s -H "x-auth-token: $CM_API_TOKEN" \
  "https://api.codemagic.io/builds?appId=YOUR_APP_ID"
```

Get build detail:

```bash
curl -s -H "x-auth-token: $CM_API_TOKEN" \
  "https://api.codemagic.io/builds/YOUR_BUILD_ID"
```

Get failed step log:

```bash
curl -s -H "x-auth-token: $CM_API_TOKEN" \
  "https://api.codemagic.io/builds/YOUR_BUILD_ID/step/YOUR_STEP_ID"
```

## Fallbacks

When API is unavailable or token not provided:

1. **UI** — Open failing step in Codemagic build log
2. **SSH** — https://docs.codemagic.io/troubleshooting/accessing-builder-machine-via-ssh/
3. **Artifacts** — Capture `/tmp/xcodebuild_logs/*.log` in workflow `artifacts:`

## v3 API

Newer endpoints may exist at `https://codemagic.io/api/v3/` — see https://codemagic.io/api/v3/schema. Legacy step endpoint above remains the practical path for per-step logs.
