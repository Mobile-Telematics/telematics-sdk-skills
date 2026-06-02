---
name: android-telematics-sdk-integration-skill
description: Use when designing, integrating, migrating, reviewing, or debugging Damoov TelematicsSDK for native Android Kotlin apps, especially Gradle/Maven setup, Application initialization, TrackingApi lifecycle, Settings builder configuration, Android runtime permissions and merged manifest checks, automatic/manual tracking flows, standard/persistent SDK tracking modes, one-time persistent manual tracking, future tags, trip APIs, receivers/listeners, and replacing deprecated Android SDK 3.x API using the SDK version resolved by Gradle.
---

# Android Telematics SDK Integration

## Operating Rule

Treat published Damoov documentation as a baseline, not as the only source of truth for the app's resolved SDK version. Before changing app code, verify the installed SDK API from Gradle dependency artifacts, downloaded sources if available, generated stubs/IDE symbol resolution, or the app lock/build files.

Do not assume access to Damoov SDK source repositories. This skill is for public integrations where the SDK is available only as a downloaded dependency.
Do not ask the user for a local SDK source checkout and do not include steps that depend on one.

## Workflow

1. Inspect the target Android app first:
   - Gradle shape: root `settings.gradle(.kts)`, app `build.gradle(.kts)`, version catalogs, dependency repositories, product flavors, min/target/compile SDK.
   - Entry points: `Application`, launch `Activity`, DI startup, WorkManager/custom initializers.
   - Manifest: app manifest and merged manifest for SDK permissions, services, receivers, foreground service type, provider authorities, and any `tools:node` or `maxSdkVersion` overrides.
   - Permissions: runtime request flow for location, background location, activity recognition, notifications, battery optimization, and exact alarm policy.
   - Existing SDK usage: `rg -n "TrackingApi|Settings\\(|setDeviceID|setDeviceToken|setEnableSdk|startTracking|startPersistentTracking|startTrackAsPersistent|TrackingMode|addFutureTrackTag|registerTrackingEventsReceiver|TagsProcessingReceiver|TrackingStateListener"`.

2. Use Maven/Gradle dependency integration for host apps. Load `references/android/common-sdk-surface.md` before editing dependency setup.

3. Verify the exact SDK version before editing dependencies:
   - Check current app dependency declarations.
   - Check the version resolved by Gradle dependency reports, lockfiles, version catalogs, or app build files.
   - Do not choose a dependency version from the public changelog alone. The changelog is release notes, not a Maven artifact index, and may lag behind or diverge from install docs.
   - If adding the SDK to an app that has no existing dependency and the user did not specify a version, ask which SDK version to use or verify available Maven versions through Gradle/Maven metadata before editing the build.

4. Inspect the resolved SDK API before relying on public APIs. Prefer downloaded sources jars when Gradle provides them; otherwise inspect AAR/classes with IDE symbol resolution, `javap`, Kotlin metadata-aware tooling, or generated API docs. Use the reference files as guidance, not proof that every method exists in the app's resolved version.

5. Prefer app-owned repositories around `TrackingApi.getInstance()` instead of calling the SDK directly from activities, fragments, Composables, or view models. Treat `TrackingApi` as the immediate external data source; do not add a separate SDK DataSource layer unless the host app already uses that pattern for third-party SDKs. Load `references/android/architecture.md` before designing or changing integration structure.

6. Before implementing a new `TelematicsRepository`, ask the user which primary tracking flow should be placed first in the repository and related optional use cases:
   - automatic tracking
   - standard manual tracking without tags
   - standard manual tracking with tags
   - app-controlled persistent manual tracking without tags
   - app-controlled persistent manual tracking with tags
   - one-time persistent manual tracking without tags
   - one-time persistent manual tracking with tags
   If the user has already stated the primary flow in the request or existing product code makes it unambiguous, use that flow without asking again.

7. Replace deprecated public API with current API. Load `references/android/api-migration.md` when touching existing SDK usage, especially apps migrating from SDK 3.x or older docs.

8. For public SDK APIs common to every integration, load `references/android/common-sdk-surface.md`. This includes Android project setup, `Application` initialization, `Settings`, `TrackingApi` status/config/permission methods, listeners, receivers, trip APIs, and tags.

9. For tracking flow sequences, SDK tracking modes, persistent tracking, and tags, load `references/android/integration-reference.md`.

