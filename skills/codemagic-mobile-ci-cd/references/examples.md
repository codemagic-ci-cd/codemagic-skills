# Examples

## Example 1: Flutter app, first build

**User says:** "Set up CI/CD for my Flutter app"

**Actions:**

1. Detect Flutter via `pubspec.yaml`
2. Depth: `validate` (unless user asks for more)
3. Phase 0: [connect-and-yaml-basics.md](connect-and-yaml-basics.md)
4. Phase 1: [stack-flutter.md](stack-flutter.md) + unsigned Android workflow
5. Commit yaml, confirm detection, start build

**Result:** Successful build with debug APK artifact.

## Example 2: React Native Expo, internal testing

**User says:** "Deploy my Expo app to TestFlight"

**Actions:**

1. Detect Expo via `app.config.js`
2. Depth: `internal` (phases 0–4)
3. Phase 1: [stack-react-native.md](stack-react-native.md) with prebuild — green unsigned iOS build
4. Phase 2–3: [ios-release-lane.md](ios-release-lane.md) — ASC key, certs, signed IPA
5. Phase 4: `submit_to_testflight: true`, `submit_to_app_store: false`

**Result:** IPA uploaded to TestFlight internal testers.

## Example 3: Failed build debugging

**User says:** "My Codemagic build failed, build ID abc123"

**Actions:**

1. [build-logs-api.md](build-logs-api.md) — GET build, find failed step, fetch step log
2. Error: Gradle SDK location → add `local.properties` script
3. Update `codemagic.yaml`, commit, retry build

**Result:** Second build succeeds.

## Example 4: Android native to Play internal

**User says:** "Publish my Kotlin app to Google Play internal track"

**Actions:**

1. Detect native Android via Gradle `android/` without RN/Flutter markers
2. Depth: `internal`
3. Phases 0–1: unsigned build via [stack-android-native.md](stack-android-native.md)
4. Phases 2–4: [android-release-lane.md](android-release-lane.md) with `track: internal`

**Result:** Signed AAB on Play internal testing track.

## Example 5: User wants production

**User says:** "Ship to production on Google Play"

**Actions:**

1. Confirm phases 0–4 already work (or run through them)
2. **Ask user to confirm** production track
3. Phase 5: separate workflow with `track: production` per [android-release-lane.md](android-release-lane.md)

**Result:** AAB submitted to production track after explicit confirmation.
