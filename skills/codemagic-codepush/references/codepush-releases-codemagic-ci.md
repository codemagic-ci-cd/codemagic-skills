# CodePush Releases with Codemagic CI

Use this flow to connect Codemagic workflow execution to CodePush delivery.

## 1) Add Codemagic workflow scripts

Create a standalone workflow in your `codemagic.yaml` to handle codepush releases, or add to your iOS and Android production release workflows.

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

