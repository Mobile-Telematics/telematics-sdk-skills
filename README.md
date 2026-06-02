# Telematics SDK Skills

Agent skills for integrating Damoov TelematicsSDK into mobile applications with Claude Code, OpenAI Codex, and other AI coding agents.

Use these skills when you want an agent to make SDK integration decisions from verified TelematicsSDK patterns instead of generic mobile assumptions.

## Available Skills

| Skill | Use When |
| --- | --- |
| [Android Telematics SDK Integration](android-telematics-sdk-integration-skill/) | Integrating, migrating, reviewing, or debugging Damoov TelematicsSDK in native Android Kotlin apps. |
| [iOS Telematics SDK Integration](ios-telematics-sdk-integration-skill/) | Integrating, migrating, reviewing, or debugging Damoov TelematicsSDK in iOS apps. |
| [Flutter Telematics SDK Integration](flutter-telematics-sdk-integration-skill/) | Integrating, migrating, reviewing, or debugging the Damoov Telematics SDK Flutter plugin in Flutter apps, including Android and iOS host setup. |
| [React Native Telematics SDK Integration](react-native-telematics-sdk-integration-skill/) | Integrating, migrating, reviewing, or debugging the Damoov Telematics SDK React Native plugin in React Native apps, including Android and iOS host setup. |

## Repository Layout

Each skill keeps platform-specific reference material under `references/<platform>/`. This keeps Android, iOS, Flutter, and React Native skills consistent without mixing platform setup details.

## Install All Skills

### Using `ai-agent-skills`

Install every skill from this repository for Claude Code:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --agent claude
```

Install every skill from this repository for Codex:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --agent codex
```

Check what would be installed without changing local files:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --agent claude --dry-run
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --agent codex --dry-run
```

### Using `skills`

Install every skill from this repository for Claude Code:

```bash
npx skills add Mobile-Telematics/telematics-sdk-skills --skill '*' --agent claude-code -y
```

Install every skill from this repository for Codex:

```bash
npx skills add Mobile-Telematics/telematics-sdk-skills --skill '*' --agent codex -y
```

Install every skill for both Claude Code and Codex:

```bash
npx skills add Mobile-Telematics/telematics-sdk-skills --skill '*' --agent claude-code --agent codex -y
```

List available skills without installing:

```bash
npx skills add Mobile-Telematics/telematics-sdk-skills --list
```

## Update All Skills

Refresh every skill from this repository by running the install command again:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --agent claude
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --agent codex
```

If installed with `skills`, update installed project skills:

```bash
npx skills update -y
```

Update installed global skills:

```bash
npx skills update --global -y
```

## Manual Install

From an existing local clone of this repository:

```bash
npx ai-agent-skills install . --agent claude
npx ai-agent-skills install . --agent codex
```

Or with `skills`:

```bash
npx skills add . --skill '*' --agent claude-code -y
npx skills add . --skill '*' --agent codex -y
```

## Skill Details

### Android Telematics SDK Integration

The Android skill helps an agent with:

- Gradle/Maven repository and dependency setup using the SDK version resolved from Gradle or Maven metadata, not the changelog alone.
- Native Android Kotlin integration around `TrackingApi`, `Settings`, and `TelematicsSDKApp`.
- App-owned repository architecture with `TelematicsRepository`, `TelematicsEventsRepository`, `TelematicsTagsRepository`, and `TelematicsTripsRepository`.
- Optional use cases for telematics workflows, with a required user choice before adding them even when the app has no existing domain layer.
- Automatic tracking, standard manual tracking, app-controlled persistent manual tracking, one-time persistent manual tracking, and tag variants.
- Higher-level operating modes such as always-on, shift mode, on-demand, and disabled mode.
- SDK callbacks, listeners, receivers, speed violation events, tracking state, location updates, tag receivers, and trip APIs.
- Runtime permissions, manifest checks, foreground service behavior, and Android lifecycle setup.
- Migration away from deprecated Android SDK 3.x API.

