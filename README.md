# Codemagic Skills for AI Agents

Agent skills for Codemagic tasks, packaged as a plugin marketplace. They work with Claude Code, OpenAI Codex, Cursor, GitHub Copilot, Windsurf, and any agent that supports the [Agent Skills spec](https://agentskills.io/).

## Plugins

| Plugin | Skills | For |
| --- | --- | --- |
| [`codemagic-ci-cd`](plugins/codemagic-ci-cd/) | 13, from [`codemagic-ci-cd-onboarding`](plugins/codemagic-ci-cd/skills/codemagic-ci-cd-onboarding/SKILL.md) to [`codemagic-publishing-agent`](plugins/codemagic-ci-cd/skills/codemagic-publishing-agent/SKILL.md) | Codemagic CI/CD for mobile apps: first build, `codemagic.yaml`, code signing, triggers, caching, testing, notifications, publishing and Shorebird |
| [`codemagic-codepush`](plugins/codemagic-codepush/) | [`codemagic-codepush`](plugins/codemagic-codepush/skills/codemagic-codepush/SKILL.md) | OTA updates for React Native apps with Codemagic-hosted CodePush (`codepush.pro`) |
| [`codemagic-patch`](plugins/codemagic-patch/) | [`patch-react-native-setup`](plugins/codemagic-patch/skills/patch-react-native-setup/SKILL.md), [`patch-capacitor-setup`](plugins/codemagic-patch/skills/patch-capacitor-setup/SKILL.md) | Self-hosted OTA updates with [Codemagic Patch](https://patch.codemagic.io/docs/) for React Native, Expo, Capacitor and Ionic apps |

Each plugin's README lists its skills.

## Install

### Claude Code

```text
/plugin marketplace add codemagic-ci-cd/codemagic-skills
/plugin install codemagic-patch@codemagic
```

Install `codemagic-ci-cd@codemagic` or `codemagic-codepush@codemagic` the same way.

### Any agent, with the skills CLI

```bash
npx skills add codemagic-ci-cd/codemagic-skills
```

The [`skills` CLI](https://github.com/vercel-labs/skills) asks which skills and agents to install. Pick one with `--skill <name>`, for example `--skill patch-react-native-setup`.

### By hand

Copy a skill folder (`plugins/<plugin>/skills/<skill>/`) into your agent's skills directory, for example `.claude/skills/` or `.agents/skills/`.

## Layout

```text
.claude-plugin/marketplace.json      the marketplace "codemagic", listing the three plugins
plugins/<plugin>/.claude-plugin/plugin.json
plugins/<plugin>/skills/<skill>/SKILL.md   (+ references/, agents/openai.yaml)
```

## License

[MIT](LICENSE). Use these however you want.
