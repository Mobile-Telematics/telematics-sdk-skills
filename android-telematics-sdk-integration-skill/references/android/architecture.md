# Recommended App Architecture

## Decision

Wrap `TrackingApi.getInstance()` in app-owned repositories. Do not scatter SDK calls across activities, fragments, Composables, reducers, interactors, or view models.

`TrackingApi` is the immediate external source of telematics data and commands. Because each repository has one SDK-backed source, do not add a separate `TelematicsDataSource` layer by default. Android's data layer guidance allows merging repository and data source responsibilities when a repository has a single source; split later only if the app adds another source such as local cache, backend sync, fake implementation, or another provider.

## Suggested Boundaries

Use names that match the host app conventions, but keep these responsibilities clear:

- `TelematicsRepository`: app-facing data layer entry point for SDK initialization support, device ID setup, SDK enable/disable, low-level tracking flows, status, permission/sensor checks, SDK permissions wizard intent creation, diagnostics, notification intent, accident detection, passive detection, RTD access when requested, and heartbeats.
- `TelematicsEventsRepository`: app-facing data layer entry point for public SDK callbacks/listeners/receivers: `TrackingStateListener`, `LocationListener`, `TrackingEventsReceiver`, and speed-violation listener/threshold controls when requested.
- `TelematicsTagsRepository`: app-facing data layer entry point for future tags, processed-trip tags, tag callbacks, and tag receivers.
- `TelematicsTripsRepository`: app-facing data layer entry point for track list/details, unsent trips, upload, origin dictionary/change, statistics/dashboard, share/unshare, and shared-track details.
- `TelematicsModePreferencesRepository`: app-facing preferences/settings entry point for persisted trip recording mode state, e.g. `(TripRecordMode, isActive)`. Implement it with the host app's existing DataStore, SharedPreferences, database, settings repository, or local data source.
- `TrackingApi`: external SDK data source. It is called by the telematics repositories, not by UI or domain layers.
- `TelematicsPermissionCoordinator`: UI/app-layer coordinator for runtime permission requests and SDK `PermissionsWizardActivity` launch/result handling. It can call repository enable methods or use cases after permissions are granted. In simple apps this may be a small Activity/Compose launcher example instead of a separate class, but generated integration must show how to launch the SDK wizard.
- `TrackingSessionRequest`: app-level input for starting tracking, e.g. flow, optional tag/source, optional persistent interval. Device identity is configured separately from tracking start/stop flows.
- `TrackingFlow`: app-level flow such as automatic enablement, standard manual start/stop, app-controlled persistent manual start/stop, or one-time persistent manual start/stop, with or without tags.
- `TelematicsError`: app-level error mapping for permission failures, invalid device ID, disabled SDK state, rejected start/stop calls, and tag processing failures.

Avoid putting business decisions into `Application` or the launch activity. `Application` should initialize the SDK or delegate initialization to the repository/DI startup. UI layers should call use cases for product workflows and repositories for lower-level data access when the app already permits direct repository use. UI must not call `TrackingApi` directly.

High-level operating modes from the Android tracking modes guide are app/domain workflows, not SDK data-access primitives. Ask the user whether to add use cases before deciding between repository-only integration and generated use cases. The question is mandatory for a new/full telematics integration in every host app, including apps with no current domain layer, unless the user has already answered it in the current request. Existing domain/use-case classes are a reason to recommend use cases, not a reason to skip the question. Absence of a domain/use-case layer is not a reason to skip use cases; if the user chooses use cases, create a minimal use-case/workflow layer that matches the app structure. The question must list the proposed use cases and briefly describe each one:

```text
Do you want me to add use cases for telematics workflows, or keep the integration repository-only?

Proposed use cases:
- EnableAutomaticModeUseCase: enable automatic SDK tracking after the device ID has already been configured.
- EnableDisabledModeUseCase: stop active tracking if needed and disable SDK collection.
- PrepareOnDemandModeUseCase: save/prepare the device ID while keeping SDK collection disabled until the user starts a trip.
- StartOnDemandTripUseCase: start a one-time persistent manual trip.
- StopOnDemandTripUseCase: stop the active on-demand trip.
- SignOnShiftUseCase: start shift/driver-on-duty tracking.
- SignOffShiftUseCase: stop shift tracking and disable SDK collection.
- SetTripRecordModeUseCase: switch ALWAYS_ON / SHIFT_MODE / ON_DEMAND / DISABLED and persist the selected mode state.
- GetTripRecordModeUseCase: read the persisted trip recording mode state.
- LogoutUseCase: set trip recording mode to DISABLED/inactive, then call SDK logout.
```

If the host app already has a suitable domain/use-case layer, recommend use cases in the question and adapt to the existing base classes, DI, dispatcher, and result patterns after the user confirms. If the host app has no domain/use-case layer, still ask; if the user wants use cases, create a small telematics workflow/use-case package in the appropriate module or package for the app. If the user chooses repository-only integration, keep these workflows out of generated use-case classes and expose only the requested repository/coordinator API. If the user wants use cases, implement them as use cases that orchestrate `TelematicsRepository`. In a single-module app, keep the same workflow names and behavior in a simple `domain`, `usecase`, or `telematics` workflow package rather than forcing a new module:

- `EnableAutomaticModeUseCase`
- `EnableDisabledModeUseCase`
- `PrepareOnDemandModeUseCase`
- `StartOnDemandTripUseCase`
- `StopOnDemandTripUseCase`
- `SignOnShiftUseCase`
- `SignOffShiftUseCase`
- `LogoutUseCase`
- `SetTripRecordModeUseCase`
- `GetTripRecordModeUseCase`

Use the project's existing use-case pattern when available and when use cases are requested. During inspection, explicitly check for existing `UseCase`, `CoroutineUseCase`, `FlowUseCase`, interactor, command, or domain workflow classes; if found, include that observation in the question and recommend use cases. If no such pattern exists and the user chooses use cases, create minimal plain Kotlin use-case classes with constructor-injected repositories and adapt DI only when the host app already uses DI. Keep repositories focused on SDK operations and SDK state; keep product trip-recording mode decisions in use cases when use cases are part of the integration.

Inside repositories, separate responsibilities with sections and private helpers rather than service/data-source classes:

- initialization and settings
- permission/status checks
- automatic tracking
- manual tracking
- persistent tracking
- listeners/receivers
- diagnostics/upload

Do not implement a large `setTripRecordMode(mode, isActive)` business branch inside `TelematicsRepository` when use cases are requested or already present. Put that transition logic in use cases that orchestrate `TelematicsRepository` and `TelematicsModePreferencesRepository`. In repository-only integrations, keep any mode-selector coordinator small and outside UI.

Do not create `TelematicsService`, `TelematicsTagsService`, or `TelematicsTripsService` for the default architecture. Use repository names so the integration matches Android data-layer conventions while still fitting the host app's existing modular or single-module structure.

## Minimal Kotlin Interface