10. Implement repositories for all SDK API groups, not only tracking flows:
    - `TelematicsRepository` for initialization/configuration helpers, SDK enablement, low-level tracking flows, status, diagnostics, notification intent, accident detection, passive detection, RTD access when requested, and heartbeats.
    - `TelematicsEventsRepository` for public SDK callbacks/listeners/receivers: tracking state callback, location listener, tracking events receiver, and speed-violation controls/callbacks when requested.
    - `TelematicsTagsRepository` for future tags, processed-trip tags, tag callbacks, and tag receivers.
    - `TelematicsTripsRepository` for track list/details, unsent trips, upload, origin dictionary/change, statistics/dashboard, share/unshare, and shared-track details.
    Keep `TrackingApi` as the immediate external data source. Do not create extra SDK DataSource wrappers by default.

11. Before choosing repository-only integration or generating use cases for a new/full telematics integration, ask whether the user wants use cases generated/integrated. This is a required architecture decision for every host app, including single-module apps and apps with no current domain layer. Do not silently skip the question just because the app can work with repositories only, and do not silently add use cases just because the app already has a domain/use-case layer. Existing use-case/domain classes only determine the recommended answer and implementation style. If no domain/use-case layer exists and the user chooses use cases, create a minimal app-appropriate use-case/workflow layer instead of falling back to repository-only. Skip this question only when the user has already answered it in the current request or the current task is a narrow fix/review that does not add or redesign telematics workflows. Ask:

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

    If the app already has a domain/use-case layer, mention that use cases are recommended for this integration, then wait for the user's choice before implementing them. If the app has no domain/use-case layer, still ask; explain that choosing use cases will create a small telematics workflow/use-case package that fits the app's modular or single-module structure. If use cases are requested, implement high-level operating modes from `android_tracking_modes.md` as domain/application use cases:
    - `EnableAutomaticModeUseCase`
    - `EnableDisabledModeUseCase`
    - `PrepareOnDemandModeUseCase`
    - `StartOnDemandTripUseCase`
    - `StopOnDemandTripUseCase`
    - `SignOnShiftUseCase`
    - `SignOffShiftUseCase`
    - `LogoutUseCase`
    These use cases orchestrate `TelematicsRepository`; do not put product-mode orchestration only in UI or only as repository methods unless the user chose repository-only integration. If the app has a trip-recording mode selector and use cases are requested, implement `SetTripRecordModeUseCase` and `GetTripRecordModeUseCase` over a persisted `TripRecordMode` state. Do not keep `setTripRecordMode(...)` business branching inside `TelematicsRepository` when use cases are part of the integration. If the user chooses repository-only integration, note the tradeoff briefly and keep the generated repository API sufficient for the app to add use cases later.

12. Implement repositories with all supported flows and public SDK method groups. In `TelematicsRepository`, order tracking methods so the user's primary flow appears first. Add a small preferences/local settings abstraction for telematics mode state when needed, using the host app's existing DataStore, SharedPreferences, database, or settings repository. Keep changes outside the integration surface minimal. Do not add dependencies beyond the SDK and existing app stack unless the user explicitly approves.

13. Validate with the narrowest available check:
    - Prefer Gradle compile for the affected module, e.g. `./gradlew :app:compileDebugKotlin` or the app's actual variant.
    - If compile is too expensive or unavailable, run focused static checks with `rg` for deprecated symbols and explain the limit.

## Coding Rules

