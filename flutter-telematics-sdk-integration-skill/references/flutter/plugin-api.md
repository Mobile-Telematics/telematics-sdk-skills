# Flutter Plugin API Reference

This reference summarizes the Dart API shape verified from `telematics_sdk` plugin version `1.1.1`. Inspect the latest package and installed package before editing an app because method names and platform support can change.

## Dependency

Before editing `pubspec.yaml`, verify the latest package version:

```bash
curl -s https://pub.dev/api/packages/telematics_sdk
```

Read `latest.version`, or let Flutter resolve the latest compatible package:

```bash
flutter pub add telematics_sdk
```

If editing `pubspec.yaml` manually, use the latest compatible version rather than the example version from this reference:

```yaml
dependencies:
  telematics_sdk: ^<latest-pub-version>
```

When the user asks for the repository version:

```bash
git ls-remote --tags --refs https://github.com/Mobile-Telematics/telematicsSDK-demoapp-flutter-.git
```

Use the latest semantic version tag exactly:

```yaml
dependencies:
  telematics_sdk:
    git:
      url: https://github.com/Mobile-Telematics/telematicsSDK-demoapp-flutter-.git
      ref: <latest-semver-tag>
```

Run `flutter pub get` after editing `pubspec.yaml`.

## Entry Point

Import and create the API:

```dart
import 'package:telematics_sdk/telematics_sdk.dart';

final trackingApi = TrackingApi();
```

No app-side credentials are passed to `TrackingApi()`. The SDK setup described by this skill does not require an API key or project credential in Dart code.

Prefer wrapping this in an app service:

```dart
class TelematicsService {
  TelematicsService({TrackingApi? trackingApi})
      : _trackingApi = trackingApi ?? TrackingApi();

  final TrackingApi _trackingApi;
}
```

Do not scatter `TrackingApi()` calls across many widgets. Centralizing the API keeps permission checks, device ID setup, and tracking flow state consistent.

## Common Methods

- `isInitialized() -> Future<bool?>`
- `setDeviceID({required String deviceId})`
- `getDeviceId() -> Future<String?>`
- `getDeviceIdRegistrationState() -> Future<DeviceIdRegistrationState>`
- `logout()`
- `isAllRequiredPermissionsAndSensorsGranted() -> Future<bool?>`
- `isSdkEnabled() -> Future<bool?>`
- `isTracking() -> Future<bool?>`
- `setEnableSdk({required bool enable})`
- `getTrackingState() -> Future<TrackingState>`
- `startManualTracking() -> Future<bool?>`
- `startTrackAsPersistent() -> Future<bool?>`
- `stopManualTracking() -> Future<bool?>`
- `setMaxPersistentTrackingInterval({required int minutes})`
- `getMaxPersistentTrackingInterval() -> Future<int?>`
- `setTrackingMode({required TrackingMode trackingMode})`
- `getTrackingMode() -> Future<TrackingMode?>`
- `uploadUnsentTrips()`
- `getUnsentTripCount() -> Future<int?>`
- `sendCustomHeartbeats({required String reason})`
- `showPermissionWizard({required bool enableAggressivePermissionsWizard, required bool enableAggressivePermissionsWizardPage})`
- `registerSpeedViolations({required double speedLimitKmH, required int speedLimitTimeout})`
- `setAccidentDetectionSensitivity({required AccidentDetectionSensitivity sensitivity})`
- `setAccidentDetectionEnabled({required bool value})`
- `isAccidentDetectionEnabled() -> Future<bool?>`
- `isRTLDEnabled() -> Future<bool?>`

## Platform-Specific Methods

iOS-only:

- `getApiLanguage()`
- `setApiLanguage({required ApiLanguage language})`
- `isAggressiveHeartbeats()`
- `setAggressiveHeartbeats({required bool value})`
- `setDisableTracking({required bool value})`
- `isDisableTracking()`
- `isWrongAccuracyState()`
- `requestIOSLocationAlwaysPermission()`
- `requestIOSMotionPermission()`
- `iOSWrongAccuracyAuthorization`
- `iOSRTLDDataCollected`

Android-only:

- `setAndroidAutoStartEnabled({required bool enable, required bool permanent})`
- `isAndroidAutoStartEnabled()`

The plugin throws `UnsupportedError` for wrong-platform calls. Guard platform-specific calls with `Platform.isIOS` or `Platform.isAndroid`.

