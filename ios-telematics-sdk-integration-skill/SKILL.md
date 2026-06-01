---
name: ios-telematics-sdk-integration-skill
description: Use when designing, integrating, migrating, reviewing, or debugging Damoov TelematicsSDK for iOS apps, especially SPM setup, service-layer architecture, RPEntry lifecycle, automatic/manual tracking flows, standard/persistent SDK tracking modes, one-time persistent manual tracking, permissions, future tags, trip tags, and replacing deprecated API from current SDK source.
metadata:
  short-description: Damoov iOS TelematicsSDK integration
---

# iOS Telematics SDK Integration

## Operating Rule

Treat published Damoov documentation as a baseline, not as the source of truth for unreleased SDK versions. Before changing app code, verify the currently installed SDK API from local source or package artifacts.

If this path does not exist, inspect the SDK version installed in the target app and use the linked docs only for concepts and setup flow.

## Workflow

1. Inspect the target iOS app first:
   - Dependency manager: use SPM. Check `Package.swift`, `Package.resolved`, `.xcodeproj`, and `.xcworkspace`.
   - App lifecycle files: `AppDelegate`, `SceneDelegate`, SwiftUI `App`.
   - Minimum iOS deployment target. Check `IPHONEOS_DEPLOYMENT_TARGET`, project settings, package/platform declarations, and app target metadata where available.
   - Permissions: `Info.plist`, entitlements, Background Modes.
   - Existing SDK usage: `rg -n "RPEntry|TelematicsSDK|RPPermissions|RPTag|RPFutureTag|tracking"`.

2. Use Swift Package Manager for TelematicsSDK dependency integration.

3. Verify the latest exact SDK version from the official SPM repository before editing dependencies:
   - `git ls-remote --tags --refs https://github.com/Mobile-Telematics/telematicsSDK-iOS-new-SPM.git`
   - Use the latest semantic version tag exactly with `.package(..., from: "<latest-spm-tag>")`.

4. Inspect current SDK API before relying on all public APIs (RPEntry, RPAPIEntry etc).

5. Prefer an app-owned reusable service/facade around `RPEntry` instead of calling the SDK directly from screens or view models. Load `references/ios/architecture.md` before designing or changing integration structure.

6. Before implementing a new Telematics service, ask the user which primary tracking flow should be placed first in the service:
   - automatic tracking
   - standard manual tracking without tags
   - standard manual tracking with tags
   - app-controlled persistent manual tracking without tags
   - app-controlled persistent manual tracking with tags
   - one-time persistent manual tracking without tags
   - one-time persistent manual tracking with tags
   If the user has already stated the primary flow in the request or existing product code makes it unambiguous, use that flow without asking again.

7. Replace deprecated public API with current API. Load `references/ios/api-migration.md` when touching `RPEntry` methods or properties.

8. For public SDK APIs that are common to every integration, load `references/ios/common-sdk-surface.md`. This includes iOS project setup, lifecycle wiring, `RPEntry` status/config/permission methods, delegates, `TelematicsAPIService`, and `TelematicsTagsService`.

9. For tracking flow sequences, SDK tracking modes, and tags, load `references/ios/integration-reference.md`.

10. Implement separate facades for `RPEntry.instance.api`:
   - `TelematicsAPIService` for track/origin APIs.
   - `TelematicsTagsService` for track tags and future tags.
   `TelematicsService` must use `TelematicsTagsService` for tagged tracking flows instead of calling `RPEntry.instance.api` directly.

11. Implement the service with all supported flows, but order methods so the user's primary flow appears first. Keep changes outside the integration surface minimal. Do not add dependencies beyond TelematicsSDK unless the user explicitly approves.

12. Validate with the narrowest available check:
   - Prefer `xcodebuild` for the app target/scheme if discoverable.
   - If the project cannot build locally, at least run `swiftc`-independent checks such as `rg` for deprecated symbols and explain the limit.

## Coding Rules

