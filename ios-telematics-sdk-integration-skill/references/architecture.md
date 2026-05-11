# Recommended App Architecture

## Decision

Wrap `TelematicsSDK` in an app-owned service/facade. Do not scatter `RPEntry.instance` calls across screens, reducers, interactors, or view models.

This is a recommendation, not a verified SDK requirement. The reason is architectural: `RPEntry` is a global singleton with mutable state, lifecycle hooks, async tag operations, tracking modes, and deprecated API migration concerns. A facade gives the app one place to sequence calls, map app concepts to SDK concepts, and test business decisions without invoking the SDK everywhere.

## Suggested Boundaries

Use names that match the host app conventions, but keep these responsibilities separate:

- `TelematicsService`: owns `RPEntry` lifecycle, configuration, tracking, and delegate calls, excluding `RPEntry.instance.api`.
- `TelematicsAPIService`: owns all `RPEntry.instance.api` calls, including track, trip-tag, and future-tag APIs.
- `TelematicsLifecycleAdapter`: centralizes `UIApplicationDelegate` / `UISceneDelegate` forwarding, but is called by standard app delegates named `AppDelegate` and `SceneDelegate`.
- `TrackingSessionRequest`: app-level input for starting tracking, e.g. device ID, flow, optional tag, optional persistent interval.
- `TrackingFlow`: app-level flow such as automatic enablement, standard manual start/stop, or persistent manual start/stop, with or without tags.
- `TrackingMode`: app-level enum that maps to SDK modes `.standard` or `.persistent`.
- `TelematicsError`: app-level error mapping for SDK/API failures.

Avoid putting business decisions into `AppDelegate`. App/scene delegates should initialize and forward lifecycle only. Use standard delegate names: `AppDelegate` and `SceneDelegate`; do not create SDK-specific delegate class names such as `DamoovAppDelegate` or `DamoovSceneDelegate`.

`TelematicsService` should receive a `TelematicsAPIService` dependency and must not call `RPEntry.instance.api` directly. This keeps SDK singleton/lifecycle/tracking calls separate from network-backed track/tag API wrappers.

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
    func configure()
    func enableAutomaticTracking(deviceId: String)
    func disableSdk(keepDeviceId: Bool)
    func startStandardManualTracking(deviceId: String)
    func startStandardManualTrackingWithTags(deviceId: String, tags: [FutureTripTag]) async throws
    func startPersistentManualTracking(deviceId: String, maxIntervalMinutes: Int) throws
    func startPersistentManualTrackingWithTags(deviceId: String, tags: [FutureTripTag], maxIntervalMinutes: Int) async throws
    func stopManualTracking(removeFutureTags: Bool) async throws
    func isTracking() -> Bool
}
```

Use `keepDeviceId: true` for `setEnableSdk(false)`. Use `keepDeviceId: false` only when logout semantics are intended.

`TelematicsAPIService` should wrap every `RPEntry.instance.api` method the integration exposes. For a full reusable integration, include wrappers for the track APIs, track-tag APIs, and future-tag APIs listed in `common-sdk-surface.md`. At minimum for this skill's generated tracking flows, it must expose async wrappers for future tags:

```swift
protocol TelematicsAPIServicing {
    func addFutureTrackTag(_ tag: RPFutureTag) async throws
    func removeAllFutureTrackTags() async throws
}
```

When the host app uses trip history, shared tracks, origins, or track-specific tags, add corresponding methods to this API service rather than calling `RPEntry.instance.api` from screens or `TelematicsService`.

## Primary Flow Selection

Before implementing a new reusable Telematics service, identify the user's primary flow. Ask directly when it is not specified:

```text
Which primary tracking flow should be placed first in the Telematics service: automatic tracking, standard manual tracking without tags, standard manual tracking with tags, persistent manual tracking without tags, or persistent manual tracking with tags?
```

The service should still include all supported flows, but the user-requested primary flow must be implemented first and marked explicitly:

```swift
// MARK: - Primary Flow: Persistent manual tracking with tags
```

Put the other supported flows below it with separate sections:

```swift
// MARK: - Additional Flow: Automatic tracking
// MARK: - Additional Flow: Standard manual tracking without tags
// MARK: - Additional Flow: Standard manual tracking with tags
// MARK: - Additional Flow: Persistent manual tracking without tags
```

Keep the primary flow first so the app team can find and customize the business-critical path quickly. Do not duplicate SDK sequencing logic unnecessarily; share private helpers for common operations such as device ID setup, SDK enablement, future-tag cleanup, and persistent-mode reset.

## Tracking Concepts

Keep product flows separate from SDK modes:

- App-level flows: automatic tracking, standard manual tracking without tags, standard manual tracking with tags, persistent manual tracking without tags, and persistent manual tracking with tags.
- SDK tracking modes: `RPTrackingMode.standard` and `RPTrackingMode.persistent`, configured with `setTrackingMode`.

Standard and persistent manual flows share the same lifecycle requirements. The difference is the SDK tracking mode and whether future tags are added before `startTracking()`.

## Sequencing Rules

Automatic tracking:

```swift
RPEntry.instance.setDeviceID(deviceId: deviceId)
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
RPEntry.instance.setDeviceID(deviceId: deviceId)
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
RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
RPEntry.instance.setTrackingMode(.standard)

