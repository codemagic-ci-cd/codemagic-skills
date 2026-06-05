# YAML Templates — Phase 2+ Release Blocks

Add these blocks incrementally to existing workflows. Do not replace phase 1 workflows unless upgrading the same workflow to signed release.

## Build machines

**Android workflows** — use `instance_type: mac_mini_m2` so free-tier users can run first builds (500 free Mac M2 minutes/month on personal accounts). Do not set `xcode` on Android-only workflows.

Optional optimization when billing is enabled — switch Android workflows to Linux:

```yaml
    instance_type: linux_x2
    environment:
      ubuntu: 24.04
```

See [Ubuntu 24.04 specs](https://docs.codemagic.io/specs-linux/ubuntu-24.04/).

**iOS workflows** — `instance_type: mac_mini_m2` and `xcode: latest`. Pin Xcode only if required — see [macOS specs](https://docs.codemagic.io/specs-macos/).

## Android signing block

```yaml
    environment:
      android_signing:
        - keystore_reference_name
```

`keystore_reference_name` must match Code signing identities in Codemagic UI.

Agent applies this Gradle `release` signing when `CI=true` (in `android/app/build.gradle` or project-equivalent path):

```groovy
signingConfigs {
    release {
        if (System.getenv()["CI"]) {
            storeFile file(System.getenv()["CM_KEYSTORE_PATH"])
            storePassword System.getenv()["CM_KEYSTORE_PASSWORD"]
            keyAlias System.getenv()["CM_KEY_ALIAS"]
            keyPassword System.getenv()["CM_KEY_PASSWORD"]
        }
    }
}
buildTypes {
    release {
        signingConfig signingConfigs.release
    }
}
```

## iOS signing block

Agent adds after user uploads credentials — see [ios-release-lane.md](ios-release-lane.md) Phase 2.

```yaml
    integrations:
      app_store_connect: your_api_key_name
    environment:
      ios_signing:
        distribution_type: app_store
        bundle_identifier: com.example.app
```

Before build scripts:

```yaml
      - name: Set up code signing
        script: xcode-project use-profiles
```

## Versioning snippets

See OS release lanes for full increment scripts. Android example:

```yaml
      - name: Build release bundle
        script: |
          LATEST=$(google-play get-latest-build-number --package-name "$PACKAGE_NAME")
          UPDATED=$((${LATEST:-0} + 1))
          cd android && ./gradlew bundleRelease -PversionCode=$UPDATED
```

iOS example:

```yaml
      - name: Increment build number
        script: |
          cd $CM_BUILD_DIR
          LATEST=$(app-store-connect get-latest-app-store-build-number "$APP_STORE_APPLE_ID")
          agvtool new-version -all $(($LATEST + 1))
```

Full guide: https://docs.codemagic.io/knowledge-codemagic/build-versioning/

## Publishing blocks

See [ios-release-lane.md](ios-release-lane.md) and [android-release-lane.md](android-release-lane.md) phases 4–5.
