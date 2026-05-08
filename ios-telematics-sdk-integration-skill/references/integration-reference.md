# Damoov TelematicsSDK iOS Integration Reference

Use this as a compact reference for app integration. Verify current SDK source before copying method names.

## Dependency

Prefer Swift Package Manager. Before editing, verify the latest exact tag from the official SPM repository:

```bash
git ls-remote --tags --refs https://github.com/Mobile-Telematics/telematicsSDK-iOS-new-SPM.git
```

As of the last verification for this skill, the latest tag returned by the official repository was `7.0.3`, so the SPM dependency is:

```swift
.package(url: "https://github.com/Mobile-Telematics/telematicsSDK-iOS-new-SPM", from: "7.0.3")
```

Use CocoaPods only when the host app already standardizes on CocoaPods or cannot use SPM:

```ruby
pod 'TelematicsSDK', '7.0.3'
```

Do not assume `7.0.3` will remain latest. Check the app lockfile and the local SDK version with `RPEntry.instance.getSdkVersion()` where runtime access is possible.

## Required iOS Configuration

Check `Info.plist` for:

```xml
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>...</string>
<key>NSLocationAlwaysUsageDescription</key>
<string>...</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>...</string>
<key>NSMotionUsageDescription</key>
<string>...</string>
<key>UIBackgroundModes</key>
<array>
  <string>fetch</string>
  <string>location</string>
  <string>remote-notification</string>
</array>
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
  <string>sdk.damoov.apprefreshtaskid</string>
  <string>sdk.damoov.appprocessingtaskid</string>
</array>
```

Also check target capabilities for Location updates, Background fetch, and Background processing where applicable.

## Lifecycle Forwarding

Forward all applicable lifecycle events to `RPEntry.instance`. Do not stop at launch-only setup.

If the app uses a `TelematicsLifecycleAdapter`, the code below can live inside that adapter, but the real `AppDelegate`, `SceneDelegate`, or SwiftUI lifecycle bridge must call the adapter from the corresponding lifecycle methods. Implementing these methods in the facade is not enough if the host app never invokes them.

Use standard lifecycle delegate class names: `AppDelegate` and `SceneDelegate`. Do not create SDK-specific delegate names such as `DamoovAppDelegate` or `DamoovSceneDelegate`.

If both `AppDelegate` and `SceneDelegate` are absent and the app's minimum iOS version is 13.0 or newer, add standard `AppDelegate` and `SceneDelegate` implementations and wire them into the app. Check the target deployment version before choosing the lifecycle shape.

Lifecycle forwarding is mandatory. Do not skip these SDK calls behind guards such as `RPEntry.isInitialized`, empty device ID checks, `hasConfiguredDeviceId`, or custom service state. The SDK handles missing device ID or disabled state where applicable.

In `application(_:didFinishLaunchingWithOptions:)`, `RPEntry.initializeSDK()` must be the first executable statement, immediately followed by `RPEntry.instance.application(application, didFinishLaunchingWithOptions: launchOptions ?? [:])`.

### AppDelegate

```swift
import TelematicsSDK

func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {
    RPEntry.initializeSDK()
    RPEntry.instance.application(application, didFinishLaunchingWithOptions: launchOptions ?? [:])
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
```

### With SceneDelegate

```swift
func sceneDidBecomeActive(_ scene: UIScene) {
    RPEntry.instance.sceneDidBecomeActive(scene)
}

func sceneWillEnterForeground(_ scene: UIScene) {
    RPEntry.instance.sceneWillEnterForeground(scene)
}

func sceneDidEnterBackground(_ scene: UIScene) {
    RPEntry.instance.sceneDidEnterBackground(scene)
}
```

### Without SceneDelegate

```swift
func applicationDidBecomeActive(_ application: UIApplication) {
    RPEntry.instance.applicationDidBecomeActive(application)
}

func applicationWillEnterForeground(_ application: UIApplication) {
    RPEntry.instance.applicationWillEnterForeground(application)
}

func applicationDidEnterBackground(_ application: UIApplication) {
    RPEntry.instance.applicationDidEnterBackground(application)
}
```

If the app uses `SceneDelegate`, put foreground/background/active forwarding in `SceneDelegate`. If it does not use scenes, put those methods in `AppDelegate`.

Automatic tracking enablement:

```swift
RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
```

Disable SDK collection while keeping the device ID:

```swift
RPEntry.instance.setEnableSdk(false)
```

Logout, when token removal is intended:

```swift
RPEntry.instance.logout()
```

`logout()` also clears the device ID, so the app must set the device ID again before enabling the SDK later.

## Tracking Flows

These are app-level flows that decide when SDK collection or a manual trip should start and stop. Do not confuse them with SDK tracking modes (`.standard` and `.persistent`), which configure how the SDK tracks after a flow enables or starts tracking.

When implementing a reusable Telematics service, ask which flow is primary if the user has not already specified it. Implement all supported flows, but place the primary flow first with a `// MARK: - Primary Flow: ...` section and place the rest below it as `// MARK: - Additional Flow: ...` sections.

Manual tracking:

```swift
RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
RPEntry.instance.startTracking()
```

Manual stop:

```swift
RPEntry.instance.stopTracking()
RPEntry.instance.setEnableSdk(false)
```

## SDK Tracking Modes

The SDK exposes two tracking modes:

- `.standard`: normal SDK tracking behavior.
- `.persistent`: SDK ignores ordinary stop triggers and uses a configured maximum interval.

Persistent mode is often used by manual app flows, but it is a mode, not a separate product flow.

Persistent mode start:

```swift
RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
do {
    try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
} catch {
    // handle invalid interval
    return
}
RPEntry.instance.setTrackingMode(.persistent)
RPEntry.instance.startTracking()
```

When persistent mode is no longer desired:

```swift
RPEntry.instance.stopTracking()
RPEntry.instance.setTrackingMode(.standard)
RPEntry.instance.setEnableSdk(false)
```

Tracking status:

```swift
let isTracking = RPEntry.instance.isTracking()
```

Delegate:

```swift
final class TrackingObserver: NSObject, RPTrackingStateListenerDelegate {
    func trackingStateChanged(_ state: Bool) {
        DispatchQueue.main.async {
            // update app state
        }
    }
}
```

## Trip Tags

Track-specific tags require a public track token and network connectivity. Completion may not be on the main thread.

```swift
let tag = RPTag(tag: "ExampleTag", source: "ExampleTagSource")

RPEntry.instance.api.addTrackTags([tag], to: trackToken) { tags, error in
    if let error {
        // handle error
        return
    }
    DispatchQueue.main.async {
        // update UI with tags
    }
}
```

Available methods:

- `getTrackTags(_:completion:)`
- `addTrackTags(_:to:completion:)`
- `removeTrackTags(_:from:completion:)`

## Future Tags

Future tags are handled through `RPEntry.instance.api` and `RPFutureTag`. The SDK exposes completion-handler methods; app code should call async service methods instead of these callbacks directly.

Available methods:

- `getFutureTrackTag(_:completion:)`
- `addFutureTrackTag(_:completion:)`
- `removeFutureTrackTag(_:completion:)`
- `removeAllFutureTrackTags(completion:)`

Prefer wrapping these callback APIs in Swift `async`/`await` inside the app-owned Telematics service:

```swift
private func addFutureTag(_ tag: RPFutureTag) async throws {
    try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
        RPEntry.instance.api.addFutureTrackTag(tag) { _, error in
            if let error {
                continuation.resume(throwing: error)
            } else {
                continuation.resume()
            }
        }
    }
}

private func removeAllFutureTags() async throws {
    try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
        RPEntry.instance.api.removeAllFutureTrackTags { _, error in
            if let error {
                continuation.resume(throwing: error)
            } else {
                continuation.resume()
            }
        }
    }
}
```

For manual tracking with a required future tag, add the tag before starting:

```swift
RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)

let tag = RPFutureTag(tag: "TAG", source: "SOURCE")
try await addFutureTag(tag)
RPEntry.instance.startTracking()
```

For stop with tag cleanup:

```swift
try await removeAllFutureTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setEnableSdk(false)
```

Persistent manual tracking with a future tag:

```swift
RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)

let tag = RPFutureTag(tag: "TAG", source: "SOURCE")
try await addFutureTag(tag)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.setTrackingMode(.persistent)
RPEntry.instance.startTracking()
```

Persistent stop with cleanup:

```swift
try await removeAllFutureTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setTrackingMode(.standard)
RPEntry.instance.setEnableSdk(false)
```

The tag APIs are asynchronous. Starting tracking before required tag completion may create a race where the trip starts without the intended tag.
