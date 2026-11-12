# Server side and releases with `cmpatch-capacitor`

Capacitor apps use their own CLI, `cmpatch-capacitor` (`npm install -g @codemagic/capacitor-patch-cli`, Node.js 20.19+ or 22.12+). It is a 0.x release; check `cmpatch-capacitor <command> --help` and the `cmpatch-capacitor` section of the [CLI reference](https://patch.codemagic.io/docs/reference/cli-reference.md). Use `cmpatch` (the React Native CLI) only to run a server (`cmpatch selfhost …`); the two CLIs keep separate sign-ins.

## No project file

There is no `init`, `context` or `codemagic-patch.config.json`. Every command takes:

- **the server**: `--server-url`, `CODEMAGIC_PATCH_SERVER_URL`, or a stored default (`cmpatch-capacitor config set --server-url <url>`). `login` does not store one.
- **the app and deployment**: `--app <name>` (looks up Capacitor apps only) and `--deployment <name>`, or `--app-id` / `--deployment-id` / `--release-id`, which skip the lookup and suit CI.

Output is JSON when stdout is piped, or with `--format json`. Commands that change what devices receive prompt in a terminal. With `--format json`, without a TTY, or with `CI` set, they fail unless you pass `--yes`.

## Sign in

- `cmpatch-capacitor login --server-url <api-url>` opens the browser. The user finishes the sign-in. `--no-browser` prints the URL instead.
- Unattended, or with a token the user provides: `CODEMAGIC_PATCH_TOKEN`.
- `cmpatch-capacitor whoami` confirms it.

## Create the apps and read the values the binary needs

```bash
cmpatch-capacitor app create --name <Name>-iOS
cmpatch-capacitor app create --name <Name>-Android
cmpatch-capacitor deployment list --app <Name>-iOS --format json       # deployment keys
cmpatch-capacitor deployment list --app <Name>-Android --format json
```

- Each new app gets `Staging` and `Production` deployments.
- Apps created as React Native apps (for example with `cmpatch init`) do not show up here.
- **Download base URL:** the server's exact `PUBLIC_BASE_URL`; do not derive it from the API URL. Take it from the installer's output or the deployment's SDK configuration panel in the dashboard. `cmpatch-capacitor` has no command that prints it. `cmpatch context --remote` (the React Native CLI) does, but it needs its own `cmpatch login`, or `CODEMAGIC_PATCH_TOKEN` and a server URL.

## Publish a release

Build the web app first; the CLI does not run the build. Then, from the app root:

```bash
cmpatch-capacitor release create --bundle-path <webDir> --app <Name>-iOS --deployment Staging \
  --target-binary-version <installed iOS version> --dry-run
cmpatch-capacitor release create --bundle-path <webDir> --app <Name>-iOS --deployment Staging \
  --target-binary-version <installed iOS version> --release-notes "<what changed>" --yes
cmpatch-capacitor release inspect --app <Name>-iOS --deployment Staging --label <label> --wait
```

Repeat with the Android app and the installed Android version.

- **Code signing:** when the app enforces it (`publicKey` in the config, or the app's `--require-code-signing` setting), add `--private-key-path <pem>`. Devices reject unsigned releases then.
- **`--bundle-path`** is the `webDir` directory (or a ZIP of its contents) with `index.html` at its top level. The CLI checks this, and refuses a top-level `node_modules`, symlinks and over-long paths.
- **`--target-binary-version`** is required and must equal the installed app's `CFBundleShortVersionString` / `versionName` exactly. Ranges are rejected, and a release cannot be retargeted later. See [Binary version compatibility](https://patch.codemagic.io/docs/using-patch/binary-version-capacitor.md).
- Devices receive a release only after processing: wait for `inspect --wait` to report it as processed.
- **Exit codes:**
  - `2`: usage error (including a bundle the CLI refuses), missing `--yes`, or not signed in;
  - `3`: an invalid `--target-binary-version`, processing failed or timed out, or the stored sign-in expired;
  - `1`: server or runtime error, a declined prompt, or a failed doctor check;
  - `4`: account disabled.

## Afterwards

- `cmpatch-capacitor release metrics --app <app> --deployment Staging --label <label>` shows adoption per release.
- `cmpatch-capacitor doctor` and `--verify-delivery`: see [troubleshooting.md](troubleshooting.md).
- Promotion, rollout, rollback: [Releasing updates](https://patch.codemagic.io/docs/using-patch/releasing-updates.md), [Production control](https://patch.codemagic.io/docs/using-patch/production-control.md). Each needs the user's go-ahead.
