# Flutter Android Host Setup

This reference covers Android host app setup for the Damoov Flutter plugin. It is based on the plugin's Android implementation and README. Verify the installed plugin and Android Gradle Plugin versions before editing.

## Plugin Native Baseline

The verified plugin uses:

- Android Gradle Plugin `8.12.0`
- Kotlin `2.2.20`
- Compile SDK `36`
- Min SDK `24`
- Java/Kotlin target `17`
- `com.telematicssdk:tracking:4.0.0`
- Damoov Maven repository `https://s3.us-east-2.amazonaws.com/android.telematics.sdk.production/`
- Gradle 8+

Do not force these versions into the host app without checking its existing Gradle setup. Make the smallest compatible changes.

## Manifest

The plugin manifest contributes location, storage, and activity-recognition permissions plus an `android:name` application value. Host apps commonly need manifest merge controls.

Add `tools` namespace and `tools:replace` when required:

```xml
<manifest
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
    <uses-permission android:name="android.permission.ACTIVITY_RECOGNITION" />

    <application
        tools:replace="android:label,android:name">
        ...
    </application>
</manifest>
```

Use Android runtime permission requests in the app before enabling SDK or manual tracking. The plugin's Android bridge returns `MISSING_PERMISSION` if `ACCESS_FINE_LOCATION` is not granted when enabling SDK.

The plugin manifest also sets `android:requestLegacyExternalStorage="true"` on its application entry. Preserve or explicitly resolve this during manifest merge only after checking the target app's Android API and storage policy.

## Gradle

The Flutter plugin should bring its native dependency transitively. If the app needs direct native access or advanced `Application` customization, add the Damoov repository and dependency in the native Android module after checking existing Gradle structure.

Repository:

```groovy
maven {
    url "https://s3.us-east-2.amazonaws.com/android.telematics.sdk.production/"
}
```

Dependency for direct native access:

```groovy
implementation "com.telematicssdk:tracking:4.0.0"
```

Release settings recommended by the plugin README:

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

Add when the host app uses minification or already has proguard files:

```proguard
-keep public class com.telematicssdk.tracking.** {*;}
```

The plugin example also uses a host `proguard-files.pro`; follow the app's existing file naming.

## Advanced Tracking Settings

For native Android settings, override the application class with `TelematicsSDKApp` and initialize `TrackingApi` before `super.onCreate()` as shown in the plugin README:

```kotlin
import com.telematicssdk.TelematicsSDKApp
import com.telematicssdk.tracking.TrackingApi
import com.telematicssdk.tracking.Settings

class App : TelematicsSDKApp() {
    override fun onCreate() {
        val api = TrackingApi.getInstance()
        api.initialize(this, setTelematicsSettings())
        super.onCreate()
    }

    override fun setTelematicsSettings(): Settings {
        return Settings()
            .stopTrackingTimeout(Settings.stopTrackingTimeHigh)
            .accuracy(Settings.accuracyHigh)
            .autoStartOn(true)
            .passiveDetectionOn(true)
    }
}
```

Then set it in `AndroidManifest.xml`:

```xml
<application android:name=".App">
    ...
</application>
```

Only add this when the app needs native tracking settings. For ordinary Flutter integration, prefer Dart `TrackingApi` methods.

## Android-Specific Dart Calls

Guard Android-specific methods:

```dart
if (Platform.isAndroid) {
  await trackingApi.setAndroidAutoStartEnabled(
    enable: true,
    permanent: true,
  );
}
```

Do not call Android-only methods on iOS.

## Validation

After Android changes, run:

```bash
flutter pub get
flutter analyze
flutter test
flutter build apk --debug
```

If a full build is too expensive locally, at minimum run `flutter pub get`, `flutter analyze`, and inspect manifest merge/build errors from the smallest Gradle task available in the app.
