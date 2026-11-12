---
name: patch-react-native-setup
description: Add Codemagic Patch over-the-air (OTA) updates to a React Native or Expo app by understanding how the Patch SDK works and fitting it to the project's layout (single app or monorepo, Expo prebuild or maintained native projects, CocoaPods or SwiftPM, any React Native version from 0.73), then publish and verify the first update. Use when the user wants to set up Patch or self-hosted OTA updates in a React Native or Expo app, connect it to a Patch server, wire the SDK and native iOS/Android config, ship or verify a first OTA release, move from CodePush or Expo Updates to Patch, or find out why a React Native app does not receive a Patch update.
license: MIT
compatibility: Needs Node.js 20.19+ or 22.12+, network access to the user's Patch server and to patch.codemagic.io, and Xcode or the Android SDK (with NDK and CMake) to build the app.
---

# Codemagic Patch for React Native and Expo

Codemagic Patch is a self-hosted OTA update service. The user runs the Patch server, and the app's release build downloads JavaScript bundle updates from it. Native code changes still need a store build.

You integrate the SDK into the app yourself, fitted to how the project is laid out. You use the `cmpatch` CLI for the server side: apps, deployment keys, releases and diagnostics. For a Capacitor or Ionic app, use the `patch-capacitor-setup` skill instead.

## Before you edit anything

Read [references/how-it-works.md](references/how-it-works.md). Every integration step exists to satisfy one of its rules, and when a project does not look like a template you will work from those rules rather than from a snippet.

Commands, flags and snippets are maintained in the docs at https://patch.codemagic.io/docs/. Every page has a markdown twin at its URL plus `.md`, and https://patch.codemagic.io/docs/llms.txt indexes them. `cmpatch <command> --help` describes the installed CLI exactly. When the docs or `--help` disagree with this skill, follow them.

## Ground rules

1. **Signing in stays with the person.** `cmpatch login` opens a browser for OAuth consent. Start it, ask the user to finish it, then work from the stored session. With a token the user hands you, use `CODEMAGIC_PATCH_TOKEN`, not a command-line flag.
2. **Never print, commit or paste secrets**: personal access tokens (`cm_pat_…`), the server's `.env.selfhost`, OAuth client secrets, code-signing private keys. Deployment keys are not secrets; they are built into the app.
3. **Ask before** installing or upgrading a server, publishing to any deployment other than Staging, promoting or rolling back, running `npx expo prebuild --clean`, removing another OTA library, or changing release-build settings the user did not ask about. Test-only changes the skill calls for (such as allowing cleartext for a local server) are fine once you have told the user, and they go in your report as something to revert.
4. **Staging first.** The first test binary carries the Staging deployment key. Production devices only get releases published to Production, and only once a binary with the Production key is installed.
5. **One Patch app per platform.** iOS and Android have separate apps and deployment keys. Never reuse a key across platforms.
6. **An OTA cannot add the SDK.** After integrating, the user must build and install a new release binary before any OTA test. Debug builds load JavaScript from Metro and never apply an update.
7. **Publish with the native state that was built.** The first release for a deployment and binary version records its native fingerprint. Commit `Podfile.lock` and whatever else `pod install` changed before the build, then change only JavaScript between the build and the release.
8. **Preview, then act.** Use `--dry-run` before anything that changes what devices receive, and `--format json` when you need to read a result.

## Workflow

Copy this checklist and keep it updated:

```text
- [ ] 1. Map the workspace and plan the integration
- [ ] 2. Choose the Patch server
- [ ] 3. Create the Patch apps and collect keys and URLs
- [ ] 4. Integrate the SDK
- [ ] 5. Check the integration
- [ ] 6. Build and install a release binary
- [ ] 7. Publish the first release to Staging
- [ ] 8. Verify on the device
```

### 1. Map the workspace and plan the integration

Follow [references/workspace.md](references/workspace.md). Find:

- the app root and the workspace around it;
- the package manager and how JavaScript is bundled;
- the React Native version and architecture;
- whether Expo generates the native projects;
- the iOS and Android host shapes, targets, variants and versions;
- the entry and root component;
- any other OTA system, and where release builds run.

