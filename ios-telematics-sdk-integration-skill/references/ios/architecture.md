# Recommended App Architecture

## Decision

Wrap `TelematicsSDK` in an app-owned service/facade. Do not scatter `RPEntry.instance` calls across screens, reducers, interactors, or view models.

This is a recommendation, not a verified SDK requirement. The reason is architectural: `RPEntry` is a global singleton with mutable state, lifecycle hooks, async tag operations, tracking modes, and deprecated API migration concerns. A facade gives the app one place to sequence calls, map app concepts to SDK concepts, and test business decisions without invoking the SDK everywhere.

## Suggested Boundaries

Use names that match the host app conventions, but keep these responsibilities separate:

- `TelematicsService`: owns `RPEntry` lifecycle, configuration, tracking, status, diagnostics, accident-detection, and delegate calls, excluding `RPEntry.instance.api`.
- `TelematicsAPIService`: owns track/origin methods from `RPEntry.instance.api`.
- `TelematicsTagsService`: owns track-tag and future-tag methods from `RPEntry.instance.api`.
- `TelematicsLifecycleAdapter`: centralizes `UIApplicationDelegate` / `UISceneDelegate` forwarding, but is called by standard app delegates named `AppDelegate` and `SceneDelegate`.
- `TrackingSessionRequest`: app-level input for starting tracking, e.g. flow, optional tag, optional persistent interval. Device identity is configured separately from tracking start/stop flows.
- `TrackingFlow`: app-level flow such as automatic enablement, standard manual start/stop, app-controlled persistent manual start/stop, or one-time persistent manual start/stop, with or without tags.
- `TrackingMode`: app-level enum that maps to SDK modes `.standard` or `.persistent`.
- `TelematicsError`: app-level error mapping for SDK/API failures.

Avoid putting business decisions into `AppDelegate`. App/scene delegates should initialize and forward lifecycle only. Use standard delegate names: `AppDelegate` and `SceneDelegate`; do not create SDK-specific delegate class names such as `DamoovAppDelegate` or `DamoovSceneDelegate`.

`TelematicsService` should receive a `TelematicsTagsService` dependency and must not call `RPEntry.instance.api` directly. This keeps SDK singleton/lifecycle/tracking calls separate from network-backed tag wrappers.

The lifecycle adapter must forward the full applicable SDK lifecycle surface:

- AppDelegate always: launch, background URL session events, memory warning, termination, background fetch.
- SceneDelegate apps: scene active, scene foreground, scene background.
- Non-SceneDelegate apps: application active, application foreground, application background.

Do not forward both scene and non-scene foreground/background methods for the same app lifecycle path unless the host app intentionally uses both and has verified there is no duplicated SDK work.

The adapter must be called from the real app lifecycle entry points. Do not only implement adapter methods and forget to invoke them from `AppDelegate`, `SceneDelegate`, or a SwiftUI lifecycle bridge.

If both `AppDelegate` and `SceneDelegate` are absent and the app's minimum iOS version is 13.0 or newer, add standard `AppDelegate` and `SceneDelegate` implementations and wire them into the host app. Do not replace them with SDK-specific delegate names.

Lifecycle forwarding is mandatory and should be unconditional. Do not skip SDK lifecycle calls behind guards such as `RPEntry.isInitialized`, empty device ID checks, `hasConfiguredDeviceId`, or service state checks. The SDK source already handles missing device ID or disabled state where needed.

## Minimal Protocol

