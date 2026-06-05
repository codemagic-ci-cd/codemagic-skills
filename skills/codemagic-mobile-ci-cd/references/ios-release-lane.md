# iOS Release Lane (Phases 2–5)

Signing: https://docs.codemagic.io/yaml-code-signing/signing-ios/
Publishing: https://docs.codemagic.io/yaml-publishing/app-store-connect/
Versioning: https://docs.codemagic.io/knowledge-codemagic/build-versioning/

## Phase 2 — Signing credentials

iOS signing mixes **manual UI steps** (user) with **yaml changes** (agent).

### User — follow the official guide

Direct the user through UI steps only — do not skip or summarize:

**[Signing iOS apps](https://docs.codemagic.io/yaml-code-signing/signing-ios/)**

Walk through **Managing and uploading files** (App Store Connect API key, certificate, provisioning profile).

Gate: do not add `ios_signing` to yaml or run signed builds until credentials are uploaded in Codemagic.

### Agent — yaml and signing scripts

After the user confirms credentials are in Codemagic, add signing to yaml per [Referencing certificates and profiles](https://docs.codemagic.io/yaml-code-signing/signing-ios/#referencing-certificates-and-profiles-in-codemagicyaml):

```yaml
    integrations:
      app_store_connect: your_api_key_name
    environment:
      ios_signing:
        distribution_type: app_store
        bundle_identifier: com.example.app
      vars:
        BUNDLE_ID: "com.example.app"
        XCODE_WORKSPACE: "YourApp.xcworkspace"
        XCODE_SCHEME: "YourApp"
```

For TestFlight internal-only builds, use this signing script:

```yaml
      - name: Set up code signing
        script: xcode-project use-profiles --custom-export-options='{"testFlightInternalTestingOnly": true}'
```

## Phase 3 — Signed release + versioning

Add before build:

```yaml
      - name: Set up code signing
        script: xcode-project use-profiles
      - name: Increment build number
        script: |
          cd $CM_BUILD_DIR
          LATEST=$(app-store-connect get-latest-app-store-build-number "$APP_STORE_APPLE_ID")
          agvtool new-version -all $(($LATEST + 1))
```

Build (native RN):

```yaml
      - name: Build IPA
        script: |
          xcode-project build-ipa \
            --workspace "$CM_BUILD_DIR/ios/$XCODE_WORKSPACE" \
            --scheme "$XCODE_SCHEME"
```

Build (Flutter):

```yaml
      - name: Build IPA
        script: flutter build ipa --export-options-plist=/Users/builder/export_options.plist
```

Artifacts:

```yaml
    artifacts:
      - build/ios/ipa/*.ipa
      - /tmp/xcodebuild_logs/*.log
```

iOS workflows use `instance_type: mac_mini_m2` and `xcode: latest` — see stack reference templates. Pin Xcode only if required: https://docs.codemagic.io/specs-macos/

## Phase 4 — Internal distribution (TestFlight)

Safe defaults — do not submit to App Store without user confirmation:

```yaml
    publishing:
      app_store_connect:
        auth: integration
        submit_to_testflight: true
        submit_to_app_store: false
        beta_groups:
          - Internal testers
```

Post-processing runs after workflow completes. Check build log or email for App Store Connect status.

## Phase 5 — Store release

**Confirm with user** before enabling. Use a **separate workflow**:

```yaml
    publishing:
      app_store_connect:
        auth: integration
        submit_to_app_store: true
```

First App Store version often requires manual upload and metadata in App Store Connect.

## Common iOS signing issues

- Run `xcode-project use-profiles` **before** `build-ipa` or `flutter build ipa`
- `distribution_type` and bundle ID must match provisioning profile
- Flutter may require `--export-options-plist=/Users/builder/export_options.plist`

Details: https://docs.codemagic.io/troubleshooting/common-ios-issues/

See also [common-failures.md](common-failures.md).
