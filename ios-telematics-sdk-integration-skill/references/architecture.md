# Recommended App Architecture

## Decision

Wrap `TelematicsSDK` in an app-owned service/facade. Do not scatter `RPEntry.instance` calls across screens, reducers, interactors, or view models.

This is a recommendation, not a verified SDK requirement. The reason is architectural: `RPEntry` is a global singleton with mutable state, lifecycle hooks, async tag operations, tracking modes, and deprecated API migration concerns. A facade gives the app one place to sequence calls, map app concepts to SDK concepts, and test business decisions without invoking the SDK everywhere.

## Suggested Boundaries

Use names that match the host app conventions, but keep these responsibilities separate:

- `TelematicsService`: owns all `RPEntry` calls.
- `TelematicsLifecycleAdapter`: centralizes `UIApplicationDelegate` / `UISceneDelegate` forwarding, but is called by standard app delegates named `AppDelegate` and `SceneDelegate`.
- `TrackingSessionRequest`: app-level input for starting tracking, e.g. device ID, flow, optional tag, optional persistent interval.
- `TrackingFlow`: app-level flow such as automatic enablement or manual trip start/stop.
- `TrackingMode`: app-level enum that maps to SDK modes `.standard` or `.persistent`.
- `TelematicsError`: app-level error mapping for SDK/API failures.

Avoid putting business decisions into `AppDelegate`. App/scene delegates should initialize and forward lifecycle only. Use standard delegate names: `AppDelegate` and `SceneDelegate`; do not create SDK-specific delegate class names such as `DamoovAppDelegate` or `DamoovSceneDelegate`.

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
    func startManualTracking(deviceId: String, tag: FutureTripTag?, persistent: PersistentTrackingOptions?) async throws
    func stopManualTracking(removeFutureTags: Bool) async throws
    func isTracking() -> Bool
}
```

Use `keepDeviceId: true` for `setEnableSdk(false)`. Use `keepDeviceId: false` only when logout semantics are intended.

## Primary Flow Selection

Before implementing a new reusable Telematics service, identify the user's primary flow. Ask directly when it is not specified:

```text
Which primary tracking flow should be placed first in the Telematics service: automatic tracking, manual tracking, manual tracking with required future tags, or manual tracking with persistent SDK mode?
```

The service should still include all supported flows, but the user-requested primary flow must be implemented first and marked explicitly:

```swift
// MARK: - Primary Flow: Manual tracking with persistent SDK mode
```

Put the other supported flows below it with separate sections:

```swift
// MARK: - Additional Flow: Automatic tracking
// MARK: - Additional Flow: Manual tracking
// MARK: - Additional Flow: Manual tracking with required future tags
```

Keep the primary flow first so the app team can find and customize the business-critical path quickly. Do not duplicate SDK sequencing logic unnecessarily; share private helpers for common operations such as device ID setup, SDK enablement, future-tag cleanup, and persistent-mode reset.

## Tracking Concepts

Keep product flows separate from SDK modes:

- App-level flows: automatic tracking enablement, manual tracking without tags, manual tracking with required future tags, and manual tracking with persistent SDK mode.
- SDK tracking modes: `RPTrackingMode.standard` and `RPTrackingMode.persistent`, configured with `setTrackingMode`.

Persistent SDK mode can be used inside a manual flow, but do not model it as a separate lifecycle flow unless the host app product requirements treat it that way.

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

Manual tracking without tags:

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

Manual tracking with required future tag. Prefer this sequence inside an async service method:

```swift
RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)

let tag = RPFutureTag(tag: tagValue, source: source)
try await addFutureTag(tag)
RPEntry.instance.startTracking()
```

Starting tracking before `addFutureTrackTag` completes is a race if the tag is required for that trip.

Manual stop with future-tag cleanup:

```swift
try await removeAllFutureTags()
RPEntry.instance.stopTracking()
RPEntry.instance.setEnableSdk(false)
```

If stopping must never be delayed by network/API cleanup, stop first and run tag cleanup as best-effort, but document that trade-off.

Persistent manual tracking with tag:

```swift
RPEntry.instance.setDeviceID(deviceId: deviceId)
RPEntry.instance.setEnableSdk(true)

let tag = RPFutureTag(tag: tagValue, source: source)
try await addFutureTag(tag)
try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: minutes)
RPEntry.instance.setTrackingMode(.persistent)
RPEntry.instance.startTracking()
```

Persistent stop:

```swift
try await removeAllFutureTags()
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

The SDK exposes callback-based APIs, but the app service should prefer Swift `async`/`await`. Keep callback APIs private to the service wrapper so callers can express sequencing with `try await`.

Recommended shape:

```swift
@MainActor
final class DamoovTelematicsService: TelematicsServicing {
    // MARK: - Primary Flow: Manual tracking with persistent SDK mode

    func startManualTracking(
        deviceId: String,
        tag: FutureTripTag?,
        persistent: PersistentTrackingOptions?
    ) async throws {
        RPEntry.instance.setDeviceID(deviceId: deviceId)
        RPEntry.instance.setEnableSdk(true)

        if let tag {
            try await addFutureTag(RPFutureTag(tag: tag.value, source: tag.source))
        }

        if let persistent {
            try RPEntry.instance.setMaxPersistentTrackingInterval(minutes: persistent.maxIntervalMinutes)
            RPEntry.instance.setTrackingMode(.persistent)
        }

        RPEntry.instance.startTracking()
    }

    func stopManualTracking(removeFutureTags: Bool) async throws {
        if removeFutureTags {
            try await removeAllFutureTags()
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

    // MARK: - Additional Flow: SDK disablement

    func disableSdk(keepDeviceId: Bool) {
        if keepDeviceId {
            RPEntry.instance.setEnableSdk(false)
        } else {
            RPEntry.instance.logout()
        }
    }
}
```

Callback wrappers:

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

If the service is not `@MainActor`, call `startTracking()` and `stopTracking()` through `await MainActor.run { ... }`. Do not expose SDK callbacks to UI code unless the host app architecture explicitly requires callback APIs.

## Architecture Checks

When reviewing an integration, flag these issues:

- Direct `RPEntry.instance` calls outside the service/lifecycle adapter.
- Missing full lifecycle forwarding from `references/integration-reference.md`.
- Duplicated foreground/background forwarding between SceneDelegate and AppDelegate paths.
- Manual tracking started before required future tag completion.
- `logout()` used when the app only meant to temporarily disable SDK collection.
- Persistent mode not reset to `.standard`.
- `setMaxPersistentTrackingInterval(minutes:)` called without error handling.
- Future-tag cleanup after `setEnableSdk(false)` when cleanup depends on SDK/API availability.
- Deprecated API from `references/api-migration.md`.