```swift
protocol TelematicsServicing {
    /// Assigns SDK delegates owned by the service.
    func configure()

    /// Sets or replaces the SDK device ID for the current logged-in user/session.
    func setDeviceId(_ deviceId: String) throws

    /// Clears SDK identity for user logout/account removal.
    func logout()

    /// Enables automatic SDK tracking. Requires a device ID to have been configured separately.
    func enableAutomaticTracking() throws

    /// Disables SDK collection while preserving the device ID.
    func disableSdk()

    /// Starts a standard manual tracking session without future tags.
    func startStandardManualTracking() throws

    /// Adds future tags and starts a standard manual tracking session.
    func startStandardManualTrackingWithTags(tags: [FutureTripTag]) async throws

    /// Starts an app-controlled persistent manual tracking session without future tags.
    func startPersistentManualTracking(maxIntervalMinutes: Int) throws

    /// Adds future tags and starts an app-controlled persistent manual tracking session.
    func startPersistentManualTrackingWithTags(tags: [FutureTripTag], maxIntervalMinutes: Int) async throws

    /// Starts a one-time persistent manual tracking session without future tags.
    func startOneTimePersistentManualTracking(maxIntervalMinutes: Int) throws

    /// Adds future tags and starts a one-time persistent manual tracking session.
    func startOneTimePersistentManualTrackingWithTags(tags: [FutureTripTag], maxIntervalMinutes: Int) async throws

    /// Stops manual tracking and optionally removes future tags first.
    func stopManualTracking(removeFutureTags: Bool) async throws

    /// Returns whether SDK tracking is currently active.
    func isTracking() -> Bool

    /// Returns the latest known device ID registration state.
    func getDeviceIdRegistrationState() -> RPDeviceIdRegistrationState

    /// Fetches the current automatic and manual tracking availability state.
    func getTrackingState(completion: @escaping (_ state: RPTrackingState) -> Void)

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
}
```

Do not model temporary SDK disablement with a `keepDeviceId` parameter. Use `disableSdk()` for `setEnableSdk(false)` and `logout()` only for real logout/account-removal semantics.

`TelematicsAPIService` should wrap track/origin methods from `RPEntry.instance.api`:

```swift
protocol TelematicsAPIServicing {
    /// Fetches a processed track by its public track token.
    func getTrackWithTrackToken(
        _ token: String,
        completion: @escaping (_ track: RPTrackProcessed?, _ error: Error?) -> Void
    )

    /// Fetches processed tracks using server pagination and an optional date range.
    func getTracksWithOffset(
        _ offset: UInt,
        limit: UInt,
        startDate: Date? = nil,
        endDate: Date? = nil,
        completion: @escaping (_ tracks: [RPTrackProcessed], _ error: Error?) -> Void
    )

    /// Fetches available track origin metadata.
    func getTrackOrigins(
        completion: @escaping (_ trackOrigins: RPTrackOrigins?, _ error: Error?) -> Void
    )

    /// Changes the origin metadata for a processed track.
    func changeTrackOrigin(
        _ originCode: RPTrackOriginCode,
        forTrackToken token: String,
        completion: @escaping (_ code: RPStatusCodeResponse?, _ error: Error?) -> Void
    )
}
```

`TelematicsTagsService` should wrap track-tag and future-tag methods from `RPEntry.instance.api`:

```swift
protocol TelematicsTagsServicing {
    /// Fetches tags attached to a processed track.
    func getTrackTags(
        _ trackToken: String,
        completion: @escaping (_ tags: [RPTag], _ error: Error?) -> Void
    )

    /// Adds tags to a processed track.
    func addTrackTags(
        _ tags: [RPTag],
        to trackToken: String,
        completion: @escaping (_ tags: [RPTag], _ error: Error?) -> Void
    )

    /// Removes tags from a processed track.
    func removeTrackTags(
        _ tags: [RPTag],
        from trackToken: String,
        completion: @escaping (_ tags: [RPTag], _ error: Error?) -> Void
    )

    /// Fetches future tags for the provided date or the SDK default scope.
    func getFutureTrackTag(
        _ date: Date? = nil,
        completion: @escaping (_ status: RPTagStatus, _ tags: [RPFutureTag]) -> Void
    )

    /// Adds a future tag for upcoming trips.
    func addFutureTrackTag(
        _ tag: RPFutureTag,
        completion: @escaping (_ status: RPTagStatus, _ error: Error?) -> Void
    )

    /// Removes a future tag from upcoming trips.
    func removeFutureTrackTag(
        _ tag: RPFutureTag,
        completion: @escaping (_ status: RPTagStatus, _ error: Error?) -> Void
    )

    /// Removes all future tags from upcoming trips.
    func removeAllFutureTrackTags(
        completion: @escaping (_ status: RPTagStatus, _ error: Error?) -> Void
    )
}
```