- Call `RPEntry.initializeSDK()` once before `RPEntry.instance`.
- Use standard lifecycle delegate names: `AppDelegate` and `SceneDelegate`. Do not invent names such as `DamoovAppDelegate` or `DamoovSceneDelegate`.
- If both `AppDelegate` and `SceneDelegate` are absent and the app's minimum iOS version is 13.0 or newer, implement standard `AppDelegate` and `SceneDelegate` lifecycle delegates and wire them into the app.
- In `application(_:didFinishLaunchingWithOptions:)`, `RPEntry.initializeSDK()` must be the first executable statement and must happen before `RPEntry.instance.application(application, didFinishLaunchingWithOptions: launchOptions ?? [:])`.
- Wire lifecycle forwarding from the app's actual `AppDelegate`, `SceneDelegate`, or SwiftUI lifecycle bridge into the Telematics lifecycle adapter/service. Do not only implement lifecycle methods inside the facade without calling them from app lifecycle entry points.
- Lifecycle forwarding methods are mandatory. Do not guard them with `RPEntry.isInitialized`, device ID checks, `hasConfiguredDeviceId`, or similar app-side conditions; the SDK already handles missing device ID or disabled state where applicable.
- Use `try setDeviceID(deviceId:)` / `getDeviceId()` instead of `virtualDeviceToken`; expose or handle the thrown error instead of ignoring invalid device IDs.
- Keep device identity separate from tracking flows and SDK tracking modes. Generated facades should expose an explicit `setDeviceId(...)` method for login/session binding and an explicit `logout()` method for user logout/account removal. Do not make start/stop tracking methods accept or reset the device ID.
- Use method-style setters/getters introduced in `RPEntry` instead of deprecated properties.
- Put track/origin `RPEntry.instance.api` calls inside `TelematicsAPIService`.
- Put track-tag and future-tag `RPEntry.instance.api` calls inside `TelematicsTagsService`.
- Do not mix network/tag/track API wrappers into `TelematicsService`.
- Call `startTracking()`, `startTrackAsPersistent()`, and `stopTracking()` on the main thread.
- Treat automatic/manual tracking as app-level flows; treat `.standard`/`.persistent` as SDK tracking modes configured with `setTrackingMode`.
- For app-controlled persistent SDK mode, use `setTrackingMode(.persistent)` plus `startTracking()` and switch back with `setTrackingMode(.standard)` when business logic requires it.
- For one-time persistent manual tracking, use `startTrackAsPersistent()`. The SDK switches `trackingMode` to `.persistent` at start and automatically restores `.standard` after `stopTracking()` or when `setMaxPersistentTrackingInterval` is reached; do not add a redundant manual mode reset for that flow.
- If one service method stops all manual flows, track whether the active session was app-controlled persistent before deciding to call `setTrackingMode(.standard)`.
- In reusable Telematics services, implement all supported flows but place the user-requested primary flow first and mark it with `// MARK: - Primary Flow: <name requested by user>`.
- Place the remaining flows below with separate `// MARK: - Additional Flow: <flow name>` sections.
- Supported flows are: automatic tracking, standard manual tracking without tags, standard manual tracking with tags, app-controlled persistent manual tracking without tags, app-controlled persistent manual tracking with tags, one-time persistent manual tracking without tags, and one-time persistent manual tracking with tags.
- For cross-platform manual tagged flows, treat "with tags" as future tags attached before the upcoming manually started trip. iOS also exposes processed-trip tag APIs through `TelematicsTagsService`; keep those as separate post-trip operations rather than mixing them into tracking start flows.
- If a future tag is required for a manually started trip, add the tag and handle the completion before starting tracking; otherwise document the race.
- Remove future tags before disabling the SDK when cleanup depends on SDK/API availability.
- Use `setEnableSdk(false)` for temporary SDK collection disablement. Use `logout()` only for real logout/account-removal semantics because it clears the device ID.
- Required facade methods should preserve the SDK callback signatures unless the host app explicitly prefers async overloads. Async convenience methods may be added, but they must not replace the required callback methods.
- Add English documentation comments to every public method in `TelematicsService`, `TelematicsAPIService`, and `TelematicsTagsService`.
- Dispatch SDK completion handlers back to the main queue before updating UI; tag API completions are not guaranteed to arrive on the main thread.
- Do not silently ignore SDK errors in completions.
- `TelematicsService` must implement and assign the public SDK delegates except `speedLimitDelegate`: `RPTrackingStateListenerDelegate`, `RPLocationDelegate`, `RPAccuracyAuthorizationDelegate`, `RPLowPowerModeDelegate`, and `RPRTLDDelegate`.
- Delegate methods generated by the skill should only call `print(...)`. Do not add business logic, state mutation, analytics, or UI updates to these delegate callbacks unless the user explicitly asks for it.
- Do not assign or implement `speedLimitDelegate` unless the user explicitly requests speed-limit behavior, because it requires app-specific threshold values.

## Official Docs

Use these docs for baseline integration behavior, but check current source before copying code:

- https://docs.damoov.com/docs/-download-the-sdk-and-install-it-in-your-environment
- https://docs.damoov.com/docs/methods-for-ios-app
- https://docs.damoov.com/docs/tracking
- https://docs.damoov.com/docs/ios-sdk-incoming-tags
- https://docs.damoov.com/docs/trip-tag-for-ios-app
