# React Native iOS Host Setup

This reference covers iOS host app setup for `react-native-telematics`. It is based on the public plugin's iOS implementation, podspec, README, and example app. Verify the installed plugin and generated React Native iOS project before editing.

## Plugin Native Baseline

The verified plugin uses:

- Podspec name `react-native-telematics-sdk`
- iOS platform `13.0`
- Swift `5.0`
- React Native CocoaPods integration through `install_modules_dependencies(s)`
- TelematicsSDK SPM dependency through `spm_dependency(...)`
- TelematicsSDK SPM URL `https://github.com/Mobile-Telematics/telematicsSDK-iOS-new-SPM.git`
- Exact native SDK version `7.1.0` in the podspec

The plugin README also describes manually adding TelematicsSDK SPM to the app target with exact version `7.0.3`; prefer the installed podspec/source when there is a conflict. The app target may still need explicit SPM linkage because TelematicsSDK is a dynamic framework and app lifecycle code imports `TelematicsSDK`.

## Podfile And SPM

In the app `ios/Podfile`, enable dynamic frameworks:

```ruby
use_frameworks! :linkage => :dynamic
```

Use dynamic framework linkage because the native TelematicsSDK iOS dependency is integrated through SPM as a dynamic framework. Do not switch this integration to static linkage for TelematicsSDK.

Then run:

```bash
cd ios
pod install
```

Verify the app target can import `TelematicsSDK` from `AppDelegate` and `SceneDelegate`. If not, add the SPM package to the app target in Xcode:

- Package URL: `https://github.com/Mobile-Telematics/telematicsSDK-iOS-new-SPM.git`
- Product: `TelematicsSDK`
- Version: exact version matching the installed plugin/podspec unless the user requested a specific version
- Target: app target, not only Pods targets
- Embed setting: `Embed & Sign`

## Info.plist

Add required permissions and background modes in the app `Info.plist`:

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

The example app also includes Bluetooth usage descriptions and `bluetooth-central` background mode. Add these only when the product uses Bluetooth-based behavior or the installed SDK/app requires them:

```xml
<key>NSBluetoothAlwaysUsageDescription</key>
<string>...</string>
<key>NSBluetoothPeripheralUsageDescription</key>
<string>...</string>
```

Use product-specific permission strings. Do not leave demo text in production apps unless the user explicitly wants it.

## AppDelegate

Initialize SDK at launch and forward lifecycle methods. `RPEntry.initializeSDK()` must run before `RPEntry.instance` usage:

```swift
import TelematicsSDK
import UIKit

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
  var window: UIWindow?

  func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    RPEntry.initializeSDK()
    RPEntry.instance.application(
      application,
      didFinishLaunchingWithOptions: launchOptions ?? [:]
    )
    return true
  }

  func application(
    _ application: UIApplication,
    handleEventsForBackgroundURLSession identifier: String,
    completionHandler: @escaping () -> Void
  ) {
    RPEntry.instance.application(
      application,
      handleEventsForBackgroundURLSession: identifier,
      completionHandler: completionHandler
    )
  }

  func applicationDidReceiveMemoryWarning(_ application: UIApplication) {
    RPEntry.instance.applicationDidReceiveMemoryWarning(application)
  }

  func applicationWillTerminate(_ application: UIApplication) {
    RPEntry.instance.applicationWillTerminate(application)
  }

  func application(
    _ application: UIApplication,
    performFetchWithCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void
  ) {
    RPEntry.instance.application(application) {
      completionHandler(.newData)
    }
  }
}
```

For SwiftUI-based iOS host apps, bridge this standard `AppDelegate` into the SwiftUI app entry point:

```swift
import SwiftUI

@main
struct ExampleApp: App {
  @UIApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate

  var body: some Scene {
    WindowGroup {
      ContentView()
    }
  }
}
```

If the app does not use scenes, forward app active/foreground/background in `AppDelegate`:

```swift
func applicationDidEnterBackground(_ application: UIApplication) {
  RPEntry.instance.applicationDidEnterBackground(application)
}

func applicationWillEnterForeground(_ application: UIApplication) {
  RPEntry.instance.applicationWillEnterForeground(application)
}

func applicationDidBecomeActive(_ application: UIApplication) {
  RPEntry.instance.applicationDidBecomeActive(application)
}
```

## SceneDelegate

For scene-based apps, forward scene lifecycle in `SceneDelegate`:

```swift
import TelematicsSDK
import UIKit

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
  var window: UIWindow?

  func sceneDidBecomeActive(_ scene: UIScene) {
    RPEntry.instance.sceneDidBecomeActive(scene)
  }

  func sceneWillEnterForeground(_ scene: UIScene) {
    RPEntry.instance.sceneWillEnterForeground(scene)
  }

  func sceneDidEnterBackground(_ scene: UIScene) {
    RPEntry.instance.sceneDidEnterBackground(scene)
  }
}
```

Do not forward both scene and non-scene foreground/background methods for the same lifecycle path unless the app intentionally uses both.

## iOS-Specific JS Calls

Guard iOS-only methods and listeners:

```ts
if (Platform.OS === 'ios') {
  await TelematicsSdk.setDisableTracking(false);
  const aggressive = await TelematicsSdk.isAggressiveHeartbeats();
}
```

iOS-only APIs include permission requests, wrong accuracy state, API language, aggressive heartbeats, disable tracking, low power listener, wrong accuracy listener, and RTLD collected data listener.

## Validation

After iOS changes, run:

```bash
yarn install
yarn typescript
cd ios && pod install
npx react-native run-ios
```

If CocoaPods or Xcode signing is not available, explain the limitation and still run TypeScript/lint checks plus a syntax-level review of `Info.plist`, `Podfile`, `AppDelegate`, and `SceneDelegate`.