## Streams

The plugin exposes native callbacks as streams:

- `onPermissionWizardClose -> Stream<PermissionWizardResult>`
- `lowPowerMode -> Stream<bool>`
- `locationChanged -> Stream<TrackLocation>`
- `trackingStateChanged -> Stream<bool>`
- `speedViolation -> Stream<SpeedViolation>`
- `iOSWrongAccuracyAuthorization -> Stream<void>`
- `iOSRTLDDataCollected -> Stream<void>`

Cancel subscriptions in `dispose` or service shutdown:

```dart
late final StreamSubscription<bool> _trackingSub;

void start() {
  _trackingSub = _trackingApi.trackingStateChanged.listen((isTracking) {
    // Update app state.
  });
}

Future<void> dispose() => _trackingSub.cancel();
```

## Flow Sequences

Supported app-level flows:

- automatic tracking
- standard manual tracking without future tags
- standard manual tracking with future tags
- app-controlled persistent manual tracking without future tags
- app-controlled persistent manual tracking with future tags
- one-time persistent manual tracking without future tags
- one-time persistent manual tracking with future tags

For Flutter tagged-flow snippets, `addFutureTrackTag(...)` means an app facade helper that calls the raw plugin `TrackingApi.addFutureTrackTag(...)` and completes only after the `onTagAdd` callback reports the native result. If the app uses the raw plugin method directly, wait for `onTagAdd` before starting tracking when tag attachment is product-critical.

Device identity setup:

```dart
await trackingApi.setDeviceID(deviceId: deviceId);
```

Set the device ID from the app's login/session binding flow before enabling automatic SDK collection or starting manual tracking. The value is a Damoov platform user identifier in GUID format. Do not generate it locally unless the product backend explicitly proxies the Damoov value, and do not repeat this call inside every tracking start method.

Automatic tracking:

```dart
await trackingApi.setEnableSdk(enable: true);
```

Disable SDK collection while preserving device ID:

```dart
await trackingApi.setEnableSdk(enable: false);
```

Logout when clearing identity is intended:

```dart
await trackingApi.logout();
```

`logout()` clears the device ID. Set the device ID again before enabling the SDK or starting tracking later.

Standard manual tracking:

```dart
await trackingApi.setEnableSdk(enable: true);
await trackingApi.setTrackingMode(trackingMode: TrackingMode.standard);
await trackingApi.startManualTracking();
```

Calling `startManualTracking()` or `startTrackAsPersistent()` while tracking is already active is idempotent: the SDK continues the existing track and does not start a new one. A facade may still check `isTracking()` to keep UI state clear.

Standard manual tracking with future tags:

```dart
await trackingApi.setEnableSdk(enable: true);
await trackingApi.setTrackingMode(trackingMode: TrackingMode.standard);
await addFutureTrackTag(tag: tag, source: source);
await trackingApi.startManualTracking();
```

App-controlled persistent manual tracking:

```dart
await trackingApi.setEnableSdk(enable: true);
await trackingApi.setMaxPersistentTrackingInterval(minutes: minutes);
await trackingApi.setTrackingMode(trackingMode: TrackingMode.persistent);
await trackingApi.startManualTracking();
```

App-controlled persistent manual tracking with future tags:

```dart
await trackingApi.setEnableSdk(enable: true);
await trackingApi.setMaxPersistentTrackingInterval(minutes: minutes);
await trackingApi.setTrackingMode(trackingMode: TrackingMode.persistent);
await addFutureTrackTag(tag: tag, source: source);
await trackingApi.startManualTracking();
```

When the app ends tagged app-controlled persistent mode in a manual-only product flow:

```dart
await trackingApi.removeAllFutureTrackTags();
await trackingApi.stopManualTracking();
await trackingApi.setTrackingMode(trackingMode: TrackingMode.standard);
await trackingApi.setEnableSdk(enable: false);
```

When the app ends app-controlled persistent mode in a manual-only product flow:

```dart
await trackingApi.stopManualTracking();
await trackingApi.setTrackingMode(trackingMode: TrackingMode.standard);
await trackingApi.setEnableSdk(enable: false);
```

One-time persistent manual tracking:

```dart
await trackingApi.setEnableSdk(enable: true);
await trackingApi.setMaxPersistentTrackingInterval(minutes: minutes);
await trackingApi.startTrackAsPersistent();
```

