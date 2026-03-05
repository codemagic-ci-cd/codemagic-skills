# Verification And Troubleshooting

Use this checklist after wiring native config and CI commands.

## Verification Checklist

1. Confirm app wraps root component with `codePush(App)`.
2. Confirm server URL key is present:
   - iOS: `CodePushServerURL`
   - Android: `CodePushServerUrl`
3. Confirm deployment key is present for each platform.
4. Release to `Staging` first.
5. Install and launch app on device/simulator, then verify update download/apply behavior.
6. Check deployment metrics:

```bash
code-push deployment ls MyApp-iOS
code-push deployment ls MyApp-Android
```

7. Promote to `Production` only after staging validation.

## Debug Commands

```bash
code-push debug ios
code-push debug android
```

## Common Failures

1. Wrong deployment key or environment:
   - Symptom: update never appears.
   - Fix: run `code-push deployment ls <appName> -k` and rewire exact key.

2. Same app used for iOS and Android:
   - Symptom: package incompatibility or failed installs.
   - Fix: split to separate app names per platform.

3. Semver mismatch on iOS:
   - Symptom: release command errors on `CFBundleShortVersionString`.
   - Fix: use valid semver or pass explicit `--targetBinaryVersion`.

4. Native bundle path not overridden:
   - Symptom: app keeps using bundled JS only.
   - Fix: verify `CodePush.bundleURL()` (iOS) and `CodePush.getJSBundleFile()` (Android) are used in release path.

5. Custom project structure:
   - Symptom: `release-react` cannot detect versions.
   - Fix: pass explicit paths (`--plistFile`, `--gradleFile`) or set `--targetBinaryVersion`.
