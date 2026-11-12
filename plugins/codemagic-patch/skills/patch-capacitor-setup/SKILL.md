---
name: patch-capacitor-setup
description: Add Codemagic Patch over-the-air (OTA) updates to a Capacitor or Ionic app (Angular, React, Vue, Nx or other monorepos, custom native shells) by understanding how the Patch plugin takes over the WebView and fitting it to the project's layout, then publish and verify the first update. Use when the user wants to set up Patch or self-hosted live updates in a Capacitor or Ionic app, configure the plugin per platform, publish or verify a web-asset release with cmpatch-capacitor, move from Appflow Live Updates or another live-update plugin to Patch, or find out why a Capacitor app does not receive a Patch update.
license: MIT
compatibility: Needs Node.js 20.19+ or 22.12+, Capacitor 7 or 8, network access to the user's Patch server and to patch.codemagic.io, Xcode for iOS (iOS 15+), and the Android SDK with JDK 21, NDK and CMake for Android.
---

# Codemagic Patch for Capacitor and Ionic

Codemagic Patch is a self-hosted OTA update service. The user runs the Patch server, and the app downloads new web assets from it and points Capacitor's WebView at them. Native code changes still need a store build.

You integrate the plugin into the app yourself, fitted to how the project is laid out. You use the `cmpatch-capacitor` CLI for the server side: apps, deployment keys, releases and diagnostics. For a React Native or Expo app, use the `patch-react-native-setup` skill instead.

## Before you edit anything

Read [references/how-it-works.md](references/how-it-works.md). Every integration step exists to satisfy one of its rules, and when a project does not look like a template you will work from those rules rather than from a snippet.

Commands, flags and snippets are maintained in the docs at https://patch.codemagic.io/docs/. Every page has a markdown twin at its URL plus `.md`, and https://patch.codemagic.io/docs/llms.txt indexes them. `cmpatch-capacitor <command> --help` describes the installed CLI exactly. When the docs or `--help` disagree with this skill, follow them.

## Ground rules

1. **Signing in stays with the person.** `cmpatch-capacitor login` opens a browser for OAuth consent. Start it, ask the user to finish it, then work from the stored session. With a token the user hands you, use `CODEMAGIC_PATCH_TOKEN`, not a command-line flag.
2. **Never print, commit or paste secrets**: personal access tokens (`cm_pat_…`), the server's `.env.selfhost`, OAuth client secrets, code-signing private keys. Deployment keys are not secrets; they are built into the app.
3. **Ask before**:
   - installing or upgrading a server;
   - publishing to any deployment other than Staging;
   - promoting or rolling back;
   - removing another live-update plugin or a custom native shell's server-path code;
   - changing release-build settings the user did not ask about.
4. **Staging first.** The first test binary carries the Staging deployment key. Production devices only get releases published to Production, and only once a binary with the Production key is installed.
5. **One Patch app per platform.** iOS and Android have separate apps and deployment keys. A shared key makes the platforms overwrite each other's releases.
6. **An OTA cannot add the plugin.** After integrating and `npx cap sync`, the user must build and install a new native binary before any OTA test. Live reload and `server.url` load from a dev server and never apply an update.
7. **Target the exact installed version.** Releases target iOS `CFBundleShortVersionString` / Android `versionName` exactly, and the CLI does not detect it.
8. **Preview, then act.** Use `--dry-run` before anything that changes what devices receive, and `--format json` when you need to read a result.

## Workflow

Copy this checklist and keep it updated:

```text
- [ ] 1. Map the workspace and plan the integration
- [ ] 2. Choose the Patch server
- [ ] 3. Create the Patch apps and collect keys and URLs
- [ ] 4. Integrate the plugin
- [ ] 5. Check the integration
- [ ] 6. Build and install a release binary
- [ ] 7. Publish the first release to Staging
- [ ] 8. Verify on the device
```

### 1. Map the workspace and plan the integration

Follow [references/workspace.md](references/workspace.md). Find:

- the app root and the workspace around it;
- the Capacitor version and config;
- `ios.path` / `android.path`, `webDir` and the web build command;
- the framework and its first screen;
- anything else that sets Capacitor's server path;
- native overrides, versions, and where release builds run.

