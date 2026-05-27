# Current RPEntry API Migration Notes

The published docs may still show deprecated members until the next SDK release documentation is updated.

## Verified Replacements

| Deprecated API | Current API |
| --- | --- |
| `RPEntry.instance.virtualDeviceToken = value` | `try RPEntry.instance.setDeviceID(deviceId: value)` |
| `RPEntry.instance.virtualDeviceToken` | `RPEntry.instance.getDeviceId()` |
| `RPEntry.instance.apiLanguage = value` | `RPEntry.instance.setApiLanguage(apiLanguage: value)` |
| `RPEntry.instance.apiLanguage` | `RPEntry.instance.getApiLanguage()` |
| `RPEntry.instance.accidentDetectionSensitivity = value` | `RPEntry.instance.setAccidentDetectionSensitivity(sensitivity: value)` |
| `RPEntry.instance.accidentDetectionSensitivity` | `RPEntry.instance.getAccidentDetectionSensitivity()` |
| `RPEntry.instance.wrongAccuracyState` | `RPEntry.instance.isWrongAccuracyState()` |
| `RPEntry.instance.disableTracking = value` | `RPEntry.instance.setDisableTracking(disableTracking: value)` |
| `RPEntry.instance.disableTracking` | `RPEntry.instance.isDisableTracking()` |
| `RPEntry.instance.removeVirtualDeviceToken()` | `RPEntry.instance.clearDeviceID()` |
| `RPEntry.instance.startPersistentTracking()` | Use `RPEntry.instance.startTrackAsPersistent()` for one-time persistent manual tracking with automatic `.standard` reset, or `setTrackingMode(.persistent)` then `startTracking()` for app-controlled persistent mode. |
| `RPEntry.instance.isAllRequiredPermissionsGranted()` | `RPEntry.instance.isAllRequiredPermissionsAndSensorsGranted()` |
| `RPEntry.instance.isRTDEnabled()` | `RPEntry.instance.isRTLDEnabled()` |
| `RPEntry.instance.isTrackingActive()` | `RPEntry.instance.isTracking()` |
| `RPEntry.instance.aggressiveHeartbeat()` | `RPEntry.instance.isAggressiveHeartbeats()` |
| `RPEntry.instance.enableAccidents(value)` | `RPEntry.instance.setAccidentDetectionEnabled(value)` |
| `RPEntry.instance.isEnabledAccidents()` | `RPEntry.instance.isAccidentDetectionEnabled()` |

## Notes For Migration

- `logout()` disables the SDK and clears the device ID. After logout, re-enable with `setEnableSdk(true)` and set a new device ID with `try setDeviceID(deviceId:)`.
- Do not add a migration item from a non-throwing `setDeviceID(deviceId:)` signature; that signature was not published.
- `setDisableTracking(disableTracking:)` prevents new user-initiated tracking sessions but does not disable the SDK or heartbeats.
- `setEnableSdk(false)` is the stronger switch when no SDK data should be collected.
- `startTrackAsPersistent()` starts a one-time persistent session. It automatically switches `trackingMode` to `.persistent`, then restores `.standard` after `stopTracking()` or when `setMaxPersistentTrackingInterval` is reached.
- For app-controlled persistent mode, `setTrackingMode(.persistent)` plus `startTracking()` still requires the app to restore `.standard` when required.
- The deprecation message for `wrongAccuracyState` mentions `setDisableTracking(_:)`; verify intent before treating wrong accuracy state as app-controlled. The available direct replacement for reading state is `isWrongAccuracyState()`.

## Review Checklist

Run this in the target app after migration:

```bash
rg -n "virtualDeviceToken|apiLanguage|accidentDetectionSensitivity|wrongAccuracyState|disableTracking|removeVirtualDeviceToken|startPersistentTracking|isAllRequiredPermissionsGranted|isRTDEnabled|isTrackingActive|aggressiveHeartbeat|enableAccidents|isEnabledAccidents" .
```

Any remaining match should be either a deliberate compatibility shim or a documented exception.
