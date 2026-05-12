# Verification And Troubleshooting

Use this checklist after wiring native config and CI commands.

## Verification Checklist

1. Confirm the JS entry uses CodePush (default `codePush(App)` or an equivalent HOC / `sync()` setup).
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
   - Fix: pass explicit paths (`--plistFile`, `--gradleFile`) or set `--targetBinaryVersion` on `release-react` (see [codepush-cli.md](codepush-cli.md)).

6. OTA feels like a random restart or blank flash when returning from background:
   - Symptom: user switches apps briefly and the app reloads or flashes white when they come back.
   - Fix: review `checkFrequency`, `installMode`, `mandatoryInstallMode`, and `minimumBackgroundDuration` against how you ship updates ([Advanced: sync options](https://docs.codemagic.io/rn-codepush/advanced-sync-options/)).

7. Update kicks users out of checkout, onboarding, or another critical flow:
   - Symptom: session or flow is interrupted when CodePush applies an update or restarts the app.
   - Fix: wrap fragile sections with `disallowRestart` / `allowRestart`, or defer activation to a safer install mode (same doc); align mandatory behavior with [Production control](https://docs.codemagic.io/rn-codepush/production-control/).

8. Optional updates rarely take effect on real devices:
   - Symptom: users stay on an old bundle for a long time even though releases succeed.
   - Fix: optional installs often wait for a full process restart; if users seldom cold-start the app, change check/install strategy or prompt for restart ([Advanced: sync options](https://docs.codemagic.io/rn-codepush/advanced-sync-options/)).

## See also (Codemagic docs)

- [Issues and debugging](https://docs.codemagic.io/rn-codepush/debugging-and-common-issues/) — native binary without CodePush SDK (zero installs), wrong `targetBinaryVersion`, **`notifyAppReady`** when not using default `sync()` on startup, source maps for crash tools, filtered `[CodePush]` logs.