Show the user the workspace map and the plan before you change files. Ask which app to set up when there is more than one. Stop to plan with the user when another live-update plugin is active; for Appflow, see [Migrating from Appflow](https://patch.codemagic.io/docs/migration/migrating-from-appflow.md).

### 2. Choose the Patch server

The app's config holds two server values: the API URL and the download base URL. Ask the user which server the app should use.

| Situation | Command (the user confirms first) | Read |
| --- | --- | --- |
| They already run Patch | none; ask for the API URL | [Install](https://patch.codemagic.io/docs/install.md) |
| Try it on a simulator or emulator | `cmpatch selfhost local-eval` (needs Docker) | [Local quickstart](https://patch.codemagic.io/docs/local-quickstart.md) |
| Try it on a real phone without a domain | `cmpatch selfhost install --ngrok` | [Local quickstart](https://patch.codemagic.io/docs/local-quickstart.md) |
| Production on their own server and domain | `cmpatch selfhost install` | [Install](https://patch.codemagic.io/docs/install.md) |
| Production without a server or domain | `cmpatch selfhost install --fly` | [Install](https://patch.codemagic.io/docs/install.md) |

Servers are installed with the React Native CLI, `cmpatch` (`npm install -g @codemagic/patch-cli`). The server commands are wizards that create infrastructure and OAuth apps in the user's accounts; let the user answer them.

### 3. Create the Patch apps and collect keys and URLs

Follow [references/server-and-release.md](references/server-and-release.md):

- install `cmpatch-capacitor` and sign in;
- create one app per platform, or reuse the user's;
- read the Staging deployment keys and the download base URL.

### 4. Integrate the plugin

Follow [references/integration.md](references/integration.md), in the shape your plan chose:

1. Add the plugin to the app package and run `npx cap sync`.
2. Add a `plugins.CodemagicPatch` block per platform, or native overrides per scheme or flavor.
3. Check the synced configs.
4. Call `start()` after the first screen.
5. Clear anything else that sets the server path.
6. Meet the native requirements.
7. Make sure the web build puts `index.html` at the top of `webDir`.

### 5. Check the integration

Go through the checklist at the end of [references/integration.md](references/integration.md), then run `cmpatch-capacitor doctor` from the app root.

### 6. Build and install a release binary

Build the web app, `npx cap sync`, then build a release native binary without live reload, and install it on a simulator, emulator or device. Ask the user to do this if you cannot run the build or reach the device. For a plain-HTTP test server, see section 10 of [references/integration.md](references/integration.md). Note the installed versions per platform.

### 7. Publish the first release to Staging

Make a small visible change, build the web app, then publish one release per platform, dry run first: [references/server-and-release.md](references/server-and-release.md). Wait for `cmpatch-capacitor release inspect … --wait` before testing.

### 8. Verify on the device

Cold-start the app, close it, and cold-start it again. The first launch downloads the update; the second runs it. The change should be visible, and `cmpatch-capacitor release metrics` should count the device. If you cannot see the device, ask the user what the app shows. See [Verify a test release](https://patch.codemagic.io/docs/using-patch/verify-test-release.md).

Report what was set up: the server, the app names, which key each binary carries, the files you changed, the release labels, and what remains (for example Production builds, or reverting a cleartext setting).

## When the update does not arrive

Run `cmpatch-capacitor doctor`, and after publishing `cmpatch-capacitor doctor --verify-delivery`, from the app root. Then map the symptom with [references/troubleshooting.md](references/troubleshooting.md).

## After the first release

Point the user to the page for what they want next, and do not do these without being asked:

- Production builds with the Production key: "Build for different deployments" in [Native setup (Capacitor)](https://patch.codemagic.io/docs/setup/native-setup-capacitor.md)
- Staging to Production: [Releasing updates](https://patch.codemagic.io/docs/using-patch/releasing-updates.md), [Preparing for production](https://patch.codemagic.io/docs/using-patch/preparing-for-production.md)
- Rollouts, mandatory updates, rollbacks: [Production control](https://patch.codemagic.io/docs/using-patch/production-control.md)
- Publishing from CI: [CI integration](https://patch.codemagic.io/docs/using-patch/ci-integration.md)
- Code signing: [Security](https://patch.codemagic.io/docs/using-patch/security.md)
