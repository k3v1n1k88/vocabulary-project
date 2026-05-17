# Scout Report — Codebase (Vocabulary Builder Chrome Extension)

**Date:** 2026-05-08
**Method:** External (Gemini 3 Flash Preview, 3 parallel agents)
**Scope:** `vocabulary-extension/src/` — Chrome MV3 extension (React 18 + TS + Vite + CRXJS + Zustand)

## Architecture Overview

```
src/
├── manifest.ts              MV3 config (defineManifest)
├── background/              Service worker + 6 message handlers
├── content/                 Content script + tooltip/floating-menu/highlight modules
├── popup/                   Dashboard / Study / Vocabulary tabs (React)
├── options/                 Settings page (React + hooks)
├── sidepanel/               PDF lookup side panel (React)
├── shared/                  Stores, APIs, translation, SM-2, components
├── styles/                  design-tokens.css
└── types/                   Single types/index.ts SoT
```

## Relevant Files — Background Layer

- `src/manifest.ts` — MV3 manifest (`defineManifest`); pulls metadata from `package.json`
- `src/background/service-worker.ts` — Message router, context-menu init, side-panel/notification orchestration; uses `runtime`, `contextMenus`, `tabs`, `sidePanel`, `action`, `notifications`
- `src/background/handlers/index.ts` — Re-exports all handlers
- `src/background/handlers/audio-handler.ts` — `handlePlayAudio` (Google TTS fetch → base64 data URL via FileReader)
- `src/background/handlers/notification-handler.ts` — `handleUpdateReminder`, `handleTestNotification` (uses `alarms`, `storage.local`)
- `src/background/handlers/options-handler.ts` — `handleOpenOptionsPage` (deep-linking via URL hash)
- `src/background/handlers/pdf-handler.ts` — `isPdfUrl`, `performPdfLookup` (uses `storage.session` since content scripts can't inject into native PDF viewer)
- `src/background/handlers/translation-handler.ts` — `handleTranslateText`, `handleTranslateSwap`
- `src/background/handlers/word-handler.ts` — `handleLookupWord`, `handleSaveWord` (persists SM-2 initial state to `storage.local`)
- `src/types/index.ts` — `Word`, `FlashcardData`, `UserStats`, `LLMProvider`, `UserSettings`, `MessageType`, `Message`, `PdfLookupResult`

## Relevant Files — Content Script Layer

- `src/content/content-script.ts` — Entry; routes `chrome.runtime.onMessage` to UI; initializes settings/keyboard/floating-menu/tooltip/highlight modules
- `src/content/modules/settings-manager.ts` — Cached settings + `storage.onChanged` sync
- `src/content/modules/tooltip-manager.ts` — `getTooltip`, `showLoadingTooltip`, `updateTooltipWithWord`, `removeTooltip`
- `src/content/modules/tooltip-positioning.ts` — Viewport-aware coords, uses saved-position from floating menu when available
- `src/content/modules/tooltip-templates.ts` — `createTooltipHTML`, `createTranslationTooltipHTML`, `createLoadingHTML`
- `src/content/modules/tooltip-error-template.ts` — Error-state markup
- `src/content/modules/tooltip-shared-elements.ts` (+ `.test.ts`) — Reusable fragments
- `src/content/modules/tooltip-event-handlers.ts` — Wires outside-click + word-tooltip events
- `src/content/modules/tooltip-button-handlers.ts` — Save / copy / audio button logic
- `src/content/modules/tooltip-dropdown-handlers.ts` — Language-swap dropdown logic
- `src/content/modules/floating-menu.ts` — `mouseup` selection menu (lookup/speak/highlight); captures `Range` and saved tooltip position
- `src/content/modules/floating-menu-template.ts` — Menu HTML
- `src/content/modules/floating-menu-lang-handlers.ts` — Language-picker handlers in menu
- `src/content/modules/highlight-renderer.ts` — Wraps text nodes in `<span class="vocab-text-highlight">` + remove button; restores on load
- `src/content/modules/highlight-storage.ts` — XPath + offset persistence keyed by normalized URL (`storage.local`)
- `src/content/modules/keyboard-shortcuts.ts` — Configurable key combos replace `mouseup` trigger
- `src/content/modules/tts-player.ts` — `HTMLAudioElement` playback + error toast
- `src/content/utils/html-escape.ts` — HTML-escape util
- `src/content/content-style.css` — Content script styles

## Relevant Files — Popup UI

- `src/popup/App.tsx`, `src/popup/main.tsx`, `src/popup/index.html`
- `src/popup/components/Header.tsx`, `TabNav.tsx`
- `src/popup/components/Dashboard.tsx` — Stats + quick actions
- `src/popup/components/StudyView.tsx` — Study session controller
- `src/popup/components/VocabularyList.tsx` — Searchable saved-words list
- `src/popup/components/study/flashcard.tsx` — Flip card
- `src/popup/components/study/rating-buttons.tsx` — SM-2 quality grid (with predicted intervals)
- `src/popup/components/study/study-progress.tsx` — `StudyProgressHeader`, `StudyNavigation`
- `src/popup/components/study/study-states.tsx` — `StudyEmptyState`, `SessionComplete`
- `src/popup/components/study/index.ts` — Barrel

## Relevant Files — Options UI

- `src/options/Options.tsx`, `src/options/main.tsx`, `src/options/index.html`
- `src/options/components/settings-content.tsx` — Section orchestrator
- `src/options/components/learning-settings.tsx` — Goals, audio, notifications, shortcuts
- `src/options/components/translation-settings.tsx` — Target lang, provider/model
- `src/options/components/ai-translation-toggle.tsx` — AI mode card
- `src/options/components/api-key-input.tsx` — Secure key UI
- `src/options/components/highlight-settings.tsx` — Color presets + custom picker
- `src/options/components/data-management.tsx` — JSON export/import
- `src/options/components/about-section.tsx` — Version + support
- `src/options/hooks/use-api-key-management.ts` — Storage + connection-test logic
- `src/options/hooks/use-shortcut-recorder.ts` — Key-combo capture
- `src/options/hooks/index.ts` — Barrel

## Relevant Files — Side Panel UI

- `src/sidepanel/SidePanel.tsx`, `src/sidepanel/main.tsx`, `src/sidepanel/index.html`
- `src/sidepanel/components/panel-header.tsx`
- `src/sidepanel/components/empty-state.tsx`, `error-state.tsx`
- `src/sidepanel/components/history-list.tsx` — Recent lookups
- `src/sidepanel/components/result-card.tsx` — Router (Word vs Translation)
- `src/sidepanel/components/word-result-card.tsx`, `translation-result-card.tsx`
- `src/sidepanel/hooks/use-sidepanel-data.ts` — Listens for `LOOKUP_WORD` / `TRANSLATE_TEXT`, manages result + history
- `src/sidepanel/hooks/use-retranslate.ts` — Re-trigger with new lang/provider

## Relevant Files — Shared Layer

- `src/shared/store.ts` — Zustand: `useVocabularyStore`, `useStatsStore`, `useSettingsStore`, `useUIStore`
- `src/shared/chrome-storage-adapter.ts` — `chromeStorage` (Zustand middleware ↔ `chrome.storage.local`)
- `src/shared/dictionary-api.ts` — `lookupWord`, `lookupWordWithTranslation` (Free Dictionary API)
- `src/shared/free-translation-api.ts` — `translateWithFreeApi` (MyMemory)
- `src/shared/translation-service.ts` (+ `.test.ts`) — `translateText`, `testConnection` (orchestrator)
- `src/shared/llm-provider-config.ts` — `LLM_PROVIDERS`, `getProviderConfig` (OpenAI / Gemini / Grok / OpenRouter / Groq / Mistral metadata)
- `src/shared/llm-request-builders.ts` — `buildTranslationPrompt`, `buildProviderRequest`
- `src/shared/translation-response-parser.ts` — `parseProviderResponse`, `parseTranslationResult` (regex extracts synonyms/antonyms/notes)
- `src/shared/translation-settings.ts` — `getApiKey`, `isLLMTranslationEnabled`
- `src/shared/spaced-repetition.ts` (+ `.test.ts`) — `calculateNextReview`, `getPredictedIntervals` (SM-2)
- `src/shared/notifications.ts` — `showDailyReminder`, `initNotifications`
- `src/shared/notification-helpers.ts` (+ `.test.ts`) — `getNotificationData`, `getRandomWordPreview`
- `src/shared/tts.ts` — `playPronunciation`
- `src/shared/components/ai-badge.tsx`, `donate-bar.tsx`, `footer-credits.tsx`, `icons.tsx`, `lang-dropdown.tsx`, `stat-item.tsx`, `toggle.tsx`, `index.ts`

## Message Contracts (Content/UI ↔ Background)

| Action | Payload | Direction |
|---|---|---|
| `LOOKUP_WORD` | `{ word, context? }` → `Word` | UI → BG |
| `LOOKUP_SELECTED` | `{ text }` | Content/Shortcut → BG |
| `TRANSLATE_TEXT` | `{ text, targetLanguage? }` → `TranslationResult` | UI/Content → BG |
| `TRANSLATE_SWAP` | `{ text, sourceLangCode, targetLangCode }` | Content → BG |
| `SAVE_WORD` | `{ word: Word }` | UI/Content → BG |
| `PLAY_AUDIO` | `{ text, lang? }` → `audioDataUrl` (base64) | Any → BG |
| `UPDATE_REMINDER` | `{ enabled, reminderInterval? }` | Options → BG |
| `OPEN_OPTIONS_PAGE` | `{ hash? }` | Any → BG |
| `TEST_NOTIFICATION` | `void` | Options → BG |
| `SHOW_LOADING` | `{ text, isPhrase }` | BG → Content |
| `SHOW_TOOLTIP` / `SHOW_TRANSLATION` | `Word` / `TranslationResult` | BG → Content |

## Core Lifecycles

**Tooltip:** `SHOW_LOADING` (BG→Content) → `tooltip-manager` creates absolute div → `tooltip-positioning` (uses saved-pos from floating menu OR Selection rect, viewport-clamped) → `tooltip-event-handlers` wires outside-click + button/dropdown delegates → `removeTooltip()` tears down + detaches `mousedown` listener.

**Floating Menu:** `mouseup` → `showFloatingButton` near cursor → captures `Range` clone + saves tooltip position → button click sends `LOOKUP_SELECTED` / `PLAY_AUDIO` or invokes `highlight-renderer` → cleanup on action / outside `mousedown` / tooltip appearance.

**Highlight render+persist:** `highlightRange` walks text nodes inside `Range`, wraps with `<span class="vocab-text-highlight">` + remove button → `saveHighlight` stores `{ xpath, offset, length, color }` to `storage.local` keyed by normalized URL → on page load `restoreHighlights` resolves XPaths via `document.evaluate` and re-applies spans.

## Translation Pipeline

`translateText` in `shared/translation-service.ts` decides path:
- **Free** (MyMemory) for single words or when `useLLMTranslation=false`
- **LLM** flow: `buildTranslationPrompt` → `buildProviderRequest` (provider-specific) → `fetch` → `parseProviderResponse` → `parseTranslationResult` (regex pulls synonyms/antonyms/notes)

Providers: OpenAI, Google Gemini, xAI Grok, OpenRouter, Groq, Mistral (per `llm-provider-config.ts`).

## SM-2 Algorithm

`calculateNextReview(quality 1–5)`: quality<3 resets repetitions+interval; quality≥3 increments repetitions, adjusts EF, computes next interval. `getPredictedIntervals` powers rating-button labels ("Hard: <1 min", etc.).

## Zustand Store Shape

Four stores persisted via `chromeStorage` adapter to `chrome.storage.local`:
- `useVocabularyStore` — `words[]`, `flashcards: Map` (serialized as entries)
- `useStatsStore` — XP / streak / level / progress
- `useSettingsStore` — listens to `chrome.storage.onChanged` for cross-context sync (options ↔ content)
- `useUIStore` — popup `activeTab` (`dashboard` | `study` | `vocabulary`)

## Test Coverage

Unit tests (Vitest + jsdom):
- `shared/translation-service.test.ts`
- `shared/spaced-repetition.test.ts`
- `shared/store.test.ts`
- `shared/notification-helpers.test.ts`
- `content/modules/tooltip-shared-elements.test.ts`

E2E: Playwright (`playwright.config.ts`, `npm run test:e2e`).

## Build / Tooling

- Vite 5 + `@crxjs/vite-plugin` 2.0-beta + `@vitejs/plugin-react`
- TypeScript 5.6, ESLint 9, Husky + commitlint (conventional)
- `rollup-plugin-obfuscator` + `javascript-obfuscator` for `build:release`
- `sharp` for icon conversion (`scripts/convert-icons.mjs`)

## Unresolved Questions

- Firebase dep listed in `package.json` but no `firebase` imports surfaced in scouted files — is it dead code, planned cloud-sync feature, or used somewhere not scanned?
- `release-manifest.json` exists at both project root and inside `vocabulary-extension/` — relationship / which one drives releases?
- `src/styles/design-tokens.css` was indexed but not opened by scouts — token contract not mapped.
- `vitest-chrome` is wired in `vitest.setup.ts` (file not opened by scouts) — extent of chrome API mocking unknown.
