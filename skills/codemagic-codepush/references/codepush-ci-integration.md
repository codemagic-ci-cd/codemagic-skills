# CodePush CI Integration

CodePush updates can be published manually from a developer machine, but many teams choose to release OTA updates from CI pipelines.

Releasing from CI allows updates to be automatically deployed after successful builds, tests, or merges. This reduces manual release steps and keeps OTA updates consistent with the rest of your CI/CD process.

Typical CI release flow:

```
commit
→ CI build
→ tests pass
→ CodePush release command
→ update deployed
```

To publish updates from CI, the pipeline must:

1. Install the CodePush CLI
2. Authenticate using an access token
3. Run the release command

## Releasing from Codemagic

Codemagic workflows can publish OTA updates by running the CodePush CLI as part of a build step. They use the same commands you would run locally (such as `release-react`), and there is no separate "dashboard publish" mechanism for OTA releases.

Example step in `codemagic.yaml`:

```yaml
scripts:
  - name: Install CodePush CLI
    script: |
      npm install -g @codemagic/code-push-cli

  - name: Release CodePush update
    script: |
      code-push login --access-key $CODEPUSH_TOKEN
      code-push release-react MyApp-Android android \
        --deployment-name Production \
        --targetBinaryVersion "~$APP_VERSION" \
        --rollout 10
```

The access token should be stored as a **secure environment variable** in the Codemagic project settings (App Settings → Environment variables).

## Releasing from GitHub Actions

CodePush releases can also be triggered from GitHub Actions or other CI systems.

Create a repository secret (e.g. `CODEPUSH_TOKEN`). Without an `env` block, `$CODEPUSH_TOKEN` in `run` is empty — map the secret as shown, or use `${{ secrets.CODEPUSH_TOKEN }}` in the command directly.

```yaml
- name: Install CodePush CLI
  run: npm install -g @codemagic/code-push-cli

- name: Release CodePush update
  env:
    CODEPUSH_TOKEN: ${{ secrets.CODEPUSH_TOKEN }}
  run: |
    code-push login --access-key $CODEPUSH_TOKEN
    code-push release-react MyApp-Android android
```

The access token should be stored as a repository secret.

## Choosing When to Release OTA Updates

Teams can choose different strategies depending on workflow, release frequency, and risk tolerance.

**1. Release on every merge to main**

Every change merged into the main branch automatically triggers an OTA release.

```
merge to main → CI build → tests pass → OTA update published
```

**2. Release based on tags or specific commits**

OTA updates are only published when a version tag or specific commit is created.

```
tag created (e.g. v1.2.0) → CI build → OTA update published
```

**3. Manual CI-triggered releases**

OTA releases are triggered manually via the CI system.

```
developer triggers pipeline → CI build → OTA update published
```

OTA release strategy is not fixed — teams choose the level of automation based on how often they want to ship and how much control they need over production deployments.

## Best Practices

- Store access tokens as secure secrets — never hardcode them in YAML
- Restrict who can trigger OTA release pipelines
- Release to **Staging** first and promote to **Production** only after validation
- Monitor update metrics after each deployment
