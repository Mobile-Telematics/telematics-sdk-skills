# Flutter iOS Host Setup

This reference covers iOS host app setup for the Damoov Flutter plugin. It is based on the plugin's iOS implementation and README. Verify the installed plugin and generated Flutter iOS project before editing.

## Plugin Native Baseline

The verified plugin uses:

- Swift Package target platform `.iOS("13.0")`
- TelematicsSDK SPM dependency `https://github.com/Mobile-Telematics/telematicsSDK-iOS-new-SPM.git`, from `7.1.0`
- Flutter plugin class `TelematicsSDKPlugin`
- `registrar.addApplicationDelegate(instance)`
- `registrar.addSceneDelegate(instance)`

The host app still must initialize `RPEntry` before Flutter plugin registration and provide permissions/background configuration.

## Info.plist

Add required permissions and background modes in `ios/Runner/Info.plist`:

```xml
<key>UIBackgroundModes</key>
<array>
    <string>fetch</string>
    <string>location</string>
    <string>processing</string>
    <string>remote-notification</string>
</array>
<key>NSMotionUsageDescription</key>
<string>Please provide motion permissions.</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>Please provide location permissions.</string>
<key>NSLocationAlwaysUsageDescription</key>
<string>Please provide always-on location permissions.</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>Please provide always-on location permissions.</string>
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>sdk.damoov.apprefreshtaskid</string>
    <string>sdk.damoov.appprocessingtaskid</string>
</array>
```

Use product-specific permission strings. Do not leave demo text in production apps unless the user explicitly wants it.

## AppDelegate Without UISceneDelegate

For Flutter apps that do not support `UISceneDelegate` and use Flutter versions below the UISceneDelegate migration path, initialize the SDK before `GeneratedPluginRegistrant`:

```swift
import Flutter
import TelematicsSDK
import UIKit

@main
@objc class AppDelegate: FlutterAppDelegate {
    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        RPEntry.initializeSDK()
        RPEntry.instance.application(
            application,
            didFinishLaunchingWithOptions: launchOptions ?? [:]
        )
        GeneratedPluginRegistrant.register(with: self)
        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }
}
```

`RPEntry.initializeSDK()` must be the first executable statement in `didFinishLaunchingWithOptions`.

## AppDelegate With Flutter Implicit Engine

For Flutter versions using the implicit engine and UISceneDelegate adoption, register plugins through `FlutterImplicitEngineDelegate`:

```swift
import Flutter
import TelematicsSDK
import UIKit

@main
@objc class AppDelegate: FlutterAppDelegate, FlutterImplicitEngineDelegate {
    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        RPEntry.initializeSDK()
        RPEntry.instance.application(
            application,
            didFinishLaunchingWithOptions: launchOptions ?? [:]
        )
        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }

    func didInitializeImplicitFlutterEngine(_ engineBridge: any FlutterImplicitEngineBridge) {
        GeneratedPluginRegistrant.register(with: engineBridge.pluginRegistry)
    }
}
```

Use Flutter's UISceneDelegate migration guide for the surrounding project changes. Do not blindly add this shape to old Flutter projects that still use generated registration in `didFinishLaunchingWithOptions`.

## Lifecycle Forwarding

The plugin registers itself as application and scene delegate, forwarding:

- background URL session events
- memory warning
- termination
- background fetch
- app foreground/background/active
- scene foreground/background/active

The host app must still initialize `RPEntry` at launch. Do not guard `RPEntry.instance.application(...)` with device ID checks or custom state flags.

Do not add a SwiftUI lifecycle bridge for ordinary Flutter integrations. The Flutter plugin registers application and scene delegates so the app developer should initialize `RPEntry` in the Flutter iOS host and let the plugin encapsulate lifecycle forwarding.

## iOS-Specific Dart Calls

Guard iOS-only methods:

```dart
if (Platform.isIOS) {
  await trackingApi.setDisableTracking(value: false);
  final aggressive = await trackingApi.isAggressiveHeartbeats();
}
```

iOS-only methods include permission requests, wrong accuracy state, API language, aggressive heartbeats, disable tracking, and iOS RTLD streams.

## Validation

After iOS changes, run:

```bash
flutter pub get
flutter analyze
flutter test
cd ios && pod install
flutter build ios --no-codesign
```

If CocoaPods or Xcode signing is not available, explain the limitation and still run Dart analysis/tests plus a syntax-level review of `Info.plist` and `AppDelegate.swift`.
