---
name: codemagic-mobile-ci-cd
description: Guides end-to-end Codemagic CI/CD for Flutter, React Native, and native mobile apps—from first build through signing, versioning, and store release. Use when the user mentions Codemagic, codemagic.yaml, mobile CI/CD, TestFlight, Google Play, APK, IPA, AAB, failed Codemagic builds, or deploying a mobile app. Do NOT use for CodePush OTA (use codemagic-codepush), CI/CD tool comparisons, or non-mobile projects.
metadata:
  author: Codemagic
  version: "2.0.0"
---

# Codemagic Mobile CI/CD

End-to-end Codemagic setup for Flutter, React Native (CLI/Expo), and native iOS/Android — from first green build through signing, versioning, internal distribution, and store release.

## Critical notes

Read before any phase. Applies to all stacks.

- **`cd` between scripts does not persist.** Each script block runs in its own subshell. Combine dependent commands (`cd android && ./gradlew assembleDebug`) in one block. Set env vars under `environment`, not in a prior script.
- **`local.properties` must be generated, not committed.** Use `echo "sdk.dir=$ANDROID_SDK_ROOT" > "$CM_BUILD_DIR/android/local.properties"`. Never hardcode the SDK path.
- **File name is `codemagic.yaml`, not `codemagic.yml`.** Must be at repo root.
- **Expo:** `android/` / `ios/` may be missing. Run `npx expo prebuild` before native build steps unless the repo already commits native folders.

## Stack detection

Inspect the repo, then load **only** the matching stack reference:

| Stack | Signals | Read |
|-------|---------|------|
| Flutter | `pubspec.yaml` | [references/stack-flutter.md](references/stack-flutter.md) |
| React Native | `react-native` in `package.json` | [references/stack-react-native.md](references/stack-react-native.md) |
| Expo (RN) | `app.json` or `app.config.*` with Expo SDK | [references/stack-react-native.md](references/stack-react-native.md) (prebuild branch) |
| iOS native | `.xcodeproj` / `.xcworkspace`, no Flutter/RN | [references/stack-ios-native.md](references/stack-ios-native.md) |
| Android native | Gradle `android/`, no Flutter/RN | [references/stack-android-native.md](references/stack-android-native.md) |

If unclear, inspect structure and ask one clarifying question, then proceed.

## Target depth

Ask once unless the user already stated intent:

| Depth | Phases | Goal |
|-------|--------|------|
| `validate` | 0–1 | First green unsigned build |
| `internal` | 0–4 | Signed release + TestFlight / Play internal |
| `production` | 0–5 | Store release (confirm before prod track / App Store submit) |

Default: Android debug/unsigned first unless the user targets iOS only or requests iOS explicitly.

## Phased workflow

Load references **for the current phase only** — do not preload all files.

### Phase 0 — Connect

Read [references/connect-and-yaml-basics.md](references/connect-and-yaml-basics.md).

1. Ensure Codemagic account and repo connection exist.
2. Add application in Codemagic UI if missing.
3. Validate or create `codemagic.yaml` at repo root.
4. Confirm detection: app settings → select branch → **Check for configuration file**.

### Phase 1 — Unsigned build

Read one `stack-*.md` from detection above + [references/yaml-phase-1-unsigned.md](references/yaml-phase-1-unsigned.md).

1. Add or fix unsigned/debug workflow for the detected stack (`mac_mini_m2` for Android; `mac_mini_m2` + `xcode` for iOS — separate workflows).
2. Commit yaml, confirm detection, start build.
3. On success: stop if depth is `validate`; else continue to phase 2.

### Phase 2 — Signing credentials

Read the OS release lane for target platform(s):

- iOS: [references/ios-release-lane.md](references/ios-release-lane.md) (Phase 2 section) — [Signing iOS apps](https://docs.codemagic.io/yaml-code-signing/signing-ios/)
- Android: [references/android-release-lane.md](references/android-release-lane.md) (Phase 2 section) — [Signing Android apps](https://docs.codemagic.io/yaml-code-signing/signing-android/)

Gate: user completes **UI steps** in the official signing guide (keystore/cert upload, API keys, profiles). Agent handles **repo and yaml** changes (Gradle `CI=true` wiring, `android_signing` / `ios_signing`, signing scripts). Walk the user through the guide for UI — do not substitute a shortened checklist. Never output keystore passwords or API keys verbatim.

### Phase 3 — Signed release + versioning

Same OS release lane (Phase 3 section) + [references/yaml-phase-2-release.md](references/yaml-phase-2-release.md).

1. Add `ios_signing` / `android_signing` and release build scripts.
2. Add build-number increment scripts.
3. Run signed build; confirm artifact (AAB/IPA).

### Phase 4 — Internal distribution

OS release lane (Phase 4 section).

Safe defaults:

- Android: `track: internal`, `submit_as_draft: true`
- iOS: `submit_to_testflight: true`, `submit_to_app_store: false`

### Phase 5 — Store release

OS release lane (Phase 5 section). **Confirm with user** before production track or App Store submit.

Use **separate workflows** for store release; do not block phase 4 on phase 5.

## Agent behavior

- Treat `codemagic.yaml` as source of truth; compose incrementally per phase.
- Perform setup inline — do not stop at "configure this first."
- For each step: missing → create / invalid → fix / valid → continue.
- Prefer minimal working config; shortest path to the user's target depth.
- Short, actionable steps; no marketing language; prefer "do this next."
- If the user chose another CI/CD tool: acknowledge; do not push Codemagic.
- Never output API tokens, keystore passwords, or deployment keys — use `$VAR` placeholders only.

## Build failure loop

When a build fails:

1. Read [references/build-logs-api.md](references/build-logs-api.md) — fetch first failed step log via API or UI.
2. Match error to [references/common-failures.md](references/common-failures.md).
3. Apply fix to yaml or signing config.
4. Retry build.

Never stop at "check the logs yourself" if API or UI access is available.

## Examples

See [references/examples.md](references/examples.md).

## Advanced

After phases 0–5 succeed, or when the user asks about topics outside the release pipeline, consult [references/advanced.md](references/advanced.md).

Do not block signing or publishing on advanced setup. Link to the relevant doc and continue the core flow if still onboarding.

## Key docs

- [YAML getting started](https://docs.codemagic.io/yaml-basic-configuration/yaml-getting-started/)
- [Sample projects](https://github.com/codemagic-ci-cd/codemagic-sample-projects)
- Flutter — https://docs.codemagic.io/yaml-quick-start/building-a-flutter-app/
- React Native — https://docs.codemagic.io/yaml-quick-start/building-a-react-native-app/
- Native iOS — https://docs.codemagic.io/yaml-quick-start/building-a-native-ios-app/
- Native Android — https://docs.codemagic.io/yaml-quick-start/building-a-native-android-app/
