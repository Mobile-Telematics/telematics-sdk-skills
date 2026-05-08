# Common SDK Surface

This reference covers public TelematicsSDK APIs that are common to every integration, regardless of the app's primary tracking flow or SDK tracking mode. 

Use published Damoov docs as baseline setup guidance, but verify method names and deprecations against the installed SDK source or package artifacts before editing app code.

## iOS Project Setup

Every integration should inspect the target app for:

- Dependency manager: use Swift Package Manager for TelematicsSDK dependency integration.
- Latest exact SDK version source: official SPM repository tags.
- `Info.plist` usage descriptions:
  - `NSLocationAlwaysAndWhenInUseUsageDescription`
  - `NSLocationAlwaysUsageDescription`
  - `NSLocationWhenInUseUsageDescription`
  - `NSMotionUsageDescription`
- `UIBackgroundModes`:
  - `fetch`
  - `location`
  - `remote-notification`
- `BGTaskSchedulerPermittedIdentifiers`:
  - `sdk.damoov.apprefreshtaskid`
  - `sdk.damoov.appprocessingtaskid`
- Target capabilities where applicable:
  - Location updates
  - Background fetch
  - Background processing

Do not treat project configuration as optional. Missing permissions or background modes can make otherwise correct SDK code fail at runtime.

## Initialization And Entry Point

Common `RPEntry` APIs:

- `RPEntry.initializeSDK()`: call once before `RPEntry.instance`; it must be the first executable statement in `application(_:didFinishLaunchingWithOptions:)`.
- `RPEntry.instance`: singleton access after initialization.
- `RPEntry.instance.api`: access to `RPAPIEntry`.
- `RPEntry.instance.getSdkVersion()`: read current SDK version.

Do not access `RPEntry.instance` before `initializeSDK()`.

Do not use `RPEntry.isInitialized()` as a guard around normal integration code. The app should call `RPEntry.initializeSDK()` once during launch, then forward the required lifecycle methods unconditionally.

## Lifecycle Wiring

Lifecycle forwarding is common to every tracking flow. A facade or lifecycle adapter may own the SDK calls, but the app must still invoke that adapter from the real app lifecycle entry points.

Use standard lifecycle delegate class names:

- `AppDelegate`
- `SceneDelegate`

Do not invent SDK-specific lifecycle delegate names such as `DamoovAppDelegate` or `DamoovSceneDelegate`.

If both `AppDelegate` and `SceneDelegate` are absent and the app's minimum iOS version is 13.0 or newer, add standard `AppDelegate` and `SceneDelegate` implementations and wire them into the host app. Check the target deployment version before deciding which lifecycle shape to add.

Required `AppDelegate` forwarding:

- `application(_:didFinishLaunchingWithOptions:)`
- `application(_:handleEventsForBackgroundURLSession:completionHandler:)`
- `applicationDidReceiveMemoryWarning(_:)`
- `applicationWillTerminate(_:)`
- `application(_:performFetchWithCompletionHandler:)`

With `SceneDelegate`, forward scene state through `SceneDelegate`:

- `sceneDidBecomeActive(_:)`
- `sceneWillEnterForeground(_:)`
- `sceneDidEnterBackground(_:)`

Without `SceneDelegate`, forward app foreground/background state through `AppDelegate`:

- `applicationDidBecomeActive(_:)`
- `applicationWillEnterForeground(_:)`
- `applicationDidEnterBackground(_:)`

Do not implement lifecycle methods only inside `TelematicsService` and leave them uncalled. The host app's actual `AppDelegate`, `SceneDelegate`, or SwiftUI app lifecycle bridge must call the service/adapter.

Do not forward both SceneDelegate and non-Scene foreground/background methods for the same lifecycle path unless the host app intentionally uses both and has verified there is no duplicated SDK work.

Lifecycle forwarding is mandatory. Do not guard SDK lifecycle calls with app-side checks such as:

- `RPEntry.isInitialized`
- empty device ID / token checks
- `hasConfiguredDeviceId`
- custom service state checks

The SDK source already guards missing device ID or disabled SDK state where applicable. App-side guards can accidentally suppress required background URL session handling, foreground/background processing, log flushing, uploads, and region monitoring.

## Device Identity And SDK Enablement

Common current APIs:

- `setDeviceID(deviceId:)`
- `getDeviceId()`
- `clearDeviceID()`
- `logout()`
- `setEnableSdk(_:)`
- `isSDKEnabled()`
- `setDisableTracking(disableTracking:)`
- `isDisableTracking()`

Key behavior:

- `setEnableSdk(false)` disables SDK collection and stops tracking-related operations, but keeps the device ID.
- `logout()` disables the SDK and clears the device ID. Use it for logout/account removal semantics only.
- `setDisableTracking(disableTracking:)` prevents new user-initiated tracking sessions; it is not the same as disabling the SDK.

Deprecated APIs to avoid:

- `virtualDeviceToken`
- `removeVirtualDeviceToken()`
- `disableTracking`

Use `references/api-migration.md` for replacement details.

## Permissions

Common permission APIs:

- `requestLocationAlwaysPermission()`
- `requestMotionPermission()`
- `isAllRequiredPermissionsAndSensorsGranted()`
- `isWrongAccuracyState()`

Deprecated APIs to avoid:

- `isAllRequiredPermissionsGranted()`
- `wrongAccuracyState`

The app should ensure `Info.plist` contains the required usage descriptions before requesting permissions. Permission checks and requests are common setup, not tracking-flow-specific code.

## Status, Diagnostics, And Uploads

Common APIs:

- `isTracking()`: read current SDK tracking activity.
- `sendCustomHeartbeat(_:)`: send an app-defined heartbeat reason.
- `uploadUnsentTrips()`: request upload of pending trips, buffers, and logs.
- `getUnsentTripCount(completion:)`
- `getUnsentBufferCount(completion:)`
- `getUnsentLogsCount()`
- `setAggressiveHeartbeats(_:)`
- `isAggressiveHeartbeats()`
- `isRTLDEnabled()`

Deprecated APIs to avoid:

- `isTrackingActive()`
- `aggressiveHeartbeat()`
- `isRTDEnabled()`

These APIs are useful for diagnostics, status surfaces, and operational screens. Do not mix their behavior with primary tracking flow selection.

## Accident Detection

Common accident APIs:

- `setAccidentDetectionEnabled(_:)`
- `isAccidentDetectionEnabled()`
- `setAccidentDetectionSensitivity(sensitivity:)`
- `getAccidentDetectionSensitivity()`

Deprecated APIs to avoid:

- `enableAccidents(_:)`
- `isEnabledAccidents()`
- `accidentDetectionSensitivity`

Expose these only when the host app needs product controls for accident detection.

## Delegates

`RPEntry` exposes weak delegate properties:

- `trackingStateDelegate: RPTrackingStateListenerDelegate?`
  - `trackingStateChanged(_ state: Bool)`
- `locationDelegate: RPLocationDelegate?`
  - `onLocationChanged(_:)`
  - `onNewEvents(_:)`
- `accuracyAuthorizationDelegate: RPAccuracyAuthorizationDelegate?`
  - `wrongAccuracyAuthorization()`
- `lowPowerModeDelegate: RPLowPowerModeDelegate?`
  - `lowPowerMode(_:)`
- `rtldDelegate: RPRTLDDelegate?`
  - `rtldColectedData()`
- `speedLimitDelegate: RPSpeedLimitDelegate?`
  - `speedLimit`
  - `timeThreshold`
  - `speedLimitNotification(_:speed:latitude:longitude:date:)`

Keep delegates alive outside the SDK because the SDK stores them weakly. Dispatch to the main queue or main actor before updating UI.

## RPAPIEntry Track APIs

Available through `RPEntry.instance.api`:

- `getTrackWithTrackToken(_:completion:)`
- `getTracksWithOffset(_:limit:startDate:endDate:completion:)`
- `getSharedTrack(with:completion:)`
- `makeTrack(_:shared:completion:)`
- `getTrackOrigins(completion:)`
- `changeTrackOrigin(_:forTrackToken:completion:)`

These APIs generally require:

- a valid device ID;
- network connectivity;
- callback error handling;
- main-thread dispatch before UI updates.

## RPAPIEntry Tag APIs

Track tags:

- `getTrackTags(_:completion:)`
- `addTrackTags(_:to:completion:)`
- `removeTrackTags(_:from:completion:)`

Future tags:

- `getFutureTrackTag(_:completion:)`
- `addFutureTrackTag(_:completion:)`
- `removeFutureTrackTag(_:completion:)`
- `removeAllFutureTrackTags(completion:)`

Future-tag APIs are commonly used by manual flows, but the API surface itself is common. If a future tag is required for a manually started trip, wait for the add completion before calling `startTracking()`.

## Callback Handling

`RPAPIEntry` and some `RPEntry` APIs are callback-based. Completion handlers are not guaranteed to run on the main thread.

Recommended service policy:

- wrap SDK callbacks in `async`/`await` where the app architecture allows it;
- keep raw SDK callbacks private to the Telematics service/facade;
- propagate errors instead of silently ignoring them;
- dispatch to `MainActor` before updating UI;
- preserve callback APIs only if the host app architecture explicitly requires them.