One-time persistent manual tracking with future tags:

```dart
await trackingApi.setEnableSdk(enable: true);
await trackingApi.setMaxPersistentTrackingInterval(minutes: minutes);
await addFutureTrackTag(tag: tag, source: source);
await trackingApi.startTrackAsPersistent();
```

Stop a manual-only tracking flow:

```dart
await trackingApi.stopManualTracking();
await trackingApi.setEnableSdk(enable: false);
```

Stop a tagged manual-only tracking flow when future-tag cleanup is required:

```dart
await trackingApi.removeAllFutureTrackTags();
await trackingApi.stopManualTracking();
await trackingApi.setEnableSdk(enable: false);
```

If the app intentionally combines manual trips with automatic tracking, keep the SDK enabled after `stopManualTracking()` and document that product behavior in the facade.

## Future Tags

Future tag operations return immediately and deliver results through callbacks assigned on `TrackingApi`:

```dart
final trackingApi = TrackingApi()
  ..onTagAdd = (status, tag, activationTime) {
    // Status and tag from native SDK.
  };

await trackingApi.addFutureTrackTag(tag: 'business', source: 'trip_form');
```

`tag` and `source` are product-defined strings. The SDK does not define an enum or SDK-side value restrictions. Use `tag` for the business label and `source` for the app module or user action that created it.

When the facade exposes `addFutureTrackTag(...)`, implement it as an async wrapper around the raw plugin method and `onTagAdd`. Preserve the app's previous `onTagAdd` handler if one is already installed:

```dart
import 'dart:async';

Future<FutureTagAddResult> addFutureTrackTag({
  required String tag,
  required String source,
}) async {
  final completer = Completer<FutureTagAddResult>();
  final previousOnTagAdd = _trackingApi.onTagAdd;

  _trackingApi.onTagAdd = (status, addedTag, activationTime) {
    previousOnTagAdd?.call(status, addedTag, activationTime);
    if (!completer.isCompleted && addedTag == tag) {
      completer.complete(
        FutureTagAddResult(
          status: status,
          tag: addedTag,
          activationTime: activationTime,
        ),
      );
    }
  };

  await _trackingApi.addFutureTrackTag(tag: tag, source: source);
  return completer.future;
}

class FutureTagAddResult {
  const FutureTagAddResult({
    required this.status,
    required this.tag,
    required this.activationTime,
  });

  final Object status;
  final String tag;
  final Object? activationTime;
}
```

Adjust the exact callback parameter types to the installed plugin source. The important contract is that the facade method completes only after native `onTagAdd` reports the result.

Available methods:

- `getFutureTrackTags()`
- `addFutureTrackTag({required String tag, required String source})`
- `removeFutureTrackTag({required String tag})`
- `removeAllFutureTrackTags()`

For a manually tagged trip, add the future tag and wait for the `onTagAdd` result before starting tracking where product correctness depends on tags being attached to the upcoming trip.

The verified Flutter wrapper exposes future-tag operations for upcoming trips. It does not expose a processed-trip tag editing API in the checked plugin surface. If a product needs post-trip tag editing, inspect the latest installed plugin first and add/verify a native bridge before claiming support.

## Recommended Service Shape

Expose app-level flows rather than raw plugin calls from widgets:

```dart
enum TelematicsFlow {
  automatic,
  standardManual,
  standardManualWithFutureTag,
  appControlledPersistentManual,
  appControlledPersistentManualWithFutureTag,
  oneTimePersistentManual,
  oneTimePersistentManualWithFutureTag,
}
```

The service should:

- Expose a separate identity method that validates and sets a non-empty device ID.
- Expose `logout()` separately for user logout/account-removal semantics.
- Check permissions before enable/start flows.
- Own whether the current session is app-controlled persistent.
- Sequence future tag calls before manual starts.
- Convert `PlatformException` and `UnsupportedError` into app-facing errors.
- Keep platform-specific controls behind platform checks.

## Testing Notes

iOS Simulator and Android Emulator can be used to exercise integration flow, permissions, simulated location, and trip recording. HF Data cannot be fully tested on emulators because accelerometer and gyroscope sensor data are not available like on a physical device. Use emulator/simulator location routes for flow checks, and run final background and sensor-heavy validation on real devices.