Install only this skill for Claude Code:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --skill android-telematics-sdk-integration-skill --agent claude
```

Install only this skill for Codex:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --skill android-telematics-sdk-integration-skill --agent codex
```

Install only this skill with `skills`:

```bash
npx skills add Mobile-Telematics/telematics-sdk-skills --skill android-telematics-sdk-integration-skill --agent claude-code -y
npx skills add Mobile-Telematics/telematics-sdk-skills --skill android-telematics-sdk-integration-skill --agent codex -y
```

Or install from the skill path:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills/android-telematics-sdk-integration-skill --agent claude
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills/android-telematics-sdk-integration-skill --agent codex
```

Update only this skill:

```bash
npx ai-agent-skills sync android-telematics-sdk-integration-skill --agent claude
npx ai-agent-skills sync android-telematics-sdk-integration-skill --agent codex
npx skills update android-telematics-sdk-integration-skill -y
```

Once installed, ask your coding agent to use the skill in a native Android Kotlin app repository.

Example prompts:

```text
Use $android-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this native Android Kotlin app.
```

Primary flow prompts:

```text
Use $android-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this native Android Kotlin app. Primary flow should be automatic tracking.
```

```text
Use $android-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this native Android Kotlin app. Primary flow should be standard manual tracking without tags.
```

```text
Use $android-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this native Android Kotlin app. Primary flow should be standard manual tracking with tags.
```

```text
Use $android-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this native Android Kotlin app. Primary flow should be app-controlled persistent manual tracking without tags.
```

```text
Use $android-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this native Android Kotlin app. Primary flow should be app-controlled persistent manual tracking with tags.
```

```text
Use $android-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this native Android Kotlin app. Primary flow should be one-time persistent manual tracking without tags.
```

```text
Use $android-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this native Android Kotlin app. Primary flow should be one-time persistent manual tracking with tags.
```

```text
Use $android-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this native Android Kotlin app. Ask whether to add telematics use cases before implementing the integration.
```

```text
Use $android-telematics-sdk-integration-skill to migrate this app away from deprecated Android TelematicsSDK APIs.
```

```text
Use $android-telematics-sdk-integration-skill to review the existing TelematicsSDK integration and identify Gradle, lifecycle, permissions, tracking, callbacks, tags, and trip API issues.
```

### iOS Telematics SDK Integration

The iOS skill helps an agent with:

- Swift Package Manager setup with exact TelematicsSDK versions.
- Service/facade architecture around `RPEntry`, `TelematicsAPIService`, and `TelematicsTagsService`.
- `AppDelegate` and `SceneDelegate` lifecycle forwarding.
- Swift `async`/`await` wrappers over SDK callback APIs.
- Automatic tracking, standard manual tracking, app-controlled persistent manual tracking, and one-time persistent manual tracking flows with and without tags.
- SDK tracking modes: `.standard` and `.persistent`.
- Future tags and trip tags.
- Migration away from deprecated `RPEntry` methods and properties.

Install only this skill for Claude Code:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --skill ios-telematics-sdk-integration-skill --agent claude
```

Install only this skill for Codex:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --skill ios-telematics-sdk-integration-skill --agent codex
```

Install only this skill with `skills`:

```bash
npx skills add Mobile-Telematics/telematics-sdk-skills --skill ios-telematics-sdk-integration-skill --agent claude-code -y
npx skills add Mobile-Telematics/telematics-sdk-skills --skill ios-telematics-sdk-integration-skill --agent codex -y
```

Or install from the skill path:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills/ios-telematics-sdk-integration-skill --agent claude
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills/ios-telematics-sdk-integration-skill --agent codex
```

Update only this skill:

```bash
npx ai-agent-skills sync ios-telematics-sdk-integration-skill --agent claude
npx ai-agent-skills sync ios-telematics-sdk-integration-skill --agent codex
npx skills update ios-telematics-sdk-integration-skill -y
```

Once installed, ask your coding agent to use the skill in an iOS app repository.

Example prompts:

```text
Use $ios-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this iOS app.
```

Primary flow prompts:

```text
Use $ios-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this iOS app. Primary flow should be automatic tracking.
```

```text
Use $ios-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this iOS app. Primary flow should be standard manual tracking without tags.
```

```text
Use $ios-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this iOS app. Primary flow should be standard manual tracking with tags.
```

```text
Use $ios-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this iOS app. Primary flow should be app-controlled persistent manual tracking without tags.
```

```text
Use $ios-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this iOS app. Primary flow should be app-controlled persistent manual tracking with tags.
```

```text
Use $ios-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this iOS app. Primary flow should be one-time persistent manual tracking without tags.
```

```text
Use $ios-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this iOS app. Primary flow should be one-time persistent manual tracking with tags.
```

```text
Use $ios-telematics-sdk-integration-skill to migrate this app away from deprecated RPEntry APIs.
```

```text
Use $ios-telematics-sdk-integration-skill to review the existing TelematicsSDK integration and identify lifecycle, permissions, tracking, and tag issues.
```

### Flutter Telematics SDK Integration

The Flutter skill helps an agent with:

- `pubspec.yaml` setup for published package or public Git plugin dependencies.
- App-owned Dart service/facade design around `TrackingApi`.
- Automatic tracking, standard manual tracking, app-controlled persistent manual tracking, one-time persistent manual tracking, and future-tagged variants for every manual flow.
- Stream subscription handling for permission wizard, location, tracking state, low power mode, speed violations, and iOS callbacks.
- Android manifest, Gradle, repository, permissions, proguard, release shrink, and optional `TelematicsSDKApp` setup.
- iOS `Info.plist`, `AppDelegate`, Flutter implicit engine, SceneDelegate, background modes, and lifecycle requirements.
- Platform guards for iOS-only and Android-only plugin APIs.

Install only this skill for Claude Code:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --skill flutter-telematics-sdk-integration-skill --agent claude
```

Install only this skill for Codex:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --skill flutter-telematics-sdk-integration-skill --agent codex
```

Install only this skill with `skills`:

```bash
npx skills add Mobile-Telematics/telematics-sdk-skills --skill flutter-telematics-sdk-integration-skill --agent claude-code -y
npx skills add Mobile-Telematics/telematics-sdk-skills --skill flutter-telematics-sdk-integration-skill --agent codex -y
```

Or install from the skill path:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills/flutter-telematics-sdk-integration-skill --agent claude
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills/flutter-telematics-sdk-integration-skill --agent codex
```

Update only this skill:

```bash
npx ai-agent-skills sync flutter-telematics-sdk-integration-skill --agent claude
npx ai-agent-skills sync flutter-telematics-sdk-integration-skill --agent codex
npx skills update flutter-telematics-sdk-integration-skill -y
```

Once installed, ask your coding agent to use the skill in a Flutter app repository.

Example prompts:

```text
Use $flutter-telematics-sdk-integration-skill to integrate Damoov Telematics SDK Flutter plugin into this Flutter app.
```

Primary flow prompts:

```text
Use $flutter-telematics-sdk-integration-skill to integrate Damoov Telematics SDK Flutter plugin into this Flutter app. Primary flow should be automatic tracking.
```

```text
Use $flutter-telematics-sdk-integration-skill to integrate Damoov Telematics SDK Flutter plugin into this Flutter app. Primary flow should be standard manual tracking without tags.
```

```text
Use $flutter-telematics-sdk-integration-skill to integrate Damoov Telematics SDK Flutter plugin into this Flutter app. Primary flow should be standard manual tracking with future tags.
```

```text
Use $flutter-telematics-sdk-integration-skill to integrate Damoov Telematics SDK Flutter plugin into this Flutter app. Primary flow should be app-controlled persistent manual tracking without tags.
```

```text
Use $flutter-telematics-sdk-integration-skill to integrate Damoov Telematics SDK Flutter plugin into this Flutter app. Primary flow should be app-controlled persistent manual tracking with future tags.
```

```text
Use $flutter-telematics-sdk-integration-skill to integrate Damoov Telematics SDK Flutter plugin into this Flutter app. Primary flow should be one-time persistent manual tracking without tags.
```

```text
Use $flutter-telematics-sdk-integration-skill to integrate Damoov Telematics SDK Flutter plugin into this Flutter app. Primary flow should be one-time persistent manual tracking with future tags.
```

```text
Use $flutter-telematics-sdk-integration-skill to review the existing Flutter plugin integration and identify Android, iOS, permissions, lifecycle, tracking, and tag issues.
```

### React Native Telematics SDK Integration

The React Native skill helps an agent with:

- `package.json` setup for published package or public Git plugin dependencies.
- App-owned TypeScript service/facade design around `react-native-telematics`.
- Old/new React Native architecture behavior through TurboModule with legacy module fallback.
- Automatic tracking, standard manual tracking, app-controlled persistent manual tracking, one-time persistent manual tracking, and future-tagged variants for every manual flow.
- Listener subscription handling for location, tracking state, low power mode, speed violations, wrong accuracy, and RTLD callbacks.
- Android manifest, Gradle, repository, permissions, proguard, release shrink, and runtime permission setup.
- iOS `Podfile`, SPM linkage, `Info.plist`, `AppDelegate`, `SceneDelegate`, background modes, and lifecycle forwarding.
- Platform guards for iOS-only and Android-only plugin APIs.

Install only this skill for Claude Code:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --skill react-native-telematics-sdk-integration-skill --agent claude
```

