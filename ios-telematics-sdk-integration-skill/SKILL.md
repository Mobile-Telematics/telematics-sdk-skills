---
name: ios-telematics-sdk-integration-skill
description: Use when designing, integrating, migrating, reviewing, or debugging Damoov TelematicsSDK for iOS apps, especially SPM setup, service-layer architecture, RPEntry lifecycle, automatic/manual tracking flows, standard/persistent SDK tracking modes, permissions, future tags, trip tags, and replacing deprecated API from current sdk-ios source.
metadata:
  short-description: Damoov iOS TelematicsSDK integration
---

# iOS Telematics SDK Integration

## Operating Rule

Treat published Damoov documentation as a baseline, not as the source of truth for unreleased SDK versions. Before changing app code, verify the currently installed SDK API from local source or package artifacts.

If this path does not exist, inspect the SDK version installed in the target app and use the linked docs only for concepts and setup flow.

## Workflow

1. Inspect the target iOS app first:
   - Dependency manager: prefer SPM. Check `Package.swift`, `Package.resolved`, `.xcodeproj`, `.xcworkspace`, then legacy `Podfile`, `Podfile.lock`.
   - App lifecycle files: `AppDelegate`, `SceneDelegate`, SwiftUI `App`.
   - Minimum iOS deployment target. Check `IPHONEOS_DEPLOYMENT_TARGET`, project settings, package/platform declarations, and app target metadata where available.
   - Permissions: `Info.plist`, entitlements, Background Modes.
   - Existing SDK usage: `rg -n "RPEntry|TelematicsSDK|RPPermissions|RPTag|RPFutureTag|tracking"`.

2. Verify the exact latest SPM version before editing dependencies:
   - `git ls-remote --tags --refs https://github.com/Mobile-Telematics/telematicsSDK-iOS-new-SPM.git`
   - Use the highest semantic version tag exactly, for example `from: "7.0.3"` when `7.0.3` is latest.

3. Inspect current SDK API before relying on all public APIs (RPEntry, RPAPIEntry etc).

4. Prefer an app-owned reusable service/facade around `RPEntry` instead of calling the SDK directly from screens or view models. Load `references/architecture.md` before designing or changing integration structure.

5. Before implementing a new Telematics service, ask the user which primary tracking flow should be placed first in the service:
   - automatic tracking
   - manual tracking
   - manual tracking with required future tags
   - manual tracking with persistent SDK mode
   If the user has already stated the primary flow in the request or existing product code makes it unambiguous, use that flow without asking again.

6. Replace deprecated public API with current API. Load `references/api-migration.md` when touching `RPEntry` methods or properties.

7. For public SDK APIs that are common to every integration, load `references/common-sdk-surface.md`. This includes iOS project setup, lifecycle wiring, `RPEntry` status/config/permission methods, delegates, and `RPEntry.instance.api` (`RPAPIEntry`) methods.

8. For tracking flow sequences, SDK tracking modes, and tags, load `references/integration-reference.md`.

9. Implement the service with all supported flows, but order methods so the user's primary flow appears first. Keep changes outside the integration surface minimal. Do not add dependencies beyond TelematicsSDK unless the user explicitly approves.

10. Validate with the narrowest available check:
   - Prefer `xcodebuild` for the app target/scheme if discoverable.
   - If the project cannot build locally, at least run `swiftc`-independent checks such as `rg` for deprecated symbols and explain the limit.

## Coding Rules

- Call `RPEntry.initializeSDK()` once before `RPEntry.instance`.
- Use standard lifecycle delegate names: `AppDelegate` and `SceneDelegate`. Do not invent names such as `DamoovAppDelegate` or `DamoovSceneDelegate`.
- If both `AppDelegate` and `SceneDelegate` are absent and the app's minimum iOS version is 13.0 or newer, implement standard `AppDelegate` and `SceneDelegate` lifecycle delegates and wire them into the app.
- In `application(_:didFinishLaunchingWithOptions:)`, `RPEntry.initializeSDK()` must be the first executable statement and must happen before `RPEntry.instance.application(application, didFinishLaunchingWithOptions: launchOptions ?? [:])`.
- Wire lifecycle forwarding from the app's actual `AppDelegate`, `SceneDelegate`, or SwiftUI lifecycle bridge into the Telematics lifecycle adapter/service. Do not only implement lifecycle methods inside the facade without calling them from app lifecycle entry points.
- Lifecycle forwarding methods are mandatory. Do not guard them with `RPEntry.isInitialized`, device ID checks, `hasConfiguredDeviceId`, or similar app-side conditions; the SDK already handles missing device ID or disabled state where applicable.
- Use `setDeviceID(deviceId:)` / `getDeviceId()` instead of `virtualDeviceToken`.
- Use method-style setters/getters introduced in `RPEntry` instead of deprecated properties.
- Call `startTracking()` / `stopTracking()` on the main thread.
- Treat automatic/manual tracking as app-level flows; treat `.standard`/`.persistent` as SDK tracking modes configured with `setTrackingMode`.
- For persistent SDK mode, prefer `setTrackingMode(.persistent)` plus `startTracking()`; switch back with `setTrackingMode(.standard)` when business logic requires it.
- In reusable Telematics services, implement all supported flows but place the user-requested primary flow first and mark it with `// MARK: - Primary Flow: <name requested by user>`.
- Place the remaining flows below with separate `// MARK: - Additional Flow: <flow name>` sections.
- If a future tag is required for a manually started trip, add the tag and handle the completion before starting tracking; otherwise document the race.
- Remove future tags before disabling the SDK when cleanup depends on SDK/API availability.
- Prefer Swift `async`/`await` in the app-owned service API. Keep SDK completion-handler calls private to the service wrapper.
- Dispatch SDK completion handlers back to the main queue before updating UI; tag API completions are not guaranteed to arrive on the main thread.
- Do not silently ignore SDK errors in completions.

## Official Docs

Use these docs for baseline integration behavior, but check current source before copying code:

- https://docs.damoov.com/docs/-download-the-sdk-and-install-it-in-your-environment
- https://docs.damoov.com/docs/methods-for-ios-app
- https://docs.damoov.com/docs/tracking
- https://docs.damoov.com/docs/ios-sdk-incoming-tags
- https://docs.damoov.com/docs/trip-tag-for-ios-app
