# codemagic-ci-cd

Agent skills for [Codemagic CI/CD](https://docs.codemagic.io/): getting a Flutter, React Native, native iOS or native Android app from zero to a signed, published build, and keeping the pipeline healthy.

| Skill | For |
| --- | --- |
| [`codemagic-ci-cd-onboarding`](skills/codemagic-ci-cd-onboarding/SKILL.md) | The first successful mobile build (APK/IPA) for Flutter or React Native |
| [`codemagic-yaml-quickstart`](skills/codemagic-yaml-quickstart/SKILL.md) | A starter `codemagic.yaml` for any framework and platform |
| [`codemagic-ios-signing-agent`](skills/codemagic-ios-signing-agent/SKILL.md) | Setting up iOS code signing |
| [`ios-code-signing`](skills/ios-code-signing/SKILL.md) | Deep-dive reference for iOS code signing errors |
| [`codemagic-android-signing-agent`](skills/codemagic-android-signing-agent/SKILL.md) | Setting up Android code signing |
| [`android-code-signing`](skills/android-code-signing/SKILL.md) | Deep-dive reference for Android code signing errors |
| [`codemagic-env-variables`](skills/codemagic-env-variables/SKILL.md) | Environment variables and variable groups |
| [`codemagic-build-triggers`](skills/codemagic-build-triggers/SKILL.md) | Push, pull request, tag and scheduled build triggers |
| [`codemagic-caching`](skills/codemagic-caching/SKILL.md) | Dependency caching |
| [`codemagic-testing`](skills/codemagic-testing/SKILL.md) | Unit, integration and UI tests in the pipeline |
| [`codemagic-notifications`](skills/codemagic-notifications/SKILL.md) | Email, Slack, Teams and Firebase App Distribution notifications |
| [`codemagic-publishing-agent`](skills/codemagic-publishing-agent/SKILL.md) | Publishing to App Store Connect, TestFlight, Google Play, Firebase and Huawei AppGallery |
| [`codemagic-shorebird-setup`](skills/codemagic-shorebird-setup/SKILL.md) | Shorebird code push for Flutter on Codemagic |

## Install

In Claude Code 2.1.275 or later:

```text
/plugin install codemagic-ci-cd --marketplace codemagic-ci-cd/codemagic-skills
```

In earlier versions, add the marketplace first:

```text
/plugin marketplace add codemagic-ci-cd/codemagic-skills
/plugin install codemagic-ci-cd@codemagic
```

In Codex:

```bash
codex plugin marketplace add codemagic-ci-cd/codemagic-skills
codex plugin add codemagic-ci-cd@codemagic
```

In Cursor, add `codemagic-ci-cd/codemagic-skills` in **Customize** as a marketplace **From GitHub Repository**, then install `codemagic-ci-cd`.

With the [`skills` CLI](https://github.com/vercel-labs/skills), for any agent:

```bash
npx skills add codemagic-ci-cd/codemagic-skills --skill codemagic-ci-cd-onboarding   # or any skill listed above
```