When the host app uses additional `RPEntry.instance.api` methods, add corresponding methods to the appropriate service rather than calling `RPEntry.instance.api` from screens or `TelematicsService`.

## Primary Flow Selection

Before implementing a new reusable Telematics service, identify the user's primary flow. Ask directly when it is not specified:

```text
Which primary tracking flow should be placed first in the Telematics service: automatic tracking, standard manual tracking without tags, standard manual tracking with tags, app-controlled persistent manual tracking without tags, app-controlled persistent manual tracking with tags, one-time persistent manual tracking without tags, or one-time persistent manual tracking with tags?
```

The service should still include all supported flows, but the user-requested primary flow must be implemented first and marked explicitly:

```swift
// MARK: - Primary Flow: App-controlled persistent manual tracking with tags
```

Put the other supported flows below it with separate sections:

```swift
// MARK: - Additional Flow: Automatic tracking
// MARK: - Additional Flow: Standard manual tracking without tags
// MARK: - Additional Flow: Standard manual tracking with tags
// MARK: - Additional Flow: App-controlled persistent manual tracking without tags
// MARK: - Additional Flow: One-time persistent manual tracking without tags
// MARK: - Additional Flow: One-time persistent manual tracking with tags
```

Keep the primary flow first so the app team can find and customize the business-critical path quickly. Do not duplicate SDK sequencing logic unnecessarily; share private helpers for common operations such as SDK enablement, future-tag cleanup, and persistent-mode reset. Keep device ID setup in its own public identity/session method.

## Tracking Concepts

Keep product flows separate from SDK modes:

- App-level flows: automatic tracking, standard manual tracking without tags, standard manual tracking with tags, app-controlled persistent manual tracking without tags, app-controlled persistent manual tracking with tags, one-time persistent manual tracking without tags, and one-time persistent manual tracking with tags.
- SDK tracking modes: `RPTrackingMode.standard` and `RPTrackingMode.persistent`, configured with `setTrackingMode`.

Standard, app-controlled persistent, and one-time persistent manual flows share the same lifecycle requirements. The differences are SDK mode ownership and whether future tags are added before starting. App-controlled persistent flows use `setTrackingMode(.persistent)` plus `startTracking()` and must restore `.standard` when needed. One-time persistent flows use `startTrackAsPersistent()`, which temporarily switches the SDK to `.persistent` and restores `.standard` automatically after `stopTracking()` or after the maximum persistent interval is reached.

## Sequencing Rules

Device identity setup:

```swift
try RPEntry.instance.setDeviceID(deviceId: deviceId)
```

Call this from login/session binding code before enabling automatic SDK collection or starting manual tracking. Do not repeat it inside every tracking start method.

Automatic tracking:

```swift
RPEntry.instance.setEnableSdk(true)
```

Disable SDK but keep device ID:

```swift
RPEntry.instance.setEnableSdk(false)
```

Logout:

```swift
RPEntry.instance.logout()
```

`logout()` clears the device ID. Use it only when the app truly wants the next enable flow to require setting the device ID again.

Standard manual tracking without tags:

```swift
RPEntry.instance.setEnableSdk(true)
RPEntry.instance.setTrackingMode(.standard)
RPEntry.instance.startTracking()
```

Manual stop:

```swift
RPEntry.instance.stopTracking()
RPEntry.instance.setEnableSdk(false)
```

Standard manual tracking with tags. Prefer this sequence inside an async service method:

```swift
RPEntry.instance.setEnableSdk(true)
RPEntry.instance.setTrackingMode(.standard)

let tag = RPFutureTag(tag: tagValue, source: source)
try await addFutureTrackTag(tag)
RPEntry.instance.startTracking()
```