Show the user the workspace map and the plan before you change files. Ask which app to set up when there is more than one. Stop to plan with the user when another OTA system is active. Migration guides: [CodePush](https://patch.codemagic.io/docs/migration/migrating-from-codepush.md), [Expo Updates](https://patch.codemagic.io/docs/migration/migrating-from-expo-updates.md).

### 2. Choose the Patch server

The binary holds two server values: the API URL and the download base URL. Ask the user which server the app should use.

| Situation | Command (the user confirms first) | Read |
| --- | --- | --- |
| They already run Patch | none; ask for the API URL | [Install](https://patch.codemagic.io/docs/install.md) |
| Try it on a simulator or emulator | `cmpatch selfhost local-eval` (needs Docker) | [Local quickstart](https://patch.codemagic.io/docs/local-quickstart.md) |
| Try it on a real phone without a domain | `cmpatch selfhost install --ngrok` | [Local quickstart](https://patch.codemagic.io/docs/local-quickstart.md) |
| Production on their own server and domain | `cmpatch selfhost install` | [Install](https://patch.codemagic.io/docs/install.md) |
| Production without a server or domain | `cmpatch selfhost install --fly` | [Install](https://patch.codemagic.io/docs/install.md) |

Install the CLI first: `npm install -g @codemagic/patch-cli`. The server commands are wizards that create infrastructure and OAuth apps in the user's accounts; let the user answer them.

### 3. Create the Patch apps and collect keys and URLs

Follow [references/server-and-release.md](references/server-and-release.md):

- sign in;
- create one app per platform, or reuse the user's;
- read the Staging deployment keys and the download base URL;
- link the project with `cmpatch init --skip-wire`.

### 4. Integrate the SDK

Follow [references/integration.md](references/integration.md), in the shape your plan chose:

1. Add the SDK to the app package.
2. Put the keys and URLs into each binary: `Info.plist` and string resources, or the Expo config plugin.
3. Hook bundle selection in the iOS and Android hosts.
4. Confirm readiness from JavaScript: `Patch.wrap`, or `sync()` after initialization.
5. Install native dependencies, and set up Jest when the app has tests.

### 5. Check the integration

Go through the checklist at the end of [references/integration.md](references/integration.md), then run `cmpatch doctor` from the app root. When `ios/` and `android/` sit in the app root and there is one app target, `cmpatch wire --dry-run --diff` is a second opinion: it changes nothing and lists the edits it would make. Treat any edit it proposes that you did not make as a question to answer.

### 6. Build and install a release binary

Commit the integration, then build a release-configuration binary from that commit, with the Staging key, and install it on a simulator, emulator or device. Ask the user to do this if you cannot run the build or reach the device, and name the commit to build from. For a plain-HTTP test server, see section 8 of [references/integration.md](references/integration.md). Note the installed binary version (iOS `CFBundleShortVersionString`, Android `versionName`).

### 7. Publish the first release to Staging

After the binary is built, make a small visible JavaScript change, so the update is distinguishable from what the binary already contains. Then publish one release per platform from the app root, dry run first: [references/server-and-release.md](references/server-and-release.md). Check that the dry run compiled Hermes bytecode when the app uses Hermes. Wait for `cmpatch release inspect … --wait` to report `published`.

### 8. Verify on the device

Cold-start the app, close it, and cold-start it again. The first launch downloads the update; the second runs it. The change should be visible. Metrics are sent when the app next comes to the foreground, so background and reopen it once more before expecting `cmpatch release metrics` to count the device. If you cannot see the device, ask the user what the app shows. See [Verify a test release](https://patch.codemagic.io/docs/using-patch/verify-test-release.md).

Report what was set up: the server, the app names, which key each binary carries, the files you changed, the release labels, and what remains (for example Production builds, or reverting a cleartext setting).

## When the update does not arrive

Run `cmpatch doctor`, and after publishing `cmpatch doctor --verify-delivery`, from the app root. Then map the symptom with [references/troubleshooting.md](references/troubleshooting.md).

## After the first release

Point the user to the page for what they want next, and do not do these without being asked:

- Production builds with the Production key: "Build for different deployments" in [Native setup](https://patch.codemagic.io/docs/setup/native-setup.md)
- Staging to Production: [Releasing updates](https://patch.codemagic.io/docs/using-patch/releasing-updates.md), [Preparing for production](https://patch.codemagic.io/docs/using-patch/preparing-for-production.md)
- Rollouts, mandatory updates, rollbacks: [Production control](https://patch.codemagic.io/docs/using-patch/production-control.md)
- Publishing from CI: [CI integration](https://patch.codemagic.io/docs/using-patch/ci-integration.md)
- Code signing: [Security](https://patch.codemagic.io/docs/using-patch/security.md)