```kotlin
interface TelematicsRepository {
    /** Initializes the SDK once from Application.onCreate() or app startup. */
    fun initialize(context: Context, settings: Settings = defaultSettings())

    /** Assigns the SDK device ID without enabling collection. */
    fun setDeviceId(deviceId: String): TelematicsResult

    /** Returns the current SDK device ID. */
    fun getDeviceId(): String

    /** Returns whether the current SDK device ID is valid. */
    fun isDeviceIdValid(): Boolean

    /** Enables SDK collection when permissions and sensors are already granted. */
    fun enableSdk(): TelematicsResult

    /** Disables SDK collection without clearing the current device ID. */
    fun disableSdk(): TelematicsResult

    /** Returns whether SDK collection is currently enabled. */
    fun isSdkEnabled(): Boolean

    /** Performs SDK logout and clears SDK-side identity when the product requests logout semantics. */
    fun logout(): TelematicsResult

    /** Enables automatic SDK tracking. Requires a valid device ID to have been configured separately. */
    fun enableAutomaticTracking(): TelematicsResult

    /** Stops automatic tracking by disabling SDK collection while preserving the device ID. */
    fun disableAutomaticTracking(): TelematicsResult

    /** Starts a standard manual tracking session without future tags. */
    fun startStandardManualTracking(): TelematicsResult

    /** Stops a standard manual tracking session without future-tag cleanup. */
    fun stopStandardManualTracking(): TelematicsResult

    /** Adds future tags and starts a standard manual tracking session. */
    suspend fun startStandardManualTrackingWithTags(
        tags: List<FutureTrackTagRequest>
    ): TelematicsResult

    /** Removes future tags when required, then stops a tagged standard manual session. */
    suspend fun stopStandardManualTrackingWithTags(): TelematicsResult

    /** Starts an app-controlled persistent manual tracking session without future tags. */
    fun startPersistentManualTracking(
        maxIntervalMinutes: Int
    ): TelematicsResult

    /** Stops app-controlled persistent manual tracking and restores standard SDK mode. */
    fun stopPersistentManualTracking(): TelematicsResult

    /** Adds future tags and starts an app-controlled persistent manual tracking session. */
    suspend fun startPersistentManualTrackingWithTags(
        tags: List<FutureTrackTagRequest>,
        maxIntervalMinutes: Int
    ): TelematicsResult

    /** Removes future tags, stops app-controlled persistent manual tracking, and restores standard SDK mode. */
    suspend fun stopPersistentManualTrackingWithTags(): TelematicsResult

    /** Starts a one-time persistent manual tracking session without future tags. */
    fun startOneTimePersistentManualTracking(
        maxIntervalMinutes: Int
    ): TelematicsResult

    /** Stops one-time persistent manual tracking without manually restoring standard SDK mode. */
    fun stopOneTimePersistentManualTracking(): TelematicsResult

    /** Adds future tags and starts a one-time persistent manual tracking session. */
    suspend fun startOneTimePersistentManualTrackingWithTags(
        tags: List<FutureTrackTagRequest>,
        maxIntervalMinutes: Int
    ): TelematicsResult

    /** Removes future tags and stops one-time persistent manual tracking without manually restoring standard SDK mode. */
    suspend fun stopOneTimePersistentManualTrackingWithTags(): TelematicsResult

    /** Returns whether SDK tracking is currently active. */
    fun isTracking(): Boolean

    /** Returns automatic/manual tracking availability state. */
    fun getTrackingState(): TrackingState

    /** Returns the latest cached device ID registration state. */
    fun getDeviceIdRegistrationState(): DeviceIdRegistrationState

    /** Returns whether all SDK-required runtime permissions and sensors are granted. */
    fun isAllRequiredPermissionsAndSensorsGranted(): Boolean

    /** Returns whether all SDK-required runtime permissions are granted. */
    fun isAllRequiredPermissionsGranted(): Boolean

    /** Creates the SDK permissions wizard intent for Activity Result API launch. */
    fun createPermissionsWizardIntent(
        context: Context,
        enableAggressivePermissionsWizard: Boolean = false,
        enableAggressivePermissionsWizardPage: Boolean = false,
    ): Intent

    /** Sends a custom SDK heartbeat with an app-defined reason. */
    fun sendCustomHeartbeats(reason: String): TelematicsResult
}
```

Keep each flow's stop method adjacent to its start method in implementations. The stop must match that flow's SDK ownership: automatic tracking disables SDK collection, standard manual tracking stops and disables collection, tagged manual tracking removes future tags before stopping when cleanup is required, app-controlled persistent tracking also restores `TrackingMode.Standard`, and one-time persistent tracking leaves SDK mode reset ownership to `startTrackAsPersistent()` semantics.

Expose permission wizard launch from UI without moving SDK calls into the UI. The repository creates the SDK intent; Activity or Compose code owns the launcher and result handling:

```kotlin
class TelematicsPermissionCoordinator(
    private val telematicsRepository: TelematicsRepository
) {
    fun createPermissionsWizardIntent(
        context: Context,
        enableAggressivePermissionsWizard: Boolean = false,
        enableAggressivePermissionsWizardPage: Boolean = false,
    ): Intent =
        telematicsRepository.createPermissionsWizardIntent(
            context = context,
            enableAggressivePermissionsWizard = enableAggressivePermissionsWizard,
            enableAggressivePermissionsWizardPage = enableAggressivePermissionsWizardPage,
        )

    fun mapWizardResult(resultCode: Int): TelematicsPermissionWizardResult =
        when (resultCode) {
            PermissionsWizardActivity.WIZARD_RESULT_ALL_GRANTED -> TelematicsPermissionWizardResult.AllGranted
            PermissionsWizardActivity.WIZARD_RESULT_NOT_ALL_GRANTED -> TelematicsPermissionWizardResult.NotAllGranted
            else -> TelematicsPermissionWizardResult.Canceled
        }
}

sealed interface TelematicsPermissionWizardResult {
    data object AllGranted : TelematicsPermissionWizardResult
    data object NotAllGranted : TelematicsPermissionWizardResult
    data object Canceled : TelematicsPermissionWizardResult
}
```

For simple apps, it is acceptable to skip a separate coordinator class and put the Activity Result launcher in Activity/Compose, but generated integration must include a concrete `createPermissionsWizardIntent(...)` call path.

Persist app-level mode state separately from SDK state:

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
    /** Returns the last selected app-level trip recording mode. */
    suspend fun getTripRecordModeState(): Result<TripRecordModeState>

    /** Persists the selected app-level trip recording mode after a successful transition. */
    suspend fun setTripRecordModeState(state: TripRecordModeState): Result<Unit>
}
```

High-level operating mode use cases should call these lower-level repository methods rather than duplicating `TrackingApi` calls:

```kotlin
class EnableAutomaticModeUseCase @Inject constructor(
    private val telematicsRepository: TelematicsRepository
) {
    suspend operator fun invoke() {
        telematicsRepository.enableAutomaticTracking().getOrThrow()
    }
}

class EnableDisabledModeUseCase @Inject constructor(
    private val telematicsRepository: TelematicsRepository
) {
    suspend operator fun invoke() {
        telematicsRepository.disableSdk().getOrThrow()
    }
}
```

For apps with a single mode selector, implement a central mode transition use case. It should read the current persisted mode before deciding what to do and persist the new state only after the SDK transition succeeds:

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

Account logout should also be a use case when use cases are part of the integration. Disable trip recording first, persist that mode state through `SetTripRecordModeUseCase`, then call SDK logout:

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

Pass the app's configured/default persistent interval into `LogoutUseCase`; do not invent a new interval value inside the use case.

Adapt the exact branches to product semantics, but preserve the boundary: use cases own the mode transition, repositories own SDK calls and persisted settings access.

Adapt names, dispatchers, DI annotations, base classes, and return types to the host app. The important boundary is: product-mode orchestration belongs in use cases or application-layer workflow classes when that layer exists; direct SDK calls stay in repositories.

Events repository:

```kotlin
interface TelematicsEventsRepository {
    /** Registers an in-process tracking state listener. */
    fun registerTrackingStateListener(listener: TrackingStateListener): Result<Boolean>

    /** Unregisters a previously registered tracking state listener. */
    fun unregisterTrackingStateListener(listener: TrackingStateListener): Result<Boolean>

    /** Sets or clears the in-process SDK location listener. */
    fun setLocationListener(listener: LocationListener?): Result<Unit>

    /** Returns the current in-process SDK location listener, if any. */
    fun getLocationListener(): LocationListener?

    /** Registers a TrackingEventsReceiver subclass for SDK events. */
    fun registerTrackingEventsReceiver(receiver: Class<out TrackingEventsReceiver>): Result<Unit>

    /** Unregisters the TrackingEventsReceiver subclass. */
    fun unregisterTrackingEventsReceiver(): Result<Unit>

    /** Registers speed violation monitoring when product thresholds are known. */
    fun registerSpeedViolations(
        speedLimitKmH: Float,
        speedLimitTimeoutMs: Long,
        listener: SpeedViolationsListener
    ): Result<Boolean>

