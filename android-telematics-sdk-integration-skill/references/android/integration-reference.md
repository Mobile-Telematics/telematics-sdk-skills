# Damoov TelematicsSDK Android Integration Reference

Use this as a compact reference for Android app integration. Verify the resolved SDK dependency before copying method names.

## Dependency

Use Maven/Gradle dependency integration for host apps:

```kotlin
repositories {
    maven {
        url = uri("https://s3.us-east-2.amazonaws.com/android.telematics.sdk.production/")
    }
}

dependencies {
    implementation("com.telematicssdk:tracking:<version>")
}
```

Published install docs show dependency examples with `4.0.0`, while public changelog pages may show a different latest entry. Do not infer the dependency version from changelog order alone. Re-check the app dependency, the user-requested version, or available Maven metadata before changing a real app.

## Required Android Configuration

Check:

- `minSdk >= 23`.
- compile/target SDK compatible with the host app and SDK release.
- merged manifest retains SDK permissions/services/receivers/provider.
- packaging excludes include `META-INF/INDEX.LIST` and `META-INF/io.netty.versions.properties`.
- R8 keep rule exists when minification is enabled.
- runtime permission flow grants SDK requirements before enabling SDK.

## Initialization

Initialize once in `Application.onCreate()`:

```kotlin
val settings = Settings()
    .accuracy(Settings.accuracyHigh)
    .stopTrackingTimeout(Settings.stopTrackingTimeHigh)
    .autoStartOn(true)
    .adOn(false)
    .passiveDetectionOn(false)

TrackingApi.getInstance().initialize(applicationContext, settings)
```

Use application context. Avoid `Activity` context for initialization.

## Device Identity

Set the SDK device ID from the app's login/session binding flow before enabling automatic SDK collection or starting manual tracking:

```kotlin
val api = TrackingApi.getInstance()
if (api.getDeviceId() != deviceId) {
    api.setDeviceID(deviceId)
}
```

The device ID should come from the product backend or Damoov platform integration and is normally GUID-formatted. Do not generate it locally unless the product backend explicitly proxies or delegates the Damoov platform value to the app. Do not pass the device ID through start/stop tracking methods; keep `setDeviceId(...)` as the explicit identity/session method.

## Tracking Flows

These are app-level flows that decide when SDK collection or a manual trip should start and stop. Do not confuse them with SDK tracking modes (`TrackingMode.Standard` and `TrackingMode.Persistent`), which configure how the SDK tracks after a flow enables or starts tracking.

When implementing a reusable `TelematicsRepository`, ask which flow is primary if the user has not already specified it. Implement all supported flows, but place the primary flow first with a `// Primary Flow: ...` section and place the rest below it as `// Additional Flow: ...` sections.

Supported flows:

- automatic tracking
- standard manual tracking without tags
- standard manual tracking with tags
- app-controlled persistent manual tracking without tags
- app-controlled persistent manual tracking with tags
- one-time persistent manual tracking without tags
- one-time persistent manual tracking with tags

## High-Level Operating Mode Use Cases

Expose these product-level modes as use cases only when the user asks for use cases after the architecture question. A matching domain/use-case layer means use cases are recommended and should follow the host app's existing pattern, but it does not remove the requirement to ask. If no domain/use-case layer exists and the user chooses use cases, create a minimal app-appropriate use-case/workflow package and keep the classes plain unless the app already has a base use-case abstraction. These modes come from the Android tracking modes guide and are app-level policies over SDK enablement/tracking calls. Use cases should orchestrate `TelematicsRepository`; they should not call `TrackingApi` directly.

Use cases that enable or start tracking assume the SDK device ID has already been configured through `setDeviceId(...)` or a prepare/session-binding use case. Do not add `deviceId` parameters to start/enable use cases.

If the app has a trip-recording mode selector, persist the selected mode and active state in the host app's preferences/settings layer. Use cases should read the previous state before deciding how to transition and write the new state after SDK operations succeed.

```kotlin
enum class TripRecordMode {
    ALWAYS_ON,
    SHIFT_MODE,
    ON_DEMAND,
    DISABLED
}

data class TripRecordModeState(
    val mode: TripRecordMode,
    val isActive: Boolean
)

interface TelematicsModePreferencesRepository {
    suspend fun getTripRecordModeState(): Result<TripRecordModeState>
    suspend fun setTripRecordModeState(state: TripRecordModeState): Result<Unit>
}
```

Automatic mode:

```kotlin
class EnableAutomaticModeUseCase @Inject constructor(
    private val telematicsRepository: TelematicsRepository
) {
    suspend operator fun invoke() {
        telematicsRepository.enableAutomaticTracking().getOrThrow()
    }
}
```

Disabled mode:

```kotlin
class EnableDisabledModeUseCase @Inject constructor(
    private val telematicsRepository: TelematicsRepository
) {
    suspend operator fun invoke() {
        telematicsRepository.disableSdk().getOrThrow()
    }
}
```

On-demand mode default state:

```kotlin
data class DeviceIdParams(val deviceId: String)

class PrepareOnDemandModeUseCase @Inject constructor(
    private val telematicsRepository: TelematicsRepository
) {
    suspend operator fun invoke(parameters: DeviceIdParams) {
        telematicsRepository.disableSdk().getOrThrow()
        telematicsRepository.setDeviceId(parameters.deviceId).getOrThrow()
    }
}
```

On-demand trip control:

```kotlin
data class OnDemandTripParams(
    val maxIntervalMinutes: Int
)

class StartOnDemandTripUseCase @Inject constructor(
    private val telematicsRepository: TelematicsRepository
) {
    suspend operator fun invoke(parameters: OnDemandTripParams) {
        telematicsRepository.startOneTimePersistentManualTracking(
            maxIntervalMinutes = parameters.maxIntervalMinutes
        ).getOrThrow()
    }
}

class StopOnDemandTripUseCase @Inject constructor(
    private val telematicsRepository: TelematicsRepository
) {
    suspend operator fun invoke() {
        telematicsRepository.stopOneTimePersistentManualTracking().getOrThrow()
    }
}
```

Shift mode:

```kotlin
class SignOnShiftUseCase @Inject constructor(
    private val telematicsRepository: TelematicsRepository
) {
    suspend operator fun invoke() {
        telematicsRepository.enableAutomaticTracking().getOrThrow()
    }
}

class SignOffShiftUseCase @Inject constructor(
    private val telematicsRepository: TelematicsRepository
) {
    suspend operator fun invoke() {
        telematicsRepository.disableAutomaticTracking().getOrThrow()
    }
}
```

Use these use cases as named app workflows. Keep standard/persistent/manual repository methods available for lower-level control. If the host app has no domain layer, expose these workflows through the app's existing structure, for example as `TelematicsRepository` methods, coordinator methods, or plain application-layer classes.

Mode selector orchestration:

```kotlin
data class SetTripRecordModeParams(
    val mode: TripRecordMode,
    val isActive: Boolean,
    val persistentIntervalMinutes: Int
)

class SetTripRecordModeUseCase @Inject constructor(
    private val telematicsRepository: TelematicsRepository,
    private val modePreferencesRepository: TelematicsModePreferencesRepository
) {
    suspend operator fun invoke(parameters: SetTripRecordModeParams) {
        val current = modePreferencesRepository.getTripRecordModeState().getOrThrow()

        when (parameters.mode) {
            TripRecordMode.ALWAYS_ON -> {
                if (current.mode != TripRecordMode.ALWAYS_ON) {
                    telematicsRepository.enableAutomaticTracking().getOrThrow()
                }
            }

            TripRecordMode.SHIFT_MODE -> {
                if (current.mode == TripRecordMode.SHIFT_MODE && parameters.isActive) {
                    telematicsRepository.enableSdk().getOrThrow()
                } else {
                    if (current.mode == TripRecordMode.SHIFT_MODE) {
                        telematicsRepository.disableAutomaticTracking().getOrThrow()
                    } else if (telematicsRepository.isSdkEnabled()) {
                        telematicsRepository.disableAutomaticTracking().getOrThrow()
                    }
                }
            }

            TripRecordMode.ON_DEMAND -> {
                if (current.mode == TripRecordMode.ON_DEMAND && parameters.isActive) {
                    telematicsRepository.startOneTimePersistentManualTracking(
                        maxIntervalMinutes = parameters.persistentIntervalMinutes
                    ).getOrThrow()
                } else {
                    if (current.mode == TripRecordMode.ON_DEMAND) {
                        telematicsRepository.stopOneTimePersistentManualTracking().getOrThrow()
                    } else if (telematicsRepository.isSdkEnabled()) {
                        telematicsRepository.disableAutomaticTracking().getOrThrow()
                    }
                }
            }

            TripRecordMode.DISABLED -> {
                if (current.mode != TripRecordMode.DISABLED) {
                    when (current.mode) {
                        TripRecordMode.ON_DEMAND ->
                            telematicsRepository.stopOneTimePersistentManualTracking().getOrThrow()
                        TripRecordMode.SHIFT_MODE, TripRecordMode.ALWAYS_ON ->
                            telematicsRepository.disableAutomaticTracking().getOrThrow()
                        TripRecordMode.DISABLED -> Unit
                    }
                }
            }
        }

        modePreferencesRepository.setTripRecordModeState(
            TripRecordModeState(
                mode = parameters.mode,
                isActive = parameters.isActive
            )
        ).getOrThrow()
    }
}
```

