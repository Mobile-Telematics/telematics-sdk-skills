# Telematics SDK Skills

Agent skills for integrating Damoov TelematicsSDK into mobile applications with Claude Code, OpenAI Codex, and other AI coding agents.

Use these skills when you want an agent to make SDK integration decisions from verified TelematicsSDK patterns instead of generic mobile assumptions.

## Available Skills

| Skill | Use When |
| --- | --- |
| [iOS Telematics SDK Integration](ios-telematics-sdk-integration-skill/) | Integrating, migrating, reviewing, or debugging Damoov TelematicsSDK in iOS apps. |

## Install All Skills

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

## Update All Skills

Refresh every skill from this repository by running the install command again:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --agent claude
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills --agent codex
```

## Manual Install

From an existing local clone of this repository:

```bash
npx ai-agent-skills install . --agent claude
npx ai-agent-skills install . --agent codex
```

## Skill Details

### iOS Telematics SDK Integration

The iOS skill helps an agent with:

- Swift Package Manager setup with exact TelematicsSDK versions.
- Service/facade architecture around `RPEntry`, `TelematicsAPIService`, and `TelematicsTagsService`.
- `AppDelegate` and `SceneDelegate` lifecycle forwarding.
- Swift `async`/`await` wrappers over SDK callback APIs.
- Automatic tracking, standard manual tracking, and persistent manual tracking flows with and without tags.
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

Or install from the skill path:

```bash
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills/ios-telematics-sdk-integration-skill --agent claude
npx ai-agent-skills install Mobile-Telematics/telematics-sdk-skills/ios-telematics-sdk-integration-skill --agent codex
```

Update only this skill:

```bash
npx ai-agent-skills sync ios-telematics-sdk-integration-skill --agent claude
npx ai-agent-skills sync ios-telematics-sdk-integration-skill --agent codex
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
Use $ios-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this iOS app. Primary flow should be persistent manual tracking without tags.
```

```text
Use $ios-telematics-sdk-integration-skill to integrate Damoov TelematicsSDK into this iOS app. Primary flow should be persistent manual tracking with tags.
```

```text
Use $ios-telematics-sdk-integration-skill to migrate this app away from deprecated RPEntry APIs.
```

```text
Use $ios-telematics-sdk-integration-skill to review the existing TelematicsSDK integration and identify lifecycle, permissions, tracking, and tag issues.
```
