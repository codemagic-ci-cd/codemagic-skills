# Codemagic Skills for AI Agents

Agent skills for Codemagic tasks, packaged as a plugin marketplace for Claude Code, Codex and Cursor. The skills also work with GitHub Copilot, Windsurf, and any agent that supports the [Agent Skills spec](https://agentskills.io/).

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

### Codex

```bash
codex plugin marketplace add codemagic-ci-cd/codemagic-skills
codex plugin add codemagic-patch@codemagic
```

### Cursor

In **Customize**, add a marketplace **From GitHub Repository** with `codemagic-ci-cd/codemagic-skills`, then install `codemagic-patch` (or another plugin) from it. Team admins can add the repository as a team marketplace in the Cursor dashboard instead.

### Any agent, with the skills CLI

```bash
npx skills add codemagic-ci-cd/codemagic-skills --skill <name>
```

For example, `--skill patch-react-native-setup` or `--skill codemagic-codepush`. Always name the skill. Without `--skill`, the [`skills` CLI](https://github.com/vercel-labs/skills) asks which of the 16 skills to install, and with `-y` it installs all of them.

### By hand

Copy a skill folder (`plugins/<plugin>/skills/<skill>/`) into your agent's skills directory, for example `.claude/skills/` or `.agents/skills/`.

## Moved from `skills/`

Skills used to live in `skills/<name>/`. They now live in `plugins/<plugin>/skills/<name>/`. For example, `skills/codemagic-codepush/` is now `plugins/codemagic-codepush/skills/codemagic-codepush/`.

- `npx skills update` finds a skill at its new path and rewrites `skills-lock.json` to match.
- Restoring an older `skills-lock.json` with `npx skills install` fails with "No matching skills found", because the lock still names the old path. Run `npx skills update codemagic-codepush` once, then commit the updated lock.
- The Shorebird skill is now `codemagic-shorebird-setup`. It was named `codemagic-codepush-setup` and could not be installed with the skills CLI.

## Layout

```text
.claude-plugin/marketplace.json      the marketplace "codemagic" (Claude Code; Codex reads it too)
.cursor-plugin/marketplace.json      the same marketplace for Cursor
plugins/<plugin>/.claude-plugin/plugin.json
plugins/<plugin>/.cursor-plugin/plugin.json
plugins/<plugin>/skills/<skill>/SKILL.md   (+ references/, agents/openai.yaml)
```

The Claude and Cursor manifests describe the same plugins. When you add a plugin or change a name, description or version, change both.

## License

[MIT](LICENSE). Use these however you want.
