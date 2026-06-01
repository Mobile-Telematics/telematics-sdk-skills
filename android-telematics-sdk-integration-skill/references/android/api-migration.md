# Android API Migration

Use this reference when upgrading an existing Android integration from SDK 3.x or stale documentation to SDK 4.x.

## Migration Matrix

| Area | SDK 3.x / stale docs | SDK 4.x | Migration |
|---|---|---|---|
| SDK dependency | `com.telematicssdk:tracking:3.x.x` | User-requested or Maven-verified SDK 4.x version | Update Gradle dependency and repository. Do not pick the version from changelog order alone. |
| Minimum Android SDK | `21` in older docs | `23` | Update `minSdk` if the host app is lower. |
| Enable SDK | `setEnableSdk(enable, withCheckingPermissions)` | `setEnableSdk(enable)` | Remove the second parameter and check permissions before enabling. |
| Disable after upload | `setDisableWithUpload()` | `setEnableSdk(false)` | Use `setEnableSdk(false)`. |
| Settings creation | Legacy `Settings(...)` constructors with HF/ELM params | `Settings()` builder functions | Replace constructors with builder calls. |
| Device token naming | `setDeviceToken(...)`, `virtualDeviceToken` in older docs | `setDeviceID(...)`, `getDeviceId()` | Use current `TrackingApi` methods. |
| ELM327 | `setElmDevicesEnabled()`, `isElmDevicesEnabled()`, `getElmManager()` | Removed | Delete ELM integration code and Bluetooth OBD flows. |
| Advertising ID | `setAdvertisingId()`, `clearAdvertisingId()` | Removed | Delete calls. No replacement is required in SDK 4.x. |
| HF recording toggle | `setHfRecordingEnabled()`, `isHfRecordingOn()` | Removed | Delete calls. |
| Persistent tracking | `startPersistentTracking()` | `startTrackAsPersistent()` | Replace with `startTrackAsPersistent()`. |
| Tracking mode | Not available | `setTrackingMode()`, `getTrackingMode()` | Use `TrackingMode.Standard` or `TrackingMode.Persistent`. |
| Persistent interval | Not available | `setMaxPersistentTrackingInterval()`, `getMaxPersistentTrackingInterval()` | Configure valid `5..600` minute intervals; default is `240`. |
| Tracking state | Custom app-side checks | `getTrackingState()` | Use SDK state API. |
| Device ID registration state | Custom app-side checks | `getDeviceIdRegistrationState()` | Use SDK state API. |
| SDK version | Custom build config or dependency version | `getSdkVersion()` | Use SDK version getter when runtime version checks are needed. |
| Passive detection | Not part of old settings builder | `passiveDetectionOn()` and `setPassiveDetectionEnabled()` | Configure via `Settings` or `TrackingApi`. |
| Track point model | `TrackPoint.vehicleIndicators` existed | removed | Remove usage or serialization dependency on this field. |
| Server DTO names | Old `*ServerModel` / `*Dto` names | Current response/request model names | Update imports if the app directly references SDK DTO classes. |

## Settings Migration

Old:

```kotlin
val settings = Settings(
    Settings.stopTrackingTimeHigh,
    Settings.accuracyHigh,
    true,
    true,
    false,
    false
)
```

New:

```kotlin
val settings = Settings()
    .accuracy(Settings.accuracyHigh)
    .stopTrackingTimeout(Settings.stopTrackingTimeHigh)
    .autoStartOn(true)
    .adOn(false)
    .passiveDetectionOn(false)
```

## Enable And Disable SDK

Old:

```kotlin
trackingApi.setEnableSdk(true, true)
```

New:

```kotlin
if (trackingApi.isAllRequiredPermissionsAndSensorsGranted()) {
    trackingApi.setEnableSdk(true)
}
```

Disable:

```kotlin
trackingApi.setEnableSdk(false)
```

Use `logout()` only when the app intends to clear device identity. Do not hide `logout()` inside a generic disable method; expose it as a separate repository/API action.

## Persistent Tracking Migration

Replace:

```kotlin
trackingApi.startPersistentTracking()
```

With:

```kotlin
trackingApi.startTrackAsPersistent()
```

For app-controlled persistent mode:

```kotlin
trackingApi.setTrackingMode(TrackingMode.Persistent)
trackingApi.startTracking()
// later
trackingApi.stopTracking()
trackingApi.setTrackingMode(TrackingMode.Standard)
```

For one-time persistent mode:

```kotlin
trackingApi.startTrackAsPersistent()
// later
trackingApi.stopTracking()
```

Do not add manual mode reset to one-time persistent flow unless the resolved SDK version requires it.

## Removed APIs To Search For

Run focused searches during migration:

```powershell
rg -n "setEnableSdk\\([^\\n,]+,[^\\n]+\\)|setDisableWithUpload|startPersistentTracking|setDeviceToken|virtualDeviceToken|setAdvertisingId|clearAdvertisingId|setElmDevicesEnabled|isElmDevicesEnabled|getElmManager|setHfRecordingEnabled|isHfRecordingOn|vehicleIndicators" .
```

Replace or remove every match unless the resolved SDK dependency proves the API is still supported and not deprecated.

## Migration Checklist

- Update dependency to the user-requested or Maven-verified SDK 4.x version.
- Ensure `minSdk >= 23`.
- Replace old `Settings(...)` constructors with builder functions.
- Remove HF and ELM settings arguments.
- Replace `setEnableSdk(enable, withCheckingPermissions)` with `setEnableSdk(enable)`.
- Replace `setDisableWithUpload()` with `setEnableSdk(false)`.
- Replace stale `setDeviceToken` docs/code with `setDeviceID`.
- Remove Advertising ID API calls.
- Remove ELM327 / Bluetooth OBD API calls, UI, imports, and Bluetooth permissions that were only needed for ELM.
- Replace `startPersistentTracking()` with `startTrackAsPersistent()`.
- Add `setTrackingMode(...)` when the app needs explicit standard or persistent mode behavior.
- Add persistent interval validation if the default `240` minutes is not suitable.
- Use `getTrackingState()` for tracking status UI or diagnostics.
- Use `getDeviceIdRegistrationState()` for device ID registration UI or diagnostics.
- Use `getSdkVersion()` for runtime SDK version checks.
- Remove usage of `TrackPoint.vehicleIndicators`.
- Update direct DTO imports if the app references old SDK DTO names.
- Move blocking trip/statistics calls off the main thread.
- Move user-visible text literals introduced during migration into the host app's `Settings`/localization layer.
- Run a clean Gradle compile and re-test login, permissions, SDK enable/disable, automatic tracking, manual tracking, persistent tracking, logout, and trip upload flows.