    /** Unregisters speed violation monitoring. */
    fun unregisterSpeedViolations(): Result<Boolean>

    /** Returns whether speed violation monitoring is registered. */
    fun isSpeedViolationsRegistered(): Boolean

    /** Sets the current speed limit threshold. */
    fun setSpeedLimit(speedLimitKmH: Float): Result<Unit>

    /** Sets the speed limit timeout threshold. */
    fun setSpeedLimitTimeout(speedLimitTimeoutMs: Long): Result<Unit>

    /** Returns the current speed limit threshold. */
    fun getSpeedLimit(): Float

    /** Returns the current speed limit timeout threshold. */
    fun getSpeedLimitTimeout(): Long
}
```

`TrackingEventsReceiver` subclasses must implement:

```kotlin
override fun onLocationChanged(context: Context, location: Location)
override fun onStartTracking(context: Context)
override fun onStopTracking(context: Context)
override fun onSpeedViolation(context: Context, violation: SpeedViolation)
override fun onNewEvents(context: Context, events: Array<Event>)
override fun onSdkDeprecated(context: Context)
```

Use `TrackingStateListener` for in-process start/stop callbacks, `LocationListener` for in-process location callbacks, and `TrackingEventsReceiver` when the app needs broadcast-style event delivery. Do not put business decisions directly inside receiver/listener callbacks; forward into app state, logs, or repository/domain handlers.

Tags repository:

```kotlin
interface TelematicsTagsRepository {
    /** Fetches future tags; result is delivered through tag callback or receiver. */
    fun getFutureTrackTags(): Result<Unit>

    /** Adds a future tag for upcoming trips. */
    fun addFutureTrackTag(tag: String?, source: String? = null): Result<Unit>

    /** Removes a future tag from upcoming trips. */
    fun removeFutureTrackTag(tag: String?): Result<Unit>

    /** Removes all future tags from upcoming trips. */
    fun removeAllFutureTrackTags(): Result<Unit>

    /** Returns tags attached to a processed trip. */
    suspend fun getTrackTags(trackId: String): Result<Array<TrackTag>>

    /** Adds tags to a processed trip and returns successfully added tags. */
    suspend fun addTrackTags(trackId: String, tags: Array<TrackTag>): Result<Array<TrackTag>>

    /** Removes tags from a processed trip and returns successfully removed tags. */
    suspend fun removeTrackTags(trackId: String, tags: Array<TrackTag>): Result<Array<TrackTag>>

    /** Registers an in-process callback for future tag operation results. */
    fun addTagsProcessingCallback(callback: TagsProcessingListener): Result<Unit>

    /** Removes the current in-process tag processing callback. */
    fun removeTagsProcessingCallback(): Result<Unit>

    /** Returns the current in-process tag processing callback, if present. */
    fun getTagsProcessingCallback(): TagsProcessingListener?

    /** Registers a broadcast receiver class for future tag operation results. */
    fun registerTagsReceiver(receiver: Class<*>): Result<Unit>

    /** Unregisters the tag processing broadcast receiver. */
    fun unregisterTagsReceiver(): Result<Unit>
}
```

`TagsProcessingListener` callbacks are:

```kotlin
fun onTagAdd(status: Status, tag: Tag, activationTime: Long)
fun onTagRemove(status: Status, tag: Tag, deactivationTime: Long)
fun onAllTagsRemove(status: Status, deactivatedTagsCount: Int, time: Long)
fun onGetTags(status: Status, tags: Array<Tag>?, time: Long)
```

`TagsProcessingReceiver` is the broadcast receiver form of the same callback contract. Use the listener for in-process orchestration and the receiver when the app needs background-capable tag result delivery.

Trips repository:

```kotlin
interface TelematicsTripsRepository {
    /** Returns cached trips using SDK pagination and optional date filters. */
    suspend fun getTracks(
        locale: Locale,
        startDate: String? = null,
        endDate: String? = null,
        offset: Int,
        limit: Int
    ): Result<Array<Track>>

