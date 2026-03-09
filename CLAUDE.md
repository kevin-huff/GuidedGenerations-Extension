# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A SillyTavern third-party extension ("Guided Generations") that adds UI buttons for guided AI generation, impersonation, and persistent context injection ("guides"). It runs as a browser-side ES module loaded by SillyTavern's extension framework — there is no build step, bundler, or test runner. The SillyTavern server runs on a separate machine and cannot be started/stopped from the dev environment.

## Development

- **No build system** — plain JavaScript ES modules loaded directly by the browser
- **No test framework** — manual testing by loading the extension in SillyTavern (v1.12.9+)
- **Entry point**: `index.js` (specified in `manifest.json`)
- **Settings UI**: `settings.html` (rendered by SillyTavern's template system via `renderExtensionTemplateAsync`)
- **Styling**: `style.css`

## Architecture

### Central Import Hub (CRITICAL PATTERN)

**All internal imports must go through `scripts/persistentGuides/guideExports.js`** — never use complex relative paths or import directly from `index.js` in submodules. This file re-exports everything needed from SillyTavern core (`getContext`, `extension_settings`, `chat`, `eventSource`, etc.) and all extension modules. When adding new exports, add them to this hub.

Exception: `index.js` itself imports directly from SillyTavern paths since it's the top-level entry point.

### Module Structure

- **`index.js`** — Extension entry point. Registers buttons, event listeners, loads settings, handles auto-triggers on `MESSAGE_RECEIVED`. Exports `debugLog`, `debugWarn`, `debugError`, shared state accessors, and `defaultSettings`.

- **`scripts/guidedResponse.js`** — Guided Response: injects user instructions into context, triggers AI generation, restores original input. Handles group chat member selection.

- **`scripts/guidedSwipe.js`** — Guided Swipe: similar to Guided Response but generates a new swipe on the last AI message.

- **`scripts/guidedContinue.js`** — Guided Continue: appends AI-generated text to the last message with undo/revert support.

- **`scripts/guidedImpersonate.js`, `guidedImpersonate2nd.js`, `guidedImpersonate3rd.js`** — Expand user outlines into full messages from 1st/2nd/3rd person perspectives using `/impersonate`.

- **`scripts/persistentGuides/`** — The guide system:
  - Individual guides (`thinkingGuide.js`, `clothesGuide.js`, `stateGuide.js`, `situationalGuide.js`, `rulesGuide.js`, `customGuide.js`, `funGuide.js`, `trackerGuide.js`) — each generates context via `/gen` and injects it with a unique ID
  - `runGuide.js` — Generic runner for guide STScript commands. Handles preset switching, injection management, and `/gen` execution
  - `trackerLogic.js` — Automatic tracker execution based on chat metadata config
  - `customAutoGuide.js` — User-defined auto-triggering guide
  - `editGuides.js`, `showGuides.js`, `flushGuides.js` — CRUD for active guide injections
  - `guideExports.js` — Central import hub (see above)
  - `updateCharacter.js` — Updates character descriptions from guide context

- **`scripts/tools/`** — Utility tools: `corrections.js`, `spellchecker.js`, `editIntros.js`, `clearInput.js`, `funPopup.js`, `editIntrosPopup.js`

- **`scripts/utils/presetUtils.js`** — Profile/preset switching logic. Handles switching to a specific API profile and preset before guide execution, then restoring the original. Uses custom window events (`gg-profile-changed`, `gg-preset-changed`).

- **`scripts/settingsPanel.js`** — Loads/renders the settings HTML, populates profile/preset dropdowns, wires up event listeners.

### Key Patterns

**STScript Integration**: Guides execute via SillyTavern's STScript system (slash commands like `/gen`, `/inject`, `/listinjects`, `/flushinjects`, `/impersonate`, `/swipes-swipe`). Commands are built as pipe-delimited strings and executed through `context.executeSlashCommandsWithOptions()`.

**Preset Switching**: Guides can use different API profiles/presets. `handleSwitching()` from `presetUtils.js` returns `{ switch, restore }` functions. Always call `restore()` after guide execution to return to the user's original preset.

**Auto-Triggers**: Thinking, Clothes, State, Custom Auto, and Tracker guides can auto-trigger on `MESSAGE_RECEIVED` events, controlled by settings toggles (`autoTriggerThinking`, etc.).

**Settings Storage**: All settings live in `extension_settings[extensionName]` (SillyTavern's extension settings system). `defaultSettings` in `index.js` defines all keys with defaults. New settings must be added to `defaultSettings` and handled in `migrateProfileSettings()` if they're profile-related.

**Injection System**: Guides inject context into the chat using SillyTavern's injection system with unique IDs (e.g., `clothes`, `state`, `thinking`, `situation`, `rule_guide`, `Custom`). The `injectionEndRole` setting controls whether injections use `system`, `assistant`, or `user` role.

### Debug Logging

Use `debugLog()` and `debugWarn()` (imported from `guideExports.js`) for all informational/debug messages — never `console.log` for debug output. Only use `console.error` for actual errors users need to see. Messages are captured in a buffer accessible via `getDebugMessages()` for mobile debugging.
