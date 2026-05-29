# CodePush Releasing Updates and Production Controls

## Standard Release Flow: Staging → Production

```bash
# Release to Staging first
code-push release-react MyApp-Android android --deployment-name Staging

# After testing, promote to Production
code-push promote MyApp-Android Staging Production
```

## Key Release Flags

| Flag | Purpose |
|---|---|
| `--deployment-name` | Target deployment (default: Staging) |
| `--targetBinaryVersion` | Semver range of native binaries this update is compatible with (e.g. `~1.2.0`) |
| `--mandatory` | Force install — users cannot skip |
| `--rollout <n>` | Gradual rollout to n% of users |
| `--description` | Release notes shown in update dialog |

---

## Gradual Rollout

```bash
# Start at 25%
code-push release-react MyApp-Android android --rollout 25

# Increase without a new release
code-push patch MyApp-Android Production --rollout 50
```

**Constraints:**
- Only one active partial rollout per deployment
- Must reach 100% before publishing a new update to the same deployment

---

## Mandatory Updates

Use for security fixes, critical bugs, or breaking changes:

```bash
code-push release-react <app_name> <platform> --mandatory
# Or patch an existing release
code-push patch <app_name> Production --mandatory true
```

`mandatoryInstallMode` in the SDK controls when the mandatory update is actually applied.

---

## Manual Rollback

```bash
code-push rollback <app_name> <deployment_name>
```

**Automatic rollback:** If the app crashes before calling `notifyAppReady()`, CodePush reverts to the previous bundle automatically on next restart.