    /** Returns cached details for a trip. */
    suspend fun getTrackDetails(trackId: String, locale: Locale): Result<TrackDetails>

    /** Returns locally stored trips waiting for upload. */
    suspend fun getUnsentTrips(): Result<Array<Track>>

    /** Returns the count of locally stored trips waiting for upload. */
    suspend fun getUnsentTripCount(): Result<Int>

    /** Requests upload of locally stored trips waiting for upload. */
    fun uploadUnsentTrips(): Result<Unit>

    /** Returns available trip origin dictionary entries. */
    suspend fun getTrackOriginDict(locale: Locale): Result<Array<TrackOriginDictionary>>

    /** Changes the origin for a processed trip. */
    suspend fun changeTrackOrigin(trackToken: String, newCode: String): Result<Boolean>

    /** Returns main dashboard statistics. */
    suspend fun getMainStatistics(trackTag: String? = null): Result<DashboardInfo>

    /** Returns driving score details. */
    suspend fun getDrivingDetailsStatistics(period: StatisticPeriod, trackTag: String? = null): Result<DrivingDetails>

    /** Returns driving time score details. */
    suspend fun getDrivingTimeDetailsStatistics(period: StatisticPeriod, trackTag: String? = null): Result<DrivingTimeDetails>

    /** Returns speed score details. */
    suspend fun getSpeedDetailStatistics(period: StatisticPeriod, trackTag: String? = null): Result<SpeedDetails>

    /** Returns mileage score details. */
    suspend fun getMileageDetailsStatistics(period: StatisticPeriod, trackTag: String? = null): Result<MileageDetails>

    /** Returns phone usage score details. */
    suspend fun getPhoneDetailStatistics(period: StatisticPeriod, trackTag: String? = null): Result<PhoneDetails>

    /** Shares or unshares a processed trip. */
    suspend fun shareTrack(trackToken: String, isShared: Boolean): Result<Boolean>

    /** Returns public details for a shared trip. */
    suspend fun getSharedTrackDetails(trackToken: String): Result<TrackDetails>
}
```

Use app-native result/error types when they already exist. For one-shot operations, prefer suspend functions when the app uses coroutines. For state changes over time, expose `Flow` where the host architecture expects observable data. If the app does not use coroutines, preserve callback-style APIs.

## Primary Flow Selection

Before implementing a new reusable `TelematicsRepository`, identify the user's primary flow. Ask directly when it is not specified:

```text
Which primary tracking flow should be placed first in the TelematicsRepository: automatic tracking, standard manual tracking without tags, standard manual tracking with tags, app-controlled persistent manual tracking without tags, app-controlled persistent manual tracking with tags, one-time persistent manual tracking without tags, or one-time persistent manual tracking with tags?
```

The repository should still include all supported flows, but the user-requested primary flow must be implemented first and marked explicitly:

```kotlin
// Primary Flow: App-controlled persistent manual tracking with tags
```

Put the other supported flows below it with separate sections:

```kotlin
// Additional Flow: Automatic tracking
// Additional Flow: Standard manual tracking without tags
// Additional Flow: Standard manual tracking with tags
// Additional Flow: App-controlled persistent manual tracking without tags
// Additional Flow: One-time persistent manual tracking without tags
// Additional Flow: One-time persistent manual tracking with tags
```

Keep the primary flow first so the app team can find and customize the business-critical path quickly. Share private helpers for common operations such as initialization checks, permission checks, device ID setup, SDK enablement, future-tag cleanup, and persistent-mode reset.

Keep device ID semantics explicit. The device ID should come from the product backend or Damoov platform integration and is normally GUID-formatted. Set it from login/session binding code through `setDeviceId(...)` before enabling SDK collection or starting manual tracking. Do not pass the device ID through start/stop flow methods. Do not add API-key or credentials setup to generated Android app code unless the resolved SDK version exposes and documents such app-facing credentials.

Repository implementations should be idempotent. Before calling SDK mutators, read the current SDK state and call the mutator only when a change is needed:

```kotlin
private fun ensureDeviceId(api: TrackingApi, deviceId: String): TelematicsResult {
    if (api.getDeviceId() != deviceId) {
        api.setDeviceID(deviceId)
    }
    return if (api.isDeviceIdValid()) TelematicsResult.Success else TelematicsResult.InvalidDeviceId
}

