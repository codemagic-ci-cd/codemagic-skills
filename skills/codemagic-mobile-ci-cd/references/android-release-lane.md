# Android Release Lane (Phases 2–5)

Android workflows use `instance_type: mac_mini_m2` by default (free-tier friendly). Keep Android and iOS in separate workflows. Switch to `linux_x2` when billing is enabled — see [yaml-phase-2-release.md](yaml-phase-2-release.md).

Signing: https://docs.codemagic.io/yaml-code-signing/signing-android/
Publishing: https://docs.codemagic.io/yaml-publishing/google-play/
Versioning: https://docs.codemagic.io/knowledge-codemagic/build-versioning/

## Phase 2 — Signing credentials

Android signing mixes **manual UI steps** (user) with **repo/yaml changes** (agent).

### User — follow the official guide

Direct the user through UI steps only — do not skip or summarize:

**[Signing Android apps](https://docs.codemagic.io/yaml-code-signing/signing-android/)**

Walk through **Managing and uploading files** (generate keystore if needed, upload to Code signing identities).

Gate: do not add `android_signing` to yaml or run release builds until the keystore is uploaded.

### Agent — Gradle and yaml

After the user confirms the keystore is in Codemagic:

1. Wire `build.gradle` release signing to `CM_KEYSTORE_*` when `CI=true` — see [yaml-phase-2-release.md](yaml-phase-2-release.md) or the [Gradle section](https://docs.codemagic.io/yaml-code-signing/signing-android/#signing-android-apps-using-gradle) of the signing guide
2. Add `android_signing` to yaml:

```yaml
    environment:
      android_signing:
        - keystore_reference_name
      vars:
        PACKAGE_NAME: "com.example.app"
```

## Phase 3 — Signed release + versioning

```yaml
      - name: Build release AAB
        script: |
          LATEST=$(google-play get-latest-build-number --package-name "$PACKAGE_NAME" 2>/dev/null || true)
          if [ -z "$LATEST" ]; then
            UPDATED=$BUILD_NUMBER
          else
            UPDATED=$(($LATEST + 1))
          fi
          cd android && ./gradlew bundleRelease -PversionCode=$UPDATED
```

Artifacts:

```yaml
    artifacts:
      - android/app/build/outputs/**/*.aab
```

For Flutter: `flutter build appbundle` after signing is configured.
For native Android at repo root: `./gradlew bundleRelease` with correct `cd` path.

## Phase 4 — Internal distribution (Play internal)

Store Play service account JSON as secret `GOOGLE_PLAY_SERVICE_ACCOUNT_CREDENTIALS`.

Safe defaults:

```yaml
    environment:
      groups:
        - google_play
    publishing:
      google_play:
        credentials: $GOOGLE_PLAY_SERVICE_ACCOUNT_CREDENTIALS
        track: internal
        submit_as_draft: true
```

First AAB for a new app must be uploaded once manually in Play Console.

## Phase 5 — Store release

**Confirm with user** before production track. Separate workflow:

```yaml
    publishing:
      google_play:
        credentials: $GOOGLE_PLAY_SERVICE_ACCOUNT_CREDENTIALS
        track: production
```

Each upload needs a higher `versionCode` than the previous on that track.

## Common Android signing issues

- Reference name mismatch between yaml and Code signing identities
- `local.properties` not generated — use `$ANDROID_SDK_ROOT`
- `signingConfigs.release` not wired for `CI=true`

Details: https://docs.codemagic.io/troubleshooting/common-android-issues/
Play errors: https://docs.codemagic.io/troubleshooting/common-google-play-errors/

See also [common-failures.md](common-failures.md).
