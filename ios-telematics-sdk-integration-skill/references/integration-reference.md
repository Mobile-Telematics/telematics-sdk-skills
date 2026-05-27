# Damoov TelematicsSDK iOS Integration Reference

Use this as a compact reference for app integration. Verify current SDK source before copying method names.

## Dependency

Use Swift Package Manager for TelematicsSDK dependency integration. Before editing dependencies, verify the latest exact tag from the official SPM repository:

```bash
git ls-remote --tags --refs https://github.com/Mobile-Telematics/telematicsSDK-iOS-new-SPM.git
```

Use the highest semantic version tag exactly:

```swift
.package(url: "https://github.com/Mobile-Telematics/telematicsSDK-iOS-new-SPM", from: "<latest-spm-tag>")
```

As of the last verification for this skill, the SPM repository reported `7.1.0` as the latest version. Do not assume this will remain true. Re-check before each dependency edit.

Also inspect the target app lockfile (`Package.resolved`) and the runtime SDK version with `RPEntry.instance.getSdkVersion()` where runtime access is possible. Lockfiles and runtime checks tell you what is installed; SPM tags tell you what latest version to integrate.

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
try RPEntry.instance.setDeviceID(deviceId: deviceId)
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

Supported flows:

- automatic tracking
- standard manual tracking without tags
- standard manual tracking with tags
- app-controlled persistent manual tracking without tags
- app-controlled persistent manual tracking with tags
- one-time persistent manual tracking without tags
- one-time persistent manual tracking with tags

Put track/origin `RPEntry.instance.api` calls behind `TelematicsAPIService`. Put track-tag and future-tag `RPEntry.instance.api` calls behind `TelematicsTagsService`. `TelematicsService` should call the tags service for tag operations instead of accessing `RPEntry.instance.api` directly.

Automatic tracking:

```swift
try RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
```

Standard manual tracking without tags:

```swift
try RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
RPEntry.instance.setTrackingMode(.standard)
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

Persistent mode is often used by manual app flows, but it is a mode, not a separate product flow. There are two persistent manual patterns:

- App-controlled persistent manual tracking: the app sets `.persistent`, starts tracking, and later decides when to restore `.standard`.
- One-time persistent manual tracking: `startTrackAsPersistent()` sets `.persistent` for this tracking session only. After `stopTracking()` or after the maximum persistent interval is reached, the SDK stops tracking and restores `.standard` automatically.

App-controlled persistent mode start:

```swift
try RPEntry.instance.setDeviceID(deviceId: deviceId)
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

When app-controlled persistent mode is no longer desired:

```swift
RPEntry.instance.stopTracking()
RPEntry.instance.setTrackingMode(.standard)
RPEntry.instance.setEnableSdk(false)
```

One-time persistent manual start:

```swift
try RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
do {
    try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
} catch {
    // handle invalid interval
    return
}
RPEntry.instance.startTrackAsPersistent()
```

One-time persistent manual stop:

```swift
RPEntry.instance.stopTracking()
RPEntry.instance.setEnableSdk(false)
```

Call `startTrackAsPersistent()` on the main thread. Do not call `setTrackingMode(.persistent)` before it and do not manually restore `.standard` after stopping this one-time flow; the SDK owns that transition.

If a service uses one shared manual stop method for all manual flows, track whether the active session was app-controlled persistent. Only that flow should manually call `setTrackingMode(.standard)` after `stopTracking()`.

Tracking status:

```swift
let isTracking = RPEntry.instance.isTracking()
```

`TelematicsService` should also expose status, diagnostics, uploads, heartbeat, RTLD, and accident-detection controls:

```swift
/// Returns whether SDK tracking is currently active.
func isTracking() -> Bool

/// Sends a custom heartbeat with an app-defined reason.
func sendCustomHeartbeat(_ reason: String)

/// Requests upload of pending trips, buffers, and logs.
func uploadUnsentTrips()

/// Fetches the number of locally stored trips waiting for upload.
func getUnsentTripCount(completion: @escaping (_ unsentTripsCount: Int) -> Void)

/// Enables or disables aggressive heartbeat mode.
func setAggressiveHeartbeats(_ aggressive: Bool)

/// Returns whether aggressive heartbeat mode is enabled.
func isAggressiveHeartbeats() -> Bool

/// Returns whether RTLD functionality is enabled.
func isRTLDEnabled() -> Bool

/// Enables or disables accident detection.
func setAccidentDetectionEnabled(_ enabled: Bool)

/// Returns whether accident detection is enabled.
func isAccidentDetectionEnabled() -> Bool

/// Sets accident-detection sensitivity.
func setAccidentDetectionSensitivity(sensitivity: RPAccidentDetectionSensitivity)

/// Returns accident-detection sensitivity.
func getAccidentDetectionSensitivity() -> RPAccidentDetectionSensitivity
```

Delegate:

```swift
final class DamoovTelematicsService: NSObject, RPTrackingStateListenerDelegate {
    func trackingStateChanged(_ state: Bool) {
        print("Telematics trackingStateChanged: \(state)")
    }
}
```

`TelematicsService` should assign and implement `trackingStateDelegate`, `locationDelegate`, `accuracyAuthorizationDelegate`, `lowPowerModeDelegate`, and `rtldDelegate`. Generated delegate methods should only call `print(...)`; the app team can replace these bodies later. Do not assign or implement `speedLimitDelegate` unless the user explicitly requests speed-limit behavior.

## Track And Origin API

Track and origin methods belong in `TelematicsAPIService`.

```swift
final class TelematicsAPIService {
    /// Fetches a processed track by its public track token.
    func getTrackWithTrackToken(
        _ token: String,
        completion: @escaping (_ track: RPTrackProcessed?, _ error: Error?) -> Void
    ) {
        RPEntry.instance.api.getTrackWithTrackToken(token, completion: completion)
    }

    /// Fetches processed tracks using server pagination and an optional date range.
    func getTracksWithOffset(
        _ offset: UInt,
        limit: UInt,
        startDate: Date? = nil,
        endDate: Date? = nil,
        completion: @escaping (_ tracks: [RPTrackProcessed], _ error: Error?) -> Void
    ) {
        RPEntry.instance.api.getTracksWithOffset(
            offset,
            limit: limit,
            startDate: startDate,
            endDate: endDate,
            completion: completion
        )
    }

    /// Fetches available track origin metadata.
    func getTrackOrigins(
        completion: @escaping (_ trackOrigins: RPTrackOrigins?, _ error: Error?) -> Void
    ) {
        RPEntry.instance.api.getTrackOrigins(completion: completion)
    }

    /// Changes the origin metadata for a processed track.
    func changeTrackOrigin(
        _ originCode: RPTrackOriginCode,
        forTrackToken token: String,
        completion: @escaping (_ code: RPStatusCodeResponse?, _ error: Error?) -> Void
    ) {
        RPEntry.instance.api.changeTrackOrigin(originCode, forTrackToken: token, completion: completion)
    }
}
```

## Trip Tags

Track-specific tags require a public track token and network connectivity. Completion may not be on the main thread.

```swift
final class TelematicsTagsService {
    /// Fetches tags attached to a processed track.
    func getTrackTags(
        _ trackToken: String,
        completion: @escaping (_ tags: [RPTag], _ error: Error?) -> Void
    ) {
        RPEntry.instance.api.getTrackTags(trackToken, completion: completion)
    }

    /// Adds tags to a processed track.
    func addTrackTags(
        _ tags: [RPTag],
        to trackToken: String,
        completion: @escaping (_ tags: [RPTag], _ error: Error?) -> Void
    ) {
        RPEntry.instance.api.addTrackTags(tags, to: trackToken, completion: completion)
    }

    /// Removes tags from a processed track.
    func removeTrackTags(
        _ tags: [RPTag],
        from trackToken: String,
        completion: @escaping (_ tags: [RPTag], _ error: Error?) -> Void
    ) {
        RPEntry.instance.api.removeTrackTags(tags, from: trackToken, completion: completion)
    }
}
```

Available methods:

- `getTrackTags(_:completion:)`
- `addTrackTags(_:to:completion:)`
- `removeTrackTags(_:from:completion:)`

## Future Tags

Future tags are handled through `RPEntry.instance.api` and `RPFutureTag`. The SDK exposes completion-handler methods; app code should call `TelematicsTagsService` methods instead of these callbacks directly.

Available methods:

- `getFutureTrackTag(_:completion:)`
- `addFutureTrackTag(_:completion:)`
- `removeFutureTrackTag(_:completion:)`
- `removeAllFutureTrackTags(completion:)`

Implement future-tag methods inside `TelematicsTagsService`:

```swift
final class TelematicsTagsService {
    /// Fetches future tags for the provided date or the SDK default scope.
    func getFutureTrackTag(
        _ date: Date? = nil,
        completion: @escaping (_ status: RPTagStatus, _ tags: [RPFutureTag]) -> Void
    ) {
        RPEntry.instance.api.getFutureTrackTag(date, completion: completion)
    }

    /// Adds a future tag for upcoming trips.
    func addFutureTrackTag(
        _ tag: RPFutureTag,
        completion: @escaping (_ status: RPTagStatus, _ error: Error?) -> Void
    ) {
        RPEntry.instance.api.addFutureTrackTag(tag, completion: completion)
    }

    /// Removes a future tag from upcoming trips.
    func removeFutureTrackTag(
        _ tag: RPFutureTag,
        completion: @escaping (_ status: RPTagStatus, _ error: Error?) -> Void
    ) {
        RPEntry.instance.api.removeFutureTrackTag(tag, completion: completion)
    }

    /// Removes all future tags from upcoming trips.
    func removeAllFutureTrackTags(
        completion: @escaping (_ status: RPTagStatus, _ error: Error?) -> Void
    ) {
        RPEntry.instance.api.removeAllFutureTrackTags(completion: completion)
    }
}
```

`TelematicsService` may add private async helpers around `TelematicsTagsService` callbacks when tracking flow sequencing needs `try await`:

```swift
private func addFutureTrackTag(_ tag: RPFutureTag) async throws {
    try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
        tagsService.addFutureTrackTag(tag) { _, error in
            if let error {
                continuation.resume(throwing: error)
            } else {
                continuation.resume()
            }
        }
    }
}

private func removeAllFutureTrackTags() async throws {
    try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
        tagsService.removeAllFutureTrackTags { _, error in
            if let error {
                continuation.resume(throwing: error)
            } else {
                continuation.resume()
            }
        }
    }
}
```

For standard manual tracking with tags, add future tags before starting:

```swift
try RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
RPEntry.instance.setTrackingMode(.standard)

let tag = RPFutureTag(tag: "TAG", source: "SOURCE")
try await addFutureTrackTag(tag)
RPEntry.instance.startTracking()
```

For stop with tag cleanup:

```swift
try await removeAllFutureTrackTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setEnableSdk(false)
```

App-controlled persistent manual tracking without tags:

```swift
try RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.setTrackingMode(.persistent)
RPEntry.instance.startTracking()
```

App-controlled persistent manual tracking with future tags:

```swift
try RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)

let tag = RPFutureTag(tag: "TAG", source: "SOURCE")
try await addFutureTrackTag(tag)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.setTrackingMode(.persistent)
RPEntry.instance.startTracking()
```

Persistent stop with cleanup for app-controlled persistent mode:

```swift
try await removeAllFutureTrackTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setTrackingMode(.standard)
RPEntry.instance.setEnableSdk(false)
```

One-time persistent manual tracking without tags:

```swift
try RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.startTrackAsPersistent()
```

One-time persistent manual tracking with future tags:

```swift
try RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)

let tag = RPFutureTag(tag: "TAG", source: "SOURCE")
try await addFutureTrackTag(tag)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.startTrackAsPersistent()
```

One-time persistent stop with cleanup:

```swift
try await removeAllFutureTrackTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setEnableSdk(false)
```

The tag APIs are asynchronous. Starting tracking before required tag completion may create a race where the trip starts without the intended tag.
