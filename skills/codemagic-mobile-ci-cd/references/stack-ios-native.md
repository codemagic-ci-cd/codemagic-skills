# Native iOS Stack (Phase 1+)

Published quickstart: https://docs.codemagic.io/yaml-quick-start/building-a-native-ios-app/

## Phase 1 — Unsigned build

Set `XCODE_WORKSPACE` and `XCODE_SCHEME` to match the project. If workspace is under `ios/`, adjust paths accordingly.

```yaml
workflows:
  ios-native-unsigned:
    name: Native iOS unsigned
    instance_type: mac_mini_m2
    max_build_duration: 60
    environment:
      xcode: latest
      cocoapods: default
      vars:
        XCODE_WORKSPACE: "YourApp.xcworkspace"
        XCODE_SCHEME: "YourApp"
    scripts:
      - name: Install CocoaPods
        script: pod install
      - name: Build without signing
        script: |
          xcodebuild \
            -workspace "$CM_BUILD_DIR/$XCODE_WORKSPACE" \
            -scheme "$XCODE_SCHEME" \
            -configuration Debug \
            -destination 'generic/platform=iOS' \
            CODE_SIGNING_ALLOWED=NO \
            build
    artifacts:
      - $HOME/Library/Developer/Xcode/DerivedData/**/Build/**/*.app
      - /tmp/xcodebuild_logs/*.log
```

## Phase 3+ — Signed release

Use [ios-release-lane.md](ios-release-lane.md) — `xcode-project use-profiles` then `xcode-project build-ipa`.

## Notes

- Workspace at repo root vs `ios/` — adjust `pod install` and `-workspace` path
- No `--project` fallback if no workspace: use `-project "App.xcodeproj"` instead

See [common-failures.md](common-failures.md).