- Do not hardcode user-visible text strings in generated app code. Store text in the host app's `Settings` mechanism or existing localization/configuration layer and read it from there.
- Initialize `TrackingApi.getInstance()` once from the real `Application.onCreate()` with an application context.
- Use `Settings()` builder functions for SDK 4.x: `accuracy(...)`, `stopTrackingTimeout(...)`, `autoStartOn(...)`, `adOn(...)`, and `passiveDetectionOn(...)`.
- Do not use old `Settings(...)` constructors with HF, ELM, or legacy boolean arguments.
- Use `setDeviceID(deviceId)` / `getDeviceId()` / `clearDeviceID()` / `logout()` instead of `setDeviceToken`, `virtualDeviceToken`, or app-side token aliases in SDK calls.
- Treat the device ID as a Damoov-issued user identifier, normally GUID-formatted. Do not generate it locally unless the product backend explicitly proxies or delegates the Damoov platform value to the app.
- Do not add API-key or credentials setup to Android app code; the SDK initialization APIs used by this skill do not take app-provided credentials.
- Set a valid device ID before enabling the SDK or starting tracking. Keep device identity separate from tracking flows: generated repositories should expose `setDeviceId(...)` for login/session binding, and start/enable tracking methods should not accept or reset the device ID.
- Check `isAllRequiredPermissionsAndSensorsGranted()` before enabling the SDK. Do not rely on compile-time manifest declarations as runtime permission proof.
- Do not remove or cap SDK manifest permissions without verifying merged manifest behavior and runtime requirements.
- Use `setEnableSdk(true)` / `setEnableSdk(false)`. Do not use removed `setEnableSdk(enable, withCheckingPermissions)` overloads.
- Expose `enableSdk()` and `disableSdk()` as separate repository methods. `disableSdk()` must only disable collection with `setEnableSdk(false)`; do not add identity-clearing parameters and do not call `logout()` from it.
- Expose `logout()` as a separate repository method for account/device identity removal. Use it only when the product explicitly wants SDK logout semantics.
- When use cases are integrated, implement `LogoutUseCase` so it first switches the app trip-recording mode to `TripRecordMode.DISABLED` with `isActive = false`, then calls `TelematicsRepository.logout()`.
- In repository methods such as `setDeviceId`, `enableAutomaticTracking`, `startStandardManualTracking`, persistent manual starts, and one-time persistent starts, call SDK mutators only when the current SDK state differs: compare `getDeviceId()` before `setDeviceID(deviceId)`, `isSdkEnabled()` before `setEnableSdk(true/false)`, `getTrackingMode()` before `setTrackingMode(...)`, and `getMaxPersistentTrackingInterval()` before `setMaxPersistentTrackingInterval(minutes)`.
- Treat automatic/manual tracking as app-level flows; treat `TrackingMode.Standard`/`TrackingMode.Persistent` as SDK tracking modes configured with `setTrackingMode(...)`.
- For standard manual tracking, use `setTrackingMode(TrackingMode.Standard)` plus `startTracking()`.
- For app-controlled persistent SDK mode, use `setTrackingMode(TrackingMode.Persistent)` plus `startTracking()` and switch back with `setTrackingMode(TrackingMode.Standard)` when business logic requires it.
- For one-time persistent manual tracking, use `startTrackAsPersistent()`. Do not call deprecated `startPersistentTracking()`.
- Configure persistent intervals with `setMaxPersistentTrackingInterval(minutes)` only in the valid `5..600` minute range; default is `240` minutes.
- Prefer explicit stop methods for each supported flow instead of one hidden-state-driven stop method. If a repository still exposes one shared manual stop helper internally, only app-controlled persistent flows may restore `TrackingMode.Standard`.
- Expose a relevant stop method for every supported flow: automatic disables SDK collection, standard manual stops tracking and disables SDK collection, tagged manual flows remove future tags before stopping when cleanup is required, app-controlled persistent flows also restore `TrackingMode.Standard`, and one-time persistent flows must not manually restore `TrackingMode.Standard`.
- Persist app-level trip recording mode state outside `TelematicsRepository`, for example as `(TripRecordMode, isActive)` in the host app's preferences/settings layer. Use cases must update this state after successful mode transitions and read it before deciding whether to enable, disable, stop, or start tracking.
- For cross-platform manual tagged flows, treat "with tags" as future tags attached before the upcoming manually started trip.
- Future tag `tag` and `source` values are product-defined strings. Do not invent SDK-side enums or hardcoded user-visible labels for them; validate allowed business values in the host app/backend when needed.
- If a future tag is required for a manually started trip, add the tag and wait for the tag processing callback/receiver status before starting tracking; otherwise document the race.
- Do not collapse future-tag result statuses to `Unit` when sequencing depends on them. Preserve or map SDK statuses such as success, offline/backend failure, invalid device ID, or generic tag operation failure into the app's result/error model.
- Remove future tags before disabling the SDK when cleanup depends on SDK/API availability.
- Do not silently ignore `Boolean` return values from `startTracking()`, `startTrackAsPersistent()`, `stopTracking()`, or registration APIs.
- Do not call blocking trip/statistics APIs such as `getTracks(...)` or `getTrackDetails(...)` on the main thread.
- Generated public facade methods must include concise English KDoc comments.
- Delegate/listener/receiver examples generated by the skill should only log or forward events into the facade. Do not add business logic, analytics, state mutation, or UI updates in SDK callbacks unless the user explicitly asks for it.
- Do not register speed violation monitoring unless the user explicitly requests speed-limit behavior, because it requires product-specific threshold and timeout values.

## Official Docs

Use these docs for baseline integration behavior, but check the resolved dependency version before copying code:

- https://docs.damoov.com/docs/-download-the-sdk-and-install-it-in-your-environment
- https://docs.damoov.com/docs/methods-for-android-app
- https://docs.damoov.com/docs/tracking
- https://docs.damoov.com/docs/android-sdk-incoming-tags-managing
- https://docs.damoov.com/changelog
