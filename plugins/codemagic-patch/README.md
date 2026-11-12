# codemagic-patch

Agent skills for [Codemagic Patch](https://patch.codemagic.io/docs/), self-hosted over-the-air (OTA) updates for React Native and Capacitor apps.

The agent first maps how your project is laid out: a single app or a monorepo, generated or hand-maintained native projects, the package manager, the web build, and how versions and release builds are set. It then integrates the Patch SDK to fit, based on how the SDK chooses bundles, confirms them and rolls them back. Finally it publishes a first update to Staging, checks that the device receives it, and diagnoses why when it does not.

| Skill | For |
| --- | --- |
| [`patch-react-native-setup`](skills/patch-react-native-setup/SKILL.md) | React Native 0.73+ and Expo SDK 52+: bare, Expo prebuild, maintained native projects, SwiftPM mode, monorepos |
| [`patch-capacitor-setup`](skills/patch-capacitor-setup/SKILL.md) | Capacitor 7 and 8, Ionic (Angular, React, Vue), Nx and other monorepos, custom native shells |

Signing in, installing a server, and anything that reaches users beyond Staging stay with you. Guide: [Set up with a coding agent](https://patch.codemagic.io/docs/agent-skill).

## Install

In Claude Code 2.1.275 or later:

```text
/plugin install codemagic-patch --marketplace codemagic-ci-cd/codemagic-skills
```

In earlier versions, add the marketplace first:

```text
/plugin marketplace add codemagic-ci-cd/codemagic-skills
/plugin install codemagic-patch@codemagic
```

With the [`skills` CLI](https://github.com/vercel-labs/skills), for any agent:

```bash
npx skills add codemagic-ci-cd/codemagic-skills --skill patch-react-native-setup   # or patch-capacitor-setup
```
