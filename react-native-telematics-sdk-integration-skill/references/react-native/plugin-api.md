# React Native Plugin API Reference

This reference summarizes the TypeScript API shape verified from the public `react-native-telematics` source at version `3.0.0`. Inspect the latest package and installed package before editing an app because method names and platform support can change.

## Dependency

Before editing `package.json`, verify the latest published package:

```bash
npm view react-native-telematics version
```

Use the app's package manager to install the latest compatible version:

```bash
yarn add react-native-telematics@latest
```

or:

```bash
npm install react-native-telematics@latest
```

When the user asks for the repository version:

```bash
git ls-remote --tags --refs https://github.com/Mobile-Telematics/telematicsSDK-demoapp-react.git
```

Use the latest semantic version tag exactly:

```json
{
  "dependencies": {
    "react-native-telematics": "github:Mobile-Telematics/telematicsSDK-demoapp-react#<latest-semver-tag>"
  }
}
```

Run the app-standard install command and rebuild native apps after adding the package.

## Entry Point

Import the default API and needed enums/listeners:

```ts
import { Platform } from 'react-native';
import TelematicsSdk, {
  AccidentDetectionSensitivity,
  ApiLanguage,
  TrackingMode,
  addOnLocationChangedListener,
  addOnTrackingStateChangedListener,
  addOnLowPowerModeListener,
} from 'react-native-telematics';
```

The package supports React Native New Architecture through a TurboModule and falls back to the legacy native module. If the module is missing, the JS wrapper throws an error that usually means pods/Gradle sync and a native rebuild are required.

## Common Methods

- `initializeSdk(): Promise<void>`
- `isInitializedSdk(): Promise<boolean>`
- `getDeviceId(): Promise<string>`
- `setDeviceId(deviceId: string): Promise<void>`
- `getDeviceIdRegistrationState(): Promise<DeviceIdRegistrationState>`
- `logout(): Promise<void>`
- `isAllRequiredPermissionsAndSensorsGranted(): Promise<boolean>`
- `isSdkEnabled(): Promise<boolean>`
- `isTracking(): Promise<boolean>`
- `setEnableSdk(enable: boolean): Promise<void>`
- `startManualTracking(): Promise<void>`
- `startTrackAsPersistent(): Promise<void>`
- `stopManualTracking(): Promise<void>`
- `setMaxPersistentTrackingInterval(minutes: number): Promise<void>`
- `getMaxPersistentTrackingInterval(): Promise<number>`
- `setTrackingMode(trackingMode: TrackingMode): Promise<void>`
- `getTrackingMode(): Promise<TrackingMode>`
- `getTrackingState(): Promise<TrackingState>`
- `uploadUnsentTrips(): Promise<void>`
- `getUnsentTripCount(): Promise<number>`
- `sendCustomHeartbeats(reason: string): Promise<void>`
- `showPermissionWizard(enableAggressivePermissionsWizard: boolean, enableAggressivePermissionsWizardPage: boolean): Promise<boolean>`
- `registerSpeedViolations({ speedLimitKmH, speedLimitTimeout }): Promise<void>`
- `setAccidentDetectionSensitivity(accidentDetectionSensitivity): Promise<void>`
- `enableAccidents(enable: boolean): Promise<void>`
- `isEnabledAccidents(): Promise<boolean>`
- `isRTLDEnabled(): Promise<boolean>`

Enums:

- `TrackingMode.Standard = 0`
- `TrackingMode.Persistent = 1`
- `AccidentDetectionSensitivity.Normal = 0`
- `AccidentDetectionSensitivity.Sensitive = 1`
- `AccidentDetectionSensitivity.Tough = 2`
- `ApiLanguage.none`, `english`, `russian`, `portuguese`, `spanish`

## Platform-Specific Methods

iOS-only:

- `getApiLanguage()`
- `setApiLanguage(language: ApiLanguage)`
- `isAggressiveHeartbeats()`
- `setAggressiveHeartbeats(enable: boolean)`
- `setDisableTracking(value: boolean)`
- `isDisableTracking()`
- `isWrongAccuracyState()`
- `requestIOSLocationAlwaysPermission()`
- `requestIOSMotionPermission()`
- `addOnLowPowerModeListener(...)`
- `addOnWrongAccuracyAuthorizationListener(...)`
- `addOnRtldColectedData(...)`

Android-only:

- `setAndroidAutoStartEnabled({ enable: boolean, permanent: boolean })`
- `isAndroidAutoStartEnabled()`

The native side rejects wrong-platform calls. Guard platform-specific calls with `Platform.OS`.

## Listeners

All listeners return subscriptions with `.remove()`:

- `addOnLowPowerModeListener(({ enabled }) => void)` iOS only
- `addOnLocationChangedListener(({ latitude, longitude }) => void)`
- `addOnTrackingStateChangedListener((state: boolean) => void)`
- `addOnWrongAccuracyAuthorizationListener(() => void)` iOS only
- `addOnRtldColectedData(() => void)` iOS only
- `addOnSpeedViolationListener((event) => void)`

Example:

```ts
useEffect(() => {
  const subs = [
    addOnLocationChangedListener(({ latitude, longitude }) => {
      // Update app state.
    }),
    addOnTrackingStateChangedListener((isTracking) => {
      // Update app state.
    }),
  ];

  if (Platform.OS === 'ios') {
    subs.push(addOnLowPowerModeListener(({ enabled }) => {}));
  }

  return () => subs.forEach((subscription) => subscription.remove());
}, []);
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

Initialize once before JS-side API usage:

```ts
await TelematicsSdk.initializeSdk();
```

On iOS, this JS call does not replace native launch initialization. `AppDelegate` still must call `RPEntry.initializeSDK()`.

Automatic tracking:

```ts
await TelematicsSdk.initializeSdk();
await TelematicsSdk.setDeviceId(deviceId);
await TelematicsSdk.setEnableSdk(true);
```

Disable SDK collection while preserving device ID:

```ts
await TelematicsSdk.setEnableSdk(false);
```

Logout when clearing identity is intended:

```ts
await TelematicsSdk.logout();
```

`logout()` clears the device ID. Set the device ID again before enabling the SDK or starting tracking later.

Standard manual tracking:

```ts
await TelematicsSdk.initializeSdk();
await TelematicsSdk.setDeviceId(deviceId);
await TelematicsSdk.setEnableSdk(true);
await TelematicsSdk.setTrackingMode(TrackingMode.Standard);
await TelematicsSdk.startManualTracking();
```

Standard manual tracking with future tags:

```ts
await TelematicsSdk.initializeSdk();
await TelematicsSdk.setDeviceId(deviceId);
await TelematicsSdk.setEnableSdk(true);
await TelematicsSdk.setTrackingMode(TrackingMode.Standard);
await TelematicsSdk.addFutureTrackTag(tag, source);
await TelematicsSdk.startManualTracking();
```

App-controlled persistent manual tracking:

```ts
await TelematicsSdk.initializeSdk();
await TelematicsSdk.setDeviceId(deviceId);
await TelematicsSdk.setEnableSdk(true);
await TelematicsSdk.setMaxPersistentTrackingInterval(minutes);
await TelematicsSdk.setTrackingMode(TrackingMode.Persistent);
await TelematicsSdk.startManualTracking();
```

App-controlled persistent manual tracking with future tags:

```ts
await TelematicsSdk.initializeSdk();
await TelematicsSdk.setDeviceId(deviceId);
await TelematicsSdk.setEnableSdk(true);
await TelematicsSdk.setMaxPersistentTrackingInterval(minutes);
await TelematicsSdk.setTrackingMode(TrackingMode.Persistent);
await TelematicsSdk.addFutureTrackTag(tag, source);
await TelematicsSdk.startManualTracking();
```

When the app ends tagged app-controlled persistent mode in a manual-only product flow:

```ts
await TelematicsSdk.removeAllFutureTrackTags();
await TelematicsSdk.stopManualTracking();
await TelematicsSdk.setTrackingMode(TrackingMode.Standard);
await TelematicsSdk.setEnableSdk(false);
```

When the app ends app-controlled persistent mode in a manual-only product flow:

```ts
await TelematicsSdk.stopManualTracking();
await TelematicsSdk.setTrackingMode(TrackingMode.Standard);
await TelematicsSdk.setEnableSdk(false);
```

One-time persistent manual tracking:

```ts
await TelematicsSdk.initializeSdk();
await TelematicsSdk.setDeviceId(deviceId);
await TelematicsSdk.setEnableSdk(true);
await TelematicsSdk.setMaxPersistentTrackingInterval(minutes);
await TelematicsSdk.startTrackAsPersistent();
```

One-time persistent manual tracking with future tags:

```ts
await TelematicsSdk.initializeSdk();
await TelematicsSdk.setDeviceId(deviceId);
await TelematicsSdk.setEnableSdk(true);
await TelematicsSdk.setMaxPersistentTrackingInterval(minutes);
await TelematicsSdk.addFutureTrackTag(tag, source);
await TelematicsSdk.startTrackAsPersistent();
```

Stop a manual-only tracking flow:

```ts
await TelematicsSdk.stopManualTracking();
await TelematicsSdk.setEnableSdk(false);
```

Stop a tagged manual-only tracking flow when future-tag cleanup is required:

```ts
await TelematicsSdk.removeAllFutureTrackTags();
await TelematicsSdk.stopManualTracking();
await TelematicsSdk.setEnableSdk(false);
```

If the app intentionally combines manual trips with automatic tracking, keep the SDK enabled after `stopManualTracking()` and document that product behavior in the facade.

## Future Tags

Future tag operations are promise-based in the React Native wrapper:

```ts
const result = await TelematicsSdk.addFutureTrackTag('business', 'trip_form');
```

Available methods:

- `getFutureTrackTags() -> Promise<{ status: string; tags: Tag[] }>`
- `addFutureTrackTag(tag: string, source?: string) -> Promise<{ status: string; tag: Tag }>`
- `removeFutureTrackTag(tag: string) -> Promise<{ status: string; tag: Tag }>`
- `removeAllFutureTrackTags() -> Promise<string>`

For a manually tagged trip, await the tag promise before starting tracking where product correctness depends on tags being attached to the upcoming trip.

The verified React Native wrapper exposes future-tag operations for upcoming trips. It does not expose a processed-trip tag editing API in the checked plugin surface. If a product needs post-trip tag editing, inspect the latest installed plugin first and add/verify a native bridge before claiming support.

## Recommended Service Shape

Expose app-level flows rather than raw plugin calls from components:

```ts
export type TelematicsFlow =
  | 'automatic'
  | 'standardManual'
  | 'standardManualWithFutureTag'
  | 'appControlledPersistentManual'
  | 'appControlledPersistentManualWithFutureTag'
  | 'oneTimePersistentManual'
  | 'oneTimePersistentManualWithFutureTag';
```

The service should:

- Ensure `initializeSdk()` has run.
- Validate non-empty device ID.
- Check permissions before enable/start flows.
- Own whether the current session is app-controlled persistent.
- Sequence future tag calls before manual starts.
- Convert promise rejections into app-facing errors.
- Keep platform-specific controls behind `Platform.OS` checks.