private fun ensureSdkEnabled(api: TrackingApi): TelematicsResult {
    if (!api.isAllRequiredPermissionsAndSensorsGranted()) return TelematicsResult.PermissionsMissing
    if (!api.isSdkEnabled()) {
        api.setEnableSdk(true)
    }
    return TelematicsResult.Success
}

private fun ensureSdkDisabled(api: TrackingApi): TelematicsResult {
    if (api.isSdkEnabled()) {
        api.setEnableSdk(false)
    }
    return TelematicsResult.Success
}

private fun ensureTrackingMode(api: TrackingApi, mode: TrackingMode): TelematicsResult {
    if (api.getTrackingMode() != mode) {
        api.setTrackingMode(mode)
    }
    return TelematicsResult.Success
}

private fun ensurePersistentInterval(api: TrackingApi, minutes: Int): TelematicsResult {
    if (minutes !in 5..600) return TelematicsResult.InvalidPersistentInterval
    if (api.getMaxPersistentTrackingInterval() != minutes) {
        api.setMaxPersistentTrackingInterval(minutes)
    }
    return TelematicsResult.Success
}
```

Use `ensureDeviceId(...)` from the public `setDeviceId(...)` repository method. Use `ensureSdkEnabled(...)`, `ensureTrackingMode(...)`, and `ensurePersistentInterval(...)` from higher-level methods such as `enableAutomaticTracking`, `startStandardManualTracking`, `startPersistentManualTracking`, and one-time persistent start methods. Keep `logout()` separate from `disableSdk()`.

For future tags, preserve or map SDK callback/receiver statuses instead of treating every completion as success. `tag` and `source` are product-defined strings; validate product-specific allowed values outside the SDK wrapper when needed.

## Architecture Checks

When reviewing an integration, flag these issues:

- Direct `TrackingApi.getInstance()` calls outside `TelematicsRepository` or SDK initialization startup code.
- Missing `TelematicsEventsRepository`, `TelematicsTagsRepository`, or `TelematicsTripsRepository` in a complete SDK integration.
- Missing wrappers for `TrackingStateListener`, `LocationListener`, `TrackingEventsReceiver`, `SpeedViolationsListener`, `TagsProcessingListener`, or `TagsProcessingReceiver`.
- Unnecessary `TelematicsDataSource` wrapper that only forwards to `TrackingApi` without adding another source or meaningful boundary.
- `TelematicsService`, `TelematicsTagsService`, or `TelematicsTripsService` introduced as the default app-facing API instead of repository naming.
- Missing flow-specific stop methods for supported tracking start flows.
- `setTripRecordMode(...)` implemented as a large business branch inside `TelematicsRepository` instead of use cases.
- `LogoutUseCase` calls `TelematicsRepository.logout()` without first switching trip recording mode to `TripRecordMode.DISABLED` and `isActive = false`.
- Use cases generated even though the user chose repository-only integration and the host app has no use-case/domain layer.
- Trip recording mode `(mode, isActive)` not persisted in the host app's preferences/settings layer.
- SDK trip/tag/statistics/share methods left unwrapped when the user asked for full SDK integration.
- High-level operating modes implemented directly in UI while the host app has a domain/use-case layer.
- SDK initialized from an `Activity` instead of `Application.onCreate()` or app startup.
- Runtime permissions requested after enabling the SDK.
- Manual tracking started before required future tag completion.
- `logout()` used when the app only meant to temporarily disable SDK collection.
- App-controlled persistent mode not reset to `TrackingMode.Standard`.
- One-time persistent flow implemented with deprecated `startPersistentTracking()` instead of `startTrackAsPersistent()`.
- Blocking trip/statistics APIs called on the main thread.
- User-visible strings hardcoded in generated Kotlin instead of read from the host app's `Settings`/localization layer.