let tag = RPFutureTag(tag: tagValue, source: source)
try await apiService.addFutureTrackTag(tag)
RPEntry.instance.startTracking()
```

Starting tracking before `addFutureTrackTag` completes is a race if the tag is required for that trip.

Manual stop with future-tag cleanup:

```swift
try await apiService.removeAllFutureTrackTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setEnableSdk(false)
```

If stopping must never be delayed by network/API cleanup, stop first and run tag cleanup as best-effort, but document that trade-off.

Persistent manual tracking without tags:

```swift
RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.setTrackingMode(.persistent)
RPEntry.instance.startTracking()
```

Persistent manual tracking with tags:

```swift
RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)

let tag = RPFutureTag(tag: tagValue, source: source)
try await apiService.addFutureTrackTag(tag)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.setTrackingMode(.persistent)
RPEntry.instance.startTracking()
```

Persistent stop:

```swift
try await apiService.removeAllFutureTrackTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setTrackingMode(.standard)
RPEntry.instance.setEnableSdk(false)
```

Always reset `.standard` after app-controlled persistent sessions unless the product explicitly wants future automatic sessions to remain persistent.

## Persistent Tracking Constraints

Current local SDK source verifies:

- `RPTrackingMode.standard`
- `RPTrackingMode.persistent`
- `setMaxPersistentTrackingInterval(minutes:) throws`
- allowed persistent interval: `5...600` minutes
- default persistent interval: `480` minutes

Because `setMaxPersistentTrackingInterval(minutes:)` throws, service APIs should expose failure rather than hiding invalid values.

## Concurrency Guidance

The SDK exposes callback-based APIs, but the app service should prefer Swift `async`/`await`. Keep `RPEntry.instance.api` callback APIs private to `TelematicsAPIService` so callers can express sequencing with `try await`.

Recommended shape:

```swift
final class DamoovTelematicsService: NSObject, TelematicsServicing {
    private let apiService: TelematicsAPIServicing

    init(apiService: TelematicsAPIServicing = TelematicsAPIService()) {
        self.apiService = apiService
        super.init()
    }

    func configure() {
        RPEntry.instance.trackingStateDelegate = self
        RPEntry.instance.locationDelegate = self
        RPEntry.instance.accuracyAuthorizationDelegate = self
        RPEntry.instance.lowPowerModeDelegate = self
        RPEntry.instance.rtldDelegate = self
    }

    // MARK: - Primary Flow: Persistent manual tracking with tags