Do not put this mode selector branching into `TelematicsRepository`; repositories should provide the SDK operations and preference access that the use case orchestrates.

Logout orchestration:

```kotlin
data class LogoutParams(
    val persistentIntervalMinutes: Int
)

class LogoutUseCase @Inject constructor(
    private val setTripRecordModeUseCase: SetTripRecordModeUseCase,
    private val telematicsRepository: TelematicsRepository
) {
    suspend operator fun invoke(parameters: LogoutParams) {
        setTripRecordModeUseCase(
            SetTripRecordModeParams(
                mode = TripRecordMode.DISABLED,
                isActive = false,
                persistentIntervalMinutes = parameters.persistentIntervalMinutes
            )
        )
        telematicsRepository.logout().getOrThrow()
    }
}
```

Pass the host app's configured/default persistent interval into the use case. The ordering is required: disable and persist trip recording mode first, then call SDK `logout()`.

## Common Helpers

Use idempotent helper sequencing like this inside the app-owned facade. Read SDK state first and call SDK mutators only when the current value differs. Use `ensureDeviceId(...)` from the public `setDeviceId(...)` method, not from tracking start methods:

```kotlin
private fun ensureDeviceId(api: TrackingApi, deviceId: String): TelematicsResult {
    if (api.getDeviceId() != deviceId) {
        api.setDeviceID(deviceId)
    }
    return if (api.isDeviceIdValid()) TelematicsResult.Success else TelematicsResult.InvalidDeviceId
}

private fun enableSdk(): TelematicsResult {
    val api = TrackingApi.getInstance()
    if (!api.isInitialized()) return TelematicsResult.NotInitialized
    if (!api.isAllRequiredPermissionsAndSensorsGranted()) return TelematicsResult.PermissionsMissing
    if (!api.isSdkEnabled()) {
        api.setEnableSdk(true)
    }
    return TelematicsResult.Success
}

private fun disableSdk(): TelematicsResult {
    val api = TrackingApi.getInstance()
    if (api.isSdkEnabled()) {
        api.setEnableSdk(false)
    }
    return TelematicsResult.Success
}

private fun logout(): TelematicsResult {
    TrackingApi.getInstance().logout()
    return TelematicsResult.Success
}

private fun configurePersistentInterval(minutes: Int): TelematicsResult {
    if (minutes !in 5..600) return TelematicsResult.InvalidPersistentInterval
    val api = TrackingApi.getInstance()
    if (api.getMaxPersistentTrackingInterval() != minutes) {
        api.setMaxPersistentTrackingInterval(minutes)
    }
    return TelematicsResult.Success
}

private fun configureTrackingMode(mode: TrackingMode): TelematicsResult {
    val api = TrackingApi.getInstance()
    if (api.getTrackingMode() != mode) {
        api.setTrackingMode(mode)
    }
    return TelematicsResult.Success
}
```

Use the host app's existing result/error style instead of introducing this exact type when a project pattern exists.

## Automatic Tracking

```kotlin
val api = TrackingApi.getInstance()
if (api.isAllRequiredPermissionsAndSensorsGranted()) {
    enableSdk()
}
```

Automatic tracking depends on `Settings().autoStartOn(true)` or later `setAutoStartEnabled(true, permanent)` depending on product behavior.

Automatic tracking stop:

```kotlin
disableSdk()
```

Disable SDK collection while keeping the device ID:

```kotlin
TrackingApi.getInstance().setEnableSdk(false)
```

Logout, when device ID removal is intended:

```kotlin
TrackingApi.getInstance().logout()
```

`logout()` clears device identity. Use it only for logout/account removal semantics.

## Standard Manual Tracking Without Tags

```kotlin
val api = TrackingApi.getInstance()
enableSdk()
configureTrackingMode(TrackingMode.Standard)
val started = api.startTracking()
```

Handle `started == false`; it can indicate missing preconditions or an already active/rejected session.

Standard manual stop without tags:

```kotlin
val stopped = TrackingApi.getInstance().stopTracking()
disableSdk()
```

## Standard Manual Tracking With Tags

For cross-platform manual tagged flows, "with tags" means future tags added before the upcoming manually started trip.

```kotlin
val api = TrackingApi.getInstance()
enableSdk()
configureTrackingMode(TrackingMode.Standard)

tags.forEach { tag ->
    api.addFutureTrackTag(tag = tag.name, source = tag.source)
}

// Start only after TagsProcessingListener/TagsProcessingReceiver confirms required tags.
val started = api.startTracking()
```

Starting tracking before required future tags are confirmed is a race. If product requirements tolerate best-effort tags, document that tradeoff in the repository method KDoc.

Standard manual stop with future-tag cleanup:

```kotlin
api.removeAllFutureTrackTags()
// Wait for cleanup status when cleanup must be guaranteed before stop/disable.
val stopped = api.stopTracking()
disableSdk()
```

## SDK Tracking Modes

`TrackingMode.Standard`

- Normal SDK tracking behavior.
- Manual `startTracking()` produces a standard trip.
- Tracking stops with `stopTracking()` or the SDK's standard stop detector.

`TrackingMode.Persistent`

- Persistent mode for trips started in manual or automatic mode.
- Track recording continues until `stopTracking()` or the maximum persistent tracking interval is reached.
- Allowed interval: `5..600` minutes.
- Default interval: `240` minutes.

Set/read mode:

```kotlin
if (api.getTrackingMode() != TrackingMode.Persistent) {
    api.setTrackingMode(TrackingMode.Persistent)
}
```

Set/read interval:

```kotlin
if (api.getMaxPersistentTrackingInterval() != 240) {
    api.setMaxPersistentTrackingInterval(240)
}
```

## App-Controlled Persistent Manual Tracking

Without tags:

```kotlin
val api = TrackingApi.getInstance()
enableSdk()
configurePersistentInterval(minutes)
configureTrackingMode(TrackingMode.Persistent)
val started = api.startTracking()
```

With tags:

```kotlin
val api = TrackingApi.getInstance()
enableSdk()

tags.forEach { tag ->
    api.addFutureTrackTag(tag = tag.name, source = tag.source)
}

// Wait for required tag confirmations before starting.
configurePersistentInterval(minutes)
configureTrackingMode(TrackingMode.Persistent)
val started = api.startTracking()
```

App-controlled persistent stop without tags:

```kotlin
val stopped = api.stopTracking()
configureTrackingMode(TrackingMode.Standard)
disableSdk()
```

App-controlled persistent stop with future-tag cleanup:

```kotlin
api.removeAllFutureTrackTags()
// Wait for cleanup result if cleanup must be guaranteed before disable.
val stopped = api.stopTracking()
configureTrackingMode(TrackingMode.Standard)
disableSdk()
```

Always reset `TrackingMode.Standard` after app-controlled persistent sessions unless the product explicitly wants future sessions to remain persistent.

## One-Time Persistent Manual Tracking

Without tags:

```kotlin
val api = TrackingApi.getInstance()
enableSdk()
configurePersistentInterval(minutes)
val started = api.startTrackAsPersistent()
```

With tags:

```kotlin
val api = TrackingApi.getInstance()
enableSdk()

tags.forEach { tag ->
    api.addFutureTrackTag(tag = tag.name, source = tag.source)
}

// Wait for required tag confirmations before starting.
configurePersistentInterval(minutes)
val started = api.startTrackAsPersistent()
```

One-time persistent stop without tags:

```kotlin
val stopped = api.stopTracking()
disableSdk()
```

One-time persistent stop with future-tag cleanup:

```kotlin
api.removeAllFutureTrackTags()
// Wait for cleanup result if cleanup must be guaranteed before disable.
val stopped = api.stopTracking()
disableSdk()
```

Do not call deprecated `startPersistentTracking()`. Do not use app-controlled `setTrackingMode(TrackingMode.Persistent)` / `setTrackingMode(TrackingMode.Standard)` to implement one-time persistent behavior unless the resolved SDK version lacks `startTrackAsPersistent()`.

Prefer explicit stop methods for app-controlled persistent flows when the repository needs to restore `TrackingMode.Standard`; do not add hidden state only to infer which stop path is active.

## Tags Repository

Future tag methods are asynchronous by result delivery:

```kotlin
api.getFutureTrackTags()
api.addFutureTrackTag(tag = "work", source = "app")
api.removeFutureTrackTag("work")
api.removeAllFutureTrackTags()
```

