# YAML Templates — Phase 1 Unsigned

Compose one workflow per target platform. Merge into existing `codemagic.yaml`; do not remove unrelated workflows.

## Rules (all stacks)

- File name: `codemagic.yaml` at repo root
- Set `max_build_duration: 30` (or 60 for iOS) to avoid hung builds consuming quota
- Android workflows: `instance_type: mac_mini_m2` (works on [free tier](https://docs.codemagic.io/billing/pricing/); omit `xcode`)
- iOS workflows: `instance_type: mac_mini_m2` and `xcode: latest`
- Use separate workflows for Android and iOS
- Optional later: switch Android to `linux_x2` + `ubuntu: 24.04` when [billing is enabled](https://docs.codemagic.io/yaml-basic-configuration/yaml-getting-started/#instance-type) — see [Ubuntu specs](https://docs.codemagic.io/specs-linux/ubuntu-24.04/)
- Combine `cd` and build commands in a single script block

## Stack-specific templates

Full yaml blocks live in the stack reference for your project:

| Stack | Reference |
|-------|-----------|
| Flutter | [stack-flutter.md](stack-flutter.md) |
| React Native / Expo | [stack-react-native.md](stack-react-native.md) |
| Native iOS | [stack-ios-native.md](stack-ios-native.md) |
| Native Android | [stack-android-native.md](stack-android-native.md) |

## After adding yaml

1. Commit and push
2. Codemagic UI → app settings → **Check for configuration file**
3. **Start new build** → select workflow → confirm artifact

On failure: [build-logs-api.md](build-logs-api.md) → [common-failures.md](common-failures.md)