Starting tracking before `addFutureTrackTag` completes is a race if the tag is required for that trip.

Manual stop with future-tag cleanup:

```swift
try await removeAllFutureTrackTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setEnableSdk(false)
```

If stopping must never be delayed by network/API cleanup, stop first and run tag cleanup as best-effort, but document that trade-off.

App-controlled persistent manual tracking without tags:

```swift
RPEntry.instance.setEnableSdk(true)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.setTrackingMode(.persistent)
RPEntry.instance.startTracking()
```

App-controlled persistent manual tracking with tags:

```swift
RPEntry.instance.setEnableSdk(true)

let tag = RPFutureTag(tag: tagValue, source: source)
try await addFutureTrackTag(tag)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.setTrackingMode(.persistent)
RPEntry.instance.startTracking()
```

App-controlled persistent stop:

```swift
try await removeAllFutureTrackTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setTrackingMode(.standard)
RPEntry.instance.setEnableSdk(false)
```

Always reset `.standard` after app-controlled persistent sessions unless the product explicitly wants future automatic sessions to remain persistent.

One-time persistent manual tracking without tags:

```swift
RPEntry.instance.setEnableSdk(true)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.startTrackAsPersistent()
```

One-time persistent manual tracking with tags:

```swift
RPEntry.instance.setEnableSdk(true)

let tag = RPFutureTag(tag: tagValue, source: source)
try await addFutureTrackTag(tag)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.startTrackAsPersistent()
```

One-time persistent stop:

```swift
try await removeAllFutureTrackTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setEnableSdk(false)
```

Do not call `setTrackingMode(.persistent)` before `startTrackAsPersistent()` and do not manually call `setTrackingMode(.standard)` after stopping a one-time persistent session; the SDK owns that transition.

If a reusable service exposes one shared manual stop method, track whether the active manual session was started with app-controlled persistent mode. Only call `setTrackingMode(.standard)` for that case.

## Persistent Tracking Constraints

Expected persistent tracking SDK surface:

- `RPTrackingMode.standard`
- `RPTrackingMode.persistent`
- `setMaxPersistentTrackingInterval(minutes:) throws`
- `startTrackAsPersistent()`
- allowed persistent interval: `5...600` minutes
- default persistent interval: `480` minutes

Verify these symbols in the installed SDK source before generating app code. Because `setMaxPersistentTrackingInterval(minutes:)` throws, service APIs should expose failure rather than hiding invalid values. `startTrackAsPersistent()` must be called on the main thread.

## Concurrency Guidance

The SDK exposes callback-based APIs. Preserve the required callback signatures in `TelematicsAPIService` and `TelematicsTagsService`; add private async helpers in `TelematicsService` only when sequencing needs `try await`.

Recommended shape:

```swift
final class DamoovTelematicsService: NSObject, TelematicsServicing {
    private let tagsService: TelematicsTagsServicing
    private var activeManualSessionUsesAppControlledPersistentMode = false

    init(tagsService: TelematicsTagsServicing = TelematicsTagsService()) {
        self.tagsService = tagsService
        super.init()
    }

    func configure() {
        RPEntry.instance.trackingStateDelegate = self
        RPEntry.instance.locationDelegate = self
        RPEntry.instance.accuracyAuthorizationDelegate = self
        RPEntry.instance.lowPowerModeDelegate = self
        RPEntry.instance.rtldDelegate = self
    }

    // MARK: - Primary Flow: App-controlled persistent manual tracking with tags

    func startPersistentManualTrackingWithTags(
        tags: [FutureTripTag],
        maxIntervalMinutes: Int
    ) async throws {
        RPEntry.instance.setEnableSdk(true)

        for tag in tags {
            try await addFutureTrackTag(RPFutureTag(tag: tag.value, source: tag.source))
        }

        try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: maxIntervalMinutes)
        RPEntry.instance.setTrackingMode(.persistent)
        RPEntry.instance.startTracking()
        activeManualSessionUsesAppControlledPersistentMode = true
    }

    func stopManualTracking(removeFutureTags: Bool) async throws {
        if removeFutureTags {
            try await removeAllFutureTrackTags()
        }

        RPEntry.instance.stopTracking()
        if activeManualSessionUsesAppControlledPersistentMode {
            RPEntry.instance.setTrackingMode(.standard)
            activeManualSessionUsesAppControlledPersistentMode = false
        }
        RPEntry.instance.setEnableSdk(false)
    }

    // MARK: - Additional Flow: Automatic tracking

    func enableAutomaticTracking() throws {
        RPEntry.instance.setEnableSdk(true)
    }

    // MARK: - Additional Flow: Standard manual tracking without tags

    func startStandardManualTracking() throws {
        RPEntry.instance.setEnableSdk(true)
        RPEntry.instance.setTrackingMode(.standard)
        RPEntry.instance.startTracking()
        activeManualSessionUsesAppControlledPersistentMode = false
    }

    // MARK: - Additional Flow: Standard manual tracking with tags

    func startStandardManualTrackingWithTags(tags: [FutureTripTag]) async throws {
        RPEntry.instance.setEnableSdk(true)
        RPEntry.instance.setTrackingMode(.standard)

        for tag in tags {
            try await addFutureTrackTag(RPFutureTag(tag: tag.value, source: tag.source))
        }

        RPEntry.instance.startTracking()
        activeManualSessionUsesAppControlledPersistentMode = false
    }

    // MARK: - Additional Flow: App-controlled persistent manual tracking without tags

    func startPersistentManualTracking(maxIntervalMinutes: Int) throws {
        RPEntry.instance.setEnableSdk(true)
        try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: maxIntervalMinutes)
        RPEntry.instance.setTrackingMode(.persistent)
        RPEntry.instance.startTracking()
        activeManualSessionUsesAppControlledPersistentMode = true
    }

    // MARK: - Additional Flow: One-time persistent manual tracking without tags

    func startOneTimePersistentManualTracking(maxIntervalMinutes: Int) throws {
        RPEntry.instance.setEnableSdk(true)
        try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: maxIntervalMinutes)
        RPEntry.instance.startTrackAsPersistent()
        activeManualSessionUsesAppControlledPersistentMode = false
    }

    // MARK: - Additional Flow: One-time persistent manual tracking with tags

    func startOneTimePersistentManualTrackingWithTags(
        tags: [FutureTripTag],
        maxIntervalMinutes: Int
    ) async throws {
        RPEntry.instance.setEnableSdk(true)

        for tag in tags {
            try await addFutureTrackTag(RPFutureTag(tag: tag.value, source: tag.source))
        }

        try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: maxIntervalMinutes)
        RPEntry.instance.startTrackAsPersistent()
        activeManualSessionUsesAppControlledPersistentMode = false
    }

    // MARK: - Identity and SDK state

    func setDeviceId(_ deviceId: String) throws {
        try RPEntry.instance.setDeviceID(deviceId: deviceId)
    }

    func logout() {
        RPEntry.instance.logout()
    }

    func disableSdk() {
        RPEntry.instance.setEnableSdk(false)
    }

    func isTracking() -> Bool {
        RPEntry.instance.isTracking()
    }

    func getDeviceIdRegistrationState() -> RPDeviceIdRegistrationState {
        RPEntry.instance.getDeviceIdRegistrationState()
    }

    func getTrackingState(completion: @escaping (_ state: RPTrackingState) -> Void) {
        RPEntry.instance.getTrackingState(completion: completion)
    }

    func sendCustomHeartbeat(_ reason: String) {
        RPEntry.instance.sendCustomHeartbeat(reason)
    }

    func uploadUnsentTrips() {
        RPEntry.instance.uploadUnsentTrips()
    }

    func getUnsentTripCount(completion: @escaping (_ unsentTripsCount: Int) -> Void) {
        RPEntry.instance.getUnsentTripCount(completion: completion)
    }

    func setAggressiveHeartbeats(_ aggressive: Bool) {
        RPEntry.instance.setAggressiveHeartbeats(aggressive)
    }

    func isAggressiveHeartbeats() -> Bool {
        RPEntry.instance.isAggressiveHeartbeats()
    }

    func isRTLDEnabled() -> Bool {
        RPEntry.instance.isRTLDEnabled()
    }

    func setAccidentDetectionEnabled(_ enabled: Bool) {
        RPEntry.instance.setAccidentDetectionEnabled(enabled)
    }

    func isAccidentDetectionEnabled() -> Bool {
        RPEntry.instance.isAccidentDetectionEnabled()
    }

    func setAccidentDetectionSensitivity(sensitivity: RPAccidentDetectionSensitivity) {
        RPEntry.instance.setAccidentDetectionSensitivity(sensitivity: sensitivity)
    }

    func getAccidentDetectionSensitivity() -> RPAccidentDetectionSensitivity {
        RPEntry.instance.getAccidentDetectionSensitivity()
    }

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
}

extension DamoovTelematicsService: RPTrackingStateListenerDelegate {
    func trackingStateChanged(_ state: Bool) {
        print("Telematics trackingStateChanged: \(state)")
    }
}

extension DamoovTelematicsService: RPLocationDelegate {
    func onLocationChanged(_ location: CLLocation) {
        print("Telematics onLocationChanged: \(location)")
    }

    func onNewEvents(_ events: [RPEventPoint]) {
        print("Telematics onNewEvents: \(events)")
    }
}

extension DamoovTelematicsService: RPAccuracyAuthorizationDelegate {
    func wrongAccuracyAuthorization() {
        print("Telematics wrongAccuracyAuthorization")
    }
}

extension DamoovTelematicsService: RPLowPowerModeDelegate {
    func lowPowerMode(_ state: Bool) {
        print("Telematics lowPowerMode: \(state)")
    }
}

extension DamoovTelematicsService: RPRTLDDelegate {
    func rtldColectedData() {
        print("Telematics rtldColectedData")
    }
}
```

