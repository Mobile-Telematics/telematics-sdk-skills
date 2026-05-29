# React Native Android Host Setup

This reference covers Android host app setup for `react-native-telematics`. It is based on the public plugin's Android implementation and README. Verify the installed plugin and Android Gradle Plugin versions before editing.

## Plugin Native Baseline

The verified plugin uses:

- React Native Gradle plugin `0.81.4`
- Android Gradle Plugin `8.12.0`
- Kotlin Gradle plugin `2.2.20`
- Compile SDK `36`
- Min SDK `24`
- Target SDK `36`
- Java target `17`
- `com.telematicssdk:tracking:4.0.0`
- Damoov Maven repository `https://s3.us-east-2.amazonaws.com/android.telematics.sdk.production/`
- `coreLibraryDesugaring "com.android.tools:desugar_jdk_libs:2.1.5"`

Do not force these versions into the host app without checking its existing Gradle setup. Make the smallest compatible changes.

## Manifest

The plugin manifest contributes:

```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
<uses-permission android:name="android.permission.ACTIVITY_RECOGNITION" />
```

The README also requires network permissions in the host app:

```xml
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
```

Remove `android:allowBackup="true"` from the host application manifest if present, per plugin README.

Use Android runtime permission requests in the app before enabling SDK or manual tracking. The Android bridge rejects `setEnableSdk` with `INVALID_PERMISSION` when `ACCESS_FINE_LOCATION` is not granted.

## Gradle

The React Native package should bring its native dependency transitively. If the app needs direct native access or Gradle cannot resolve the dependency, verify that the Damoov repository is available:

```groovy
maven {
    url "https://s3.us-east-2.amazonaws.com/android.telematics.sdk.production/"
}
```

Direct native dependency, when needed:

```groovy
implementation "com.telematicssdk:tracking:4.0.0"
```

Keep Java 17 and desugaring if the app's Gradle setup requires it:

```groovy
android {
    compileOptions {
        coreLibraryDesugaringEnabled true
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }
}

dependencies {
    coreLibraryDesugaring "com.android.tools:desugar_jdk_libs:2.1.5"
}
```

Release settings recommended by the example:

```groovy
android {
    buildTypes {
        release {
            shrinkResources false
            minifyEnabled false
        }
    }
}
```

If the product requires minification, do not just enable it. Add and verify keep rules first.

## Proguard

When minification is enabled or the app already has proguard files, include:

```proguard
-keep public class com.telematicssdk.tracking.** {*;}
-keep class com.reactnativetelematicssdk.** { *; }
```

Also preserve React Native codegen/TurboModule rules according to the app's React Native version.

## Android Initialization Behavior

`TelematicsSdk.initializeSdk()` calls native Android `TrackingApi.initialize(context, settings)` with high accuracy, high stop timeout, autostart enabled, and passive detection enabled. It also registers tag, location, and tracking callbacks.

Do not add a custom native `Application` initialization unless the product needs settings that differ from the plugin defaults. If custom native initialization is required, verify that it does not conflict with the module's own `initializeSdk()` behavior.

## Android-Specific JS Calls

Guard Android-specific methods:

```ts
if (Platform.OS === 'android') {
  await TelematicsSdk.setAndroidAutoStartEnabled({
    enable: true,
    permanent: true,
  });
}
```

Do not call Android-only methods on iOS; the native iOS bridge rejects them with `PLATFORM_ERROR`.

## Validation

After Android changes, run the app-standard checks:

```bash
yarn install
yarn lint
yarn typescript
cd android && ./gradlew assembleDebug
```

For npm apps, use equivalent scripts from `package.json`. If a full Android build is too expensive locally, at minimum run package install, TypeScript/lint checks, and a Gradle sync or smallest available Gradle task.