Register a `TagsProcessingListener` or `TagsProcessingReceiver` before issuing tag operations whose result matters.

`tag` and `source` are product-defined strings. The SDK does not define a fixed enum or SDK-side value list for product tags. Use `tag` for the business label and `source` for the app module or user action that created it.

Future tag callbacks return SDK status. Preserve or map that status into the app's result/error model, especially for manual start sequencing. Treat success as the only guaranteed state for required tags; offline, invalid-device, backend rejection, and generic operation failures should not be silently converted to success.

Processed-trip tag APIs are separate post-trip operations:

```kotlin
api.getTrackTags(trackId)
api.addTrackTags(trackId, tags)
api.removeTrackTags(trackId, tags)
```

Keep processed-trip tag methods in `TelematicsTagsRepository`; do not mix them into tracking start flow methods. For tagged manual starts, `TelematicsRepository` should use `TelematicsTagsRepository` or a shared private helper for future tags instead of duplicating tag calls in UI code.

## Events Repository

Move SDK callbacks, listeners, and event receivers into `TelematicsEventsRepository`:

```kotlin
api.registerCallback(listener)
api.unregisterCallback(listener)
api.setLocationListener(listener)
api.getLocationListener()
api.registerTrackingEventsReceiver(receiver)
api.unregisterTrackingEventsReceiver()
api.registerSpeedViolations(speedLimitKmH, speedLimitTimeoutMs, listener)
api.unregisterSpeedViolations()
api.isSpeedViolationsRegistered()
api.setSpeedLimit(speedLimitKmH)
api.setSpeedLimitTimeout(speedLimitTimeoutMs)
api.getSpeedLimit()
api.getTimeouts()
```

`TrackingStateListener` is an in-process start/stop listener:

```kotlin
object : TrackingStateListener {
    override fun onStartTracking() = Unit
    override fun onStopTracking() = Unit
}
```

`LocationListener` is an in-process location listener:

```kotlin
object : LocationListener {
    override fun onLocationChanged(location: Location?) = Unit
}
```

`TrackingEventsReceiver` is a broadcast-style receiver:

```kotlin
class AppTrackingEventsReceiver : TrackingEventsReceiver() {
    override fun onLocationChanged(context: Context, location: Location) = Unit
    override fun onStartTracking(context: Context) = Unit
    override fun onStopTracking(context: Context) = Unit
    override fun onSpeedViolation(context: Context, violation: SpeedViolation) = Unit
    override fun onNewEvents(context: Context, events: Array<Event>) = Unit
    override fun onSdkDeprecated(context: Context) = Unit
}
```

Register receiver classes through `TelematicsEventsRepository`, not directly from UI. Keep receiver bodies thin: log, forward, or notify app state only. Do not add business logic directly inside receiver callbacks unless explicitly requested.

Only enable speed violation registration when the user/product supplies `speedLimitKmH` and `speedLimitTimeoutMs`; generated default integrations should expose wrappers but not invent thresholds.

## Trips Repository

Move these into `TelematicsTripsRepository`:

```kotlin
api.getTracks(locale, startDate, endDate, offset, count)
api.getTrackDetails(trackId, locale)
api.getTrackOriginDict(locale)
api.changeTrackOrigin(trackToken, newCode)
api.uploadUnsentTrips()
api.getUnsentTripCount()
api.getDashboardInfo(trackTag)
api.getMainStatistics(trackTag)
api.shareTrack(trackToken, isShared)
api.getSharedTrackDetails(trackToken)
```

Do not call blocking APIs on the main thread. Use the host app's existing coroutine dispatcher, Rx scheduler, executor, or repository pattern.

## Review Checklist

- `TrackingApi.initialize(...)` happens once in `Application.onCreate()`.
- `Settings` uses SDK 4.x builder methods.
- Device ID is set before SDK enablement or tracking start.
- Runtime permissions and sensors are granted before `setEnableSdk(true)`.
- Manual start/stop `Boolean` results are handled.
- Tagged manual starts wait for tag processing when tags are required.
- App-controlled persistent sessions reset `TrackingMode.Standard`.
- One-time persistent sessions use `startTrackAsPersistent()`.
- `startPersistentTracking()`, `setDeviceToken`, old `Settings(...)`, and removed SDK 3.x APIs are absent.
- Blocking trip/statistics calls are off the main thread.
- User-visible strings are read from the app's `Settings`/localization layer.
