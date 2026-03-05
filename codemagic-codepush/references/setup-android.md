# Android Setup (React Native + CodePush)

Apply these steps for Android integration.

## 1) Add CodePush gradle script

In `android/app/build.gradle`, add:

```gradle
apply from: "../../node_modules/@code-push-next/react-native-code-push/android/codepush.gradle"
```

## 2) Update MainApplication

For React Native `0.76+`, update `android/app/src/main/java/.../MainApplication.kt` to override bundle resolution:

```kotlin
import com.microsoft.codepush.react.CodePush

override fun getJSBundleFile(): String {
  return CodePush.getJSBundleFile()
}
```

Keep `PackageList(this)` initialized once in app lifetime.

## 3) Configure deployment/server strings

Add these to `android/app/src/main/res/values/strings.xml`:

```xml
<string moduleConfig="true" name="CodePushServerUrl">https://codepush.pro/</string>
<string moduleConfig="true" name="CodePushDeploymentKey">YOUR_DEPLOYMENT_KEY</string>
```

Optional for code signing verification:

```xml
<string name="CodePushPublicKey">-----BEGIN PUBLIC KEY-----
...public key...
-----END PUBLIC KEY-----</string>
```

## 4) Expo caveat

Pure Expo managed apps are not supported by this native setup. Bare React Native apps with Expo SDK can work when `ReactNativeHostWrapper` does not override `getJSBundleFile`.
