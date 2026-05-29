# CodePush Migrations

## Migrating from AppCenter CodePush

AppCenter CodePush was shut down in March 2025. Codemagic runs its own independent server at `codepush.pro` — nothing carries over from AppCenter. This is not a redirect or data import.

### What needs to change

**1. Swap the npm package**

```bash
npm uninstall react-native-code-push
npm install @code-push-next/react-native-code-push
cd ios && pod install && cd ..
```

Update all JS imports:
```javascript
// Before
import codePush from 'react-native-code-push';

// After
import codePush from '@code-push-next/react-native-code-push';
```

**2. Swap the CLI**

```bash
npm uninstall -g appcenter-cli
npm install -g @codemagic/code-push-cli
```

**3. Update the server URL in three places**

- `Info.plist` (iOS) — `CodePushServerURL`
- `strings.xml` (Android) — `CodePushServerUrl`
- CLI login command — `code-push login "https://codepush.pro/" --accessKey $TOKEN`

**4. Re-register apps and get new deployment keys**

AppCenter apps and deployment keys do not exist on Codemagic's server. Start fresh:

```bash
code-push app add MyApp-iOS
code-push app add MyApp-Android
code-push deployment list MyApp-iOS -k
```

Update the new keys in `Info.plist` and `strings.xml`.

**5. Ship a new store binary**

This is the critical step customers miss. Until a new binary containing the `codepush.pro` server URL and new deployment keys reaches users, existing installs will never receive OTA updates — the old AppCenter keys are permanently dead. There is no way to migrate existing users without a store release.

### Agent checklist for AppCenter migration tickets

When a customer says "CodePush stopped working" or "I migrated from AppCenter":

1. Are they still using `react-native-code-push` (old) or `@code-push-next/react-native-code-push` (new)?
2. Is the server URL `codepush.pro` in all three locations?
3. Have they re-registered their apps on Codemagic's server?
4. Have they shipped a new binary with the updated keys?
5. Do they understand users on old binaries are permanently cut off until that new binary reaches them?

---

## Migrating from Expo Updates (EAS Update)

Customers moving from Expo Updates to Codemagic CodePush need to understand the concept mapping:

| Expo Updates | CodePush |
|---|---|
| EAS project / `projectId` | CodePush app registration |
| Channel (e.g., production) | Deployment (Staging / Production) |
| `eas update --channel` | `code-push release-react` |
| Runtime version | Target binary version |
| `expo-updates` package | `@code-push-next/react-native-code-push` |

### Migration steps

1. Install CLI and authenticate with CodePush token
2. Register separate apps per platform (`MyApp-iOS`, `MyApp-Android`)
3. Remove `expo-updates` package, EAS config from `app.json`, and native EAS metadata
4. Install `@code-push-next/react-native-code-push`
5. Configure deployment keys in `Info.plist` (iOS) and `strings.xml` (Android)
6. Update native app delegates to load JS bundle from CodePush (see `setup-ios.md` and `setup-android.md`)
7. Wrap root component with `codePush(App)` (see `codepush-js-sdk.md`)
8. Test with `code-push release-react` to Staging
9. Update CI/CD pipeline to use CodePush commands instead of `eas update`
10. Adopt Staging → Production promotion workflow

### Agent checklist for Expo migration tickets

1. Have they removed `expo-updates` and EAS config from `app.json`?
2. Is the server URL `codepush.pro` in `Info.plist` and `strings.xml`?
3. Have they re-registered apps on Codemagic's server? (EAS channels do not carry over)
4. Have they shipped a new store binary with the new deployment keys?
5. Do they understand users on old Expo binaries won't receive CodePush updates until the new binary reaches them?
