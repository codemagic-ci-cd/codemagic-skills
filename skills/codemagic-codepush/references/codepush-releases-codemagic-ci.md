# CodePush Releases with Codemagic CI

Use Codemagic to run the CodePush CLI (`release-react`, promote, etc.) from CI. That step only produces and uploads a **JavaScript bundle**; it does **not** build the native app binary, so it should **not** be treated as part of a typical store-release workflow.

## 1) Add Codemagic workflow scripts

Add a **dedicated** workflow in `codemagic.yaml` for CodePush (for example triggered on its own branch, tag, or manual run). Do not assume it belongs inside your normal iOS/Android app build and submit pipelines—those compile and sign binaries; OTA delivery is a separate concern. Under each workflow id, `name` and `scripts` are siblings (same pattern as [Codemagic workflow YAML](https://docs.codemagic.io/yaml/yaml/)).

```yaml
workflows:
  codepush:
    name: CodePush
    scripts:
      - name: Install CodePush CLI
        script: npm install -g @codemagic/code-push-cli

      - name: CodePush login
        script: code-push login https://codepush.pro --access-key "$CODEPUSH_TOKEN"

      - name: Install dependencies
        script: npm ci

      - name: Release iOS OTA to Staging
        script: |
          code-push release-react MyApp-iOS ios --deploymentName Staging

      - name: Release Android OTA to Staging
        script: |
          code-push release-react MyApp-Android android --deploymentName Staging
```
