# codemagic-codepush

An agent skill for [Codemagic-hosted CodePush](https://docs.codemagic.io/rn-codepush/concepts/) (`codepush.pro`), OTA updates for React Native apps on iOS and Android.

| Skill | For |
| --- | --- |
| [`codemagic-codepush`](skills/codemagic-codepush/SKILL.md) | Integrating `@code-push-next/react-native-code-push` with native iOS and Android wiring, CodePush CLI apps and deployments, Codemagic CI release workflows, Staging-to-Production promotion, rollbacks, security, and migrating from App Center |

The skill works from Codemagic's CodePush documentation and keeps deployment keys and access tokens in environment variables or placeholders, never in its output.

For self-hosted OTA updates, see the `codemagic-patch` plugin in this marketplace.

## Install

In Claude Code 2.1.275 or later:

```text
/plugin install codemagic-codepush --marketplace codemagic-ci-cd/codemagic-skills
```

In earlier versions, add the marketplace first:

```text
/plugin marketplace add codemagic-ci-cd/codemagic-skills
/plugin install codemagic-codepush@codemagic
```

With the [`skills` CLI](https://github.com/vercel-labs/skills), for any agent:

```bash
npx skills add codemagic-ci-cd/codemagic-skills --skill codemagic-codepush
```