Do not implement or assign `RPSpeedLimitDelegate` unless the user explicitly requests speed-limit behavior. It requires product-specific `speedLimit` and `timeThreshold` values.

Track/origin API facade:

```swift
final class TelematicsAPIService: TelematicsAPIServicing {
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

Tags API facade:

```swift
final class TelematicsTagsService: TelematicsTagsServicing {
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

If the app needs additional `RPEntry.instance.api` methods, add them to `TelematicsAPIService` for track/origin methods or `TelematicsTagsService` for tag/future-tag methods. Every public facade method must include an English documentation comment.

If the service methods are not already called from the main actor, call `startTracking()`, `startTrackAsPersistent()`, and `stopTracking()` through `await MainActor.run { ... }`. Do not expose SDK callbacks to UI code unless the host app architecture explicitly requires callback APIs.

## Architecture Checks

When reviewing an integration, flag these issues:

- Direct `RPEntry.instance` calls outside the service/lifecycle adapter.
- Direct track/origin `RPEntry.instance.api` calls outside `TelematicsAPIService`.
- Direct tag/future-tag `RPEntry.instance.api` calls outside `TelematicsTagsService`.
- Missing full lifecycle forwarding from `references/ios/integration-reference.md`.
- Duplicated foreground/background forwarding between SceneDelegate and AppDelegate paths.
- Manual tracking started before required future tag completion.
- `logout()` used when the app only meant to temporarily disable SDK collection.
- App-controlled persistent mode not reset to `.standard`.
- One-time persistent flow implemented with manual `setTrackingMode(.persistent)` / `setTrackingMode(.standard)` instead of `startTrackAsPersistent()`.
- `setMaxPersistentTrackingInterval(minutes:)` called without error handling.
- Future-tag cleanup after `setEnableSdk(false)` when cleanup depends on SDK/API availability.
- Deprecated API from `references/ios/api-migration.md`.