    func startPersistentManualTrackingWithTags(
        deviceId: String,
        tags: [FutureTripTag],
        maxIntervalMinutes: Int
    ) async throws {
        RPEntry.instance.setDeviceID(deviceId: deviceId)
        RPEntry.instance.setEnableSdk(true)

        for tag in tags {
            try await apiService.addFutureTrackTag(RPFutureTag(tag: tag.value, source: tag.source))
        }

        try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: maxIntervalMinutes)
        RPEntry.instance.setTrackingMode(.persistent)
        RPEntry.instance.startTracking()
    }

    func stopManualTracking(removeFutureTags: Bool) async throws {
        if removeFutureTags {
            try await apiService.removeAllFutureTrackTags()
        }

        RPEntry.instance.stopTracking()
        RPEntry.instance.setTrackingMode(.standard)
        RPEntry.instance.setEnableSdk(false)
    }

    // MARK: - Additional Flow: Automatic tracking

    func enableAutomaticTracking(deviceId: String) {
        RPEntry.instance.setDeviceID(deviceId: deviceId)
        RPEntry.instance.setEnableSdk(true)
    }

    // MARK: - Additional Flow: Standard manual tracking without tags

    func startStandardManualTracking(deviceId: String) {
        RPEntry.instance.setDeviceID(deviceId: deviceId)
        RPEntry.instance.setEnableSdk(true)
        RPEntry.instance.setTrackingMode(.standard)
        RPEntry.instance.startTracking()
    }

    // MARK: - Additional Flow: Standard manual tracking with tags

    func startStandardManualTrackingWithTags(deviceId: String, tags: [FutureTripTag]) async throws {
        RPEntry.instance.setDeviceID(deviceId: deviceId)
        RPEntry.instance.setEnableSdk(true)
        RPEntry.instance.setTrackingMode(.standard)

        for tag in tags {
            try await apiService.addFutureTrackTag(RPFutureTag(tag: tag.value, source: tag.source))
        }

        RPEntry.instance.startTracking()
    }

    // MARK: - Additional Flow: Persistent manual tracking without tags

    func startPersistentManualTracking(deviceId: String, maxIntervalMinutes: Int) throws {
        RPEntry.instance.setDeviceID(deviceId: deviceId)
        RPEntry.instance.setEnableSdk(true)
        try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: maxIntervalMinutes)
        RPEntry.instance.setTrackingMode(.persistent)
        RPEntry.instance.startTracking()
    }

    // MARK: - Additional Flow: SDK disablement

    func disableSdk(keepDeviceId: Bool) {
        if keepDeviceId {
            RPEntry.instance.setEnableSdk(false)
        } else {
            RPEntry.instance.logout()
        }
    }

    func isTracking() -> Bool {
        RPEntry.instance.isTracking()
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

API service callback wrappers:

```swift
final class TelematicsAPIService: TelematicsAPIServicing {
    func addFutureTrackTag(_ tag: RPFutureTag) async throws {
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

    func removeAllFutureTrackTags() async throws {
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
}
```

If the app needs additional `RPEntry.instance.api` methods, add them to `TelematicsAPIService` with the same callback-to-async policy.

Callback wrapper pattern:

```swift
func addFutureTrackTag(_ tag: RPFutureTag) async throws {
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
```

If the service methods are not already called from the main actor, call `startTracking()` and `stopTracking()` through `await MainActor.run { ... }`. Do not expose SDK callbacks to UI code unless the host app architecture explicitly requires callback APIs.

## Architecture Checks

When reviewing an integration, flag these issues:

- Direct `RPEntry.instance` calls outside the service/lifecycle adapter.
- Direct `RPEntry.instance.api` calls outside `TelematicsAPIService`.
- Missing full lifecycle forwarding from `references/integration-reference.md`.
- Duplicated foreground/background forwarding between SceneDelegate and AppDelegate paths.
- Manual tracking started before required future tag completion.
- `logout()` used when the app only meant to temporarily disable SDK collection.
- Persistent mode not reset to `.standard`.
- `setMaxPersistentTrackingInterval(minutes:)` called without error handling.
- Future-tag cleanup after `setEnableSdk(false)` when cleanup depends on SDK/API availability.
- Deprecated API from `references/api-migration.md`.
