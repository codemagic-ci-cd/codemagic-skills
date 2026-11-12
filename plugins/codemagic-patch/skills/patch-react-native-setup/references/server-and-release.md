# Server side and releases with `cmpatch`

The CLI does the server-side work: apps, deployment keys, server values, releases and diagnostics. Install it with `npm install -g @codemagic/patch-cli` (Node.js 20.19+ or 22.12+). Exact flags: `cmpatch <command> --help` and the [CLI reference](https://patch.codemagic.io/docs/reference/cli-reference.md). Most commands take `--format json`, which is the form to read.

## Sign in

- `cmpatch login --server-url <api-url>` opens the browser. The user finishes the sign-in; then continue.
- Unattended, or with a token the user provides: `export CODEMAGIC_PATCH_TOKEN=…`, and pass `--server-url` (or `cmpatch config set server-url <url>`).
- `cmpatch whoami` confirms it.

## Create the apps and read the values the binary needs

One Patch app per platform; each new app gets `Staging` and `Production` deployments.

```bash
cmpatch app create --name <Name>-iOS
cmpatch app create --name <Name>-Android
cmpatch deployment list --app <Name>-iOS --format json       # deployment keys
cmpatch deployment list --app <Name>-Android --format json
```

Ask the user for the app names; `<Name>-iOS` / `<Name>-Android` is the docs' convention. Reuse existing apps when the user already has them (`cmpatch app list --format json`).

**API URL:** the server the user named. **Download base URL:** the server's exact `PUBLIC_BASE_URL`. Do not derive it from the API URL. Take it from the installer's output, from `cmpatch context --remote` once the project is linked (below), or from the deployment's SDK configuration panel in the dashboard.

## Link the project

`codemagic-patch.config.json` in the app root lets later commands omit `--server-url` and `--app`. Write it with:

```bash
cmpatch init --server-url <api-url> --ios-app <Name>-iOS --android-app <Name>-Android --deployment Staging --skip-wire --yes
```

- With flags, `init` only selects apps that already exist. Create them first, as above; otherwise it exits `2` with "app was not found".
- `--skip-wire` keeps `init` away from the app's code: you integrate the SDK yourself.
- Commit the file; it holds no secrets.
- `cmpatch context --remote` then shows the resolved server, team and project root, and the server's SDK configuration, including the download base URL.

## Publish a release

Release only once a binary with the SDK has been built and installed, and publish from the **same commit and native state** that binary was built from. The first release for a binary version records the native fingerprint. That fingerprint covers `ios/`, `android/`, committed lockfiles such as `ios/Podfile.lock`, autolinked native dependencies, the React Native version, and `.gitignore`. A later release whose fingerprint disagrees is blocked. So commit `Podfile.lock` after `pod install`, before the first release. Read [Fingerprinting](https://patch.codemagic.io/docs/using-patch/fingerprinting.md).

From the app root:

```bash
cmpatch release-react --platform ios --deployment Staging --dry-run
cmpatch release-react --platform ios --deployment Staging --release-notes "<what changed>" --yes
cmpatch release list --app <Name>-iOS --deployment Staging --format json
cmpatch release inspect --app <Name>-iOS --deployment Staging --label <label> --wait
```

Repeat for Android.

- `release-react` runs the project's bundler (Metro or Expo), resolves the target binary version, builds the bundle, and uploads it. Pass:
  - `--target-binary-version` when the version comes from flavors, build-type overrides, CI or computed Gradle code;
  - `--xcode-target-name` / `--build-configuration-name` when there are several app targets or configurations with different versions;
  - `--bundler` / `--entry-file` when detection fails.
- The target binary version must equal the installed app's `CFBundleShortVersionString` / `versionName` exactly.
- Devices receive a release only after the server has processed it: wait for `inspect --wait` to report `published`.
- On a fingerprint disagreement, stop and find out why. Pass `--allow-fingerprint-mismatch` only after the user confirms the release targets the binary they mean.
- For each store build, the docs recommend registering its fingerprint with a disabled release (`--disabled`) from that build's commit. Mention it when the user moves to Production.

## Afterwards

- `cmpatch release metrics --app <app> --deployment Staging --label <label>` shows downloads and installs per release.
- `cmpatch doctor` and `cmpatch doctor --verify-delivery` check setup and delivery: [troubleshooting.md](troubleshooting.md).
- Staging to Production, rollouts and rollbacks: [Releasing updates](https://patch.codemagic.io/docs/using-patch/releasing-updates.md), [Production control](https://patch.codemagic.io/docs/using-patch/production-control.md). Each needs the user's go-ahead.