Install only this skill for Codex:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --skill react-native-telematics-sdk-integration-skill --agent codex
```

Install only this skill with `skills`:

```bash
npx skills add Mobile-Telematics/telematics-sdk-skills --skill react-native-telematics-sdk-integration-skill --agent claude-code -y
npx skills add Mobile-Telematics/telematics-sdk-skills --skill react-native-telematics-sdk-integration-skill --agent codex -y
```

Or install from the skill path:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills/react-native-telematics-sdk-integration-skill --agent claude
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills/react-native-telematics-sdk-integration-skill --agent codex
```

Update only this skill:

```bash
npx ai-agent-skills sync react-native-telematics-sdk-integration-skill --agent claude
npx ai-agent-skills sync react-native-telematics-sdk-integration-skill --agent codex
npx skills update react-native-telematics-sdk-integration-skill -y
```

Once installed, ask your coding agent to use the skill in a React Native app repository.

Example prompts:

```text
Use $react-native-telematics-sdk-integration-skill to integrate Damoov Telematics SDK React Native plugin into this React Native app.
```

Primary flow prompts:

```text
Use $react-native-telematics-sdk-integration-skill to integrate Damoov Telematics SDK React Native plugin into this React Native app. Primary flow should be automatic tracking.
```

```text
Use $react-native-telematics-sdk-integration-skill to integrate Damoov Telematics SDK React Native plugin into this React Native app. Primary flow should be standard manual tracking without tags.
```

```text
Use $react-native-telematics-sdk-integration-skill to integrate Damoov Telematics SDK React Native plugin into this React Native app. Primary flow should be standard manual tracking with future tags.
```

```text
Use $react-native-telematics-sdk-integration-skill to integrate Damoov Telematics SDK React Native plugin into this React Native app. Primary flow should be app-controlled persistent manual tracking without tags.
```

```text
Use $react-native-telematics-sdk-integration-skill to integrate Damoov Telematics SDK React Native plugin into this React Native app. Primary flow should be app-controlled persistent manual tracking with future tags.
```

```text
Use $react-native-telematics-sdk-integration-skill to integrate Damoov Telematics SDK React Native plugin into this React Native app. Primary flow should be one-time persistent manual tracking without tags.
```

```text
Use $react-native-telematics-sdk-integration-skill to integrate Damoov Telematics SDK React Native plugin into this React Native app. Primary flow should be one-time persistent manual tracking with future tags.
```

```text
Use $react-native-telematics-sdk-integration-skill to review the existing React Native plugin integration and identify Android, iOS, permissions, lifecycle, tracking, and tag issues.
```
