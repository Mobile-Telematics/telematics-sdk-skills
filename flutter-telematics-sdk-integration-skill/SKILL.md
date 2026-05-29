---
name: flutter-telematics-sdk-integration-skill
description: Use when designing, integrating, migrating, reviewing, or debugging the Damoov Telematics SDK Flutter plugin in Flutter apps, especially pubspec setup, TrackingApi usage, Android Gradle/manifest/proguard setup, iOS Info.plist/AppDelegate/SceneDelegate setup, permission wizard wiring, automatic/manual tracking flows, persistent tracking, future tags, platform-specific APIs, and validation against the Flutter plugin source.
---

# Flutter Telematics SDK Integration

## Operating Rule

Treat the Flutter plugin source and the target app's lockfiles as the source of truth. Public docs and README examples are baseline guidance; verify the plugin API from `lib/src/tracking_api.dart`, native bridge code, and installed package version before editing app code.

The reference plugin source used to build this skill is the public repository [Mobile-Telematics/telematicsSDK-demoapp-flutter-](https://github.com/Mobile-Telematics/telematicsSDK-demoapp-flutter-), verified at version `1.1.1`, with Flutter `>=3.41.0`, Dart `>=3.11.0 <4.0.0`, Android native SDK `com.telematicssdk:tracking:4.0.0`, and iOS SPM SDK `7.1.0`. Treat those as API reference checkpoints, not install targets. Always verify and install the latest compatible Flutter plugin version before changing dependencies.

## Workflow

1. Inspect the target Flutter app first:
   - `pubspec.yaml`, `pubspec.lock`, `.flutter-plugins-dependencies`.
   - `android/build.gradle`, `android/app/build.gradle`, `android/app/src/main/AndroidManifest.xml`, proguard files, app `Application` class.
   - `ios/Podfile`, `ios/Runner/Info.plist`, `ios/Runner/AppDelegate.swift`, scene setup, deployment target.
   - Existing usage: `rg -n "telematics_sdk|TrackingApi|setDeviceID|setEnableSdk|startManualTracking|startTrackAsPersistent|setTrackingMode|FutureTrack|PermissionWizard" .`.

2. Verify the latest plugin version before editing dependencies:
   - Published package: query pub.dev, e.g. `curl -s https://pub.dev/api/packages/telematics_sdk` and read `latest.version`, or use `flutter pub add telematics_sdk` to resolve the latest compatible version.
   - Git dependency: query tags with `git ls-remote --tags --refs https://github.com/Mobile-Telematics/telematicsSDK-demoapp-flutter-.git` and use the latest semantic version tag exactly when the user needs the repository version.
   - Do not hardcode the verified reference version from this skill unless it is still the latest compatible version.

3. Choose dependency style based on the app:
   - Published package: add `telematics_sdk` to `dependencies` when available in the app's package source.
   - Git dependency: use the public plugin repository when the user needs the repository version instead of a published package.
   - Do not edit native plugin internals in the app unless the task is to patch the plugin itself.

4. Load platform references only as needed:
   - `references/flutter/plugin-api.md` for Dart `TrackingApi`, streams, flow sequencing, and facade recommendations.
   - `references/android/host-setup.md` for Android manifest, Gradle, permissions, proguard, `Application`, and advanced tracking settings.
   - `references/ios/host-setup.md` for iOS `Info.plist`, `AppDelegate`, Flutter implicit engine registration, SceneDelegate, and lifecycle forwarding.

5. Before implementing a new reusable Flutter service/facade, ask which primary tracking flow should be placed first:
   - automatic tracking
   - standard manual tracking without tags
   - standard manual tracking with future tags
   - app-controlled persistent manual tracking without tags
   - app-controlled persistent manual tracking with future tags
   - one-time persistent manual tracking without tags
   - one-time persistent manual tracking with future tags
   If the user already stated the primary flow or product code makes it clear, use it without asking.

6. Prefer an app-owned Dart facade around `TrackingApi` instead of calling the plugin directly from many widgets. The facade should own device ID assignment, SDK enablement, permission checks, manual tracking state, persistent mode policy, future-tag sequencing, and stream subscription lifecycle.

7. Configure host platforms completely. Flutter Dart code alone is insufficient:
   - Android must have required permissions, Gradle/repository settings, release shrink settings, proguard rules, and `tools:replace` where manifest merge requires it.
   - iOS must have required permission keys, background modes, BG task identifiers, and `RPEntry.initializeSDK()` before plugin registration.

8. Implement flow methods in Dart with the selected primary flow first and mark it:
   - `// Primary Flow: <name requested by user>`
   - Use separate sections for other supported flows.

9. Validate with the narrowest available checks:
   - `flutter pub get`
   - `dart analyze` or `flutter analyze`
   - relevant tests with `flutter test`
   - Android build or at least `./gradlew` task when Android files changed
   - iOS `pod install` / `flutter build ios --no-codesign` when feasible after iOS setup changes

## Coding Rules

- Create one `TrackingApi` ownership point per feature/service. Avoid creating new `TrackingApi()` instances in many widgets unless the app architecture already does that deliberately.
- Subscribe to plugin streams in a lifecycle-owned object and cancel subscriptions in `dispose`/service shutdown.
- Set the device ID before enabling SDK or starting manual tracking.
- Check `isAllRequiredPermissionsAndSensorsGranted()` before enabling SDK or starting flows; use `showPermissionWizard(...)` when the product wants plugin-managed permission onboarding.
- For automatic tracking, call `setDeviceID(...)` then `setEnableSdk(enable: true)`.
- For standard manual tracking, ensure device ID, permissions, SDK enabled, `setTrackingMode(TrackingMode.standard)`, then `startManualTracking()`.
- For app-controlled persistent manual tracking, set `TrackingMode.persistent`, start manual tracking, and restore `TrackingMode.standard` when business logic ends persistent mode.
- For one-time persistent manual tracking, use `startTrackAsPersistent()` and do not add a redundant manual `setTrackingMode(TrackingMode.standard)` reset unless the installed native API proves it does not restore automatically on that platform.
- For manual-only product flows, stop manual tracking and then disable the SDK with `setEnableSdk(enable: false)`. Keep the SDK enabled after manual stop only when the product intentionally combines manual trips with automatic tracking.
- If one service method stops all manual flows, track whether the active session was app-controlled persistent before deciding to restore `TrackingMode.standard`.
- Add future tags before starting a manually tagged trip; wait for the native callback or document the race if the product accepts it.
- Remove future tags before disabling the SDK when cleanup depends on SDK/API availability.
- Treat tagged manual flows as future-tag flows for upcoming trips. Do not imply processed-trip tag editing unless the latest installed plugin exposes that API.
- Generated facades should expose all seven app-level flows explicitly: automatic, standard manual with and without future tags, app-controlled persistent manual with and without future tags, and one-time persistent manual with and without future tags.
- On iOS, call iOS-only methods only behind `Platform.isIOS` or through facade methods that no-op/throw intentionally.
- On Android, call Android-only methods only behind `Platform.isAndroid` or through facade methods that no-op/throw intentionally.
- Do not treat `logout()` as a temporary disable switch. It clears the device ID; use `setEnableSdk(enable: false)` when logout semantics are not intended.
- Surface plugin errors to the caller/UI; do not swallow `PlatformException`, `UnsupportedError`, or permission failures.
- Do not enable Android code shrinking/resource shrinking for release unless the app has verified keep rules for Telematics SDK.
- Keep host app platform changes minimal and explicit; avoid rewriting generated Flutter files unless the integration requires it.

## Official Docs

Use these docs for baseline behavior, but check current Flutter plugin and native source before copying code:

- https://docs.damoov.com/docs/-download-the-sdk-and-install-it-in-your-environment
- https://docs.damoov.com/docs/tracking
- https://docs.damoov.com/docs/ios-sdk-incoming-tags
- https://docs.damoov.com/docs/trip-tag-for-ios-app
- https://docs.flutter.dev/release/breaking-changes/uiscenedelegate
