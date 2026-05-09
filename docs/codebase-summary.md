# Codebase Summary

**Vocabulary Builder Chrome Extension v1.0.5**  
**Total Source:** ~11.7k LOC across 95 source files

---

## Directory Structure & Roles

```
vocabulary-extension/src/
├── manifest.ts                           MV3 manifest via @crxjs/vite-plugin
├── types/
│   └── index.ts (208 LOC)               Single source of truth: Word, FlashcardData, UserStats, 
│                                         LLMProvider, UserSettings, MessageType, Message, PdfLookupResult
│
├── background/ (354 LOC across 7 files) Service worker + 6 message handlers
│   ├── service-worker.ts                Router, context-menu init, side-panel/notification orchestration
│   └── handlers/
│       ├── audio-handler.ts             playAudio → Google TTS → base64 data URL
│       ├── notification-handler.ts      updateReminder, testNotification (alarms + storage)
│       ├── options-handler.ts           openOptionsPage (URL hash deep-linking)
│       ├── pdf-handler.ts               PDF URL detection, session storage lookup
│       ├── translation-handler.ts       translateText, translateSwap
│       ├── word-handler.ts              lookupWord, saveWord (SM-2 initial state)
│       └── index.ts                     Re-exports all handlers
│
├── content/ (3,065 LOC across 19 files) Content script + tooltip/menu/highlight modules
│   ├── content-script.ts                Entry; routes chrome.runtime.onMessage
│   ├── content-style.css                Tooltip + highlight + menu styles
│   ├── modules/
│   │   ├── settings-manager.ts          Cached settings + chrome.storage.onChanged sync
│   │   ├── tooltip-manager.ts           showLoadingTooltip, updateTooltipWithWord, removeTooltip
│   │   ├── tooltip-positioning.ts       Viewport-aware coords (uses saved-position from menu)
│   │   ├── tooltip-templates.ts         HTML for word tooltip, translation, loading states
│   │   ├── tooltip-error-template.ts    Error-state markup
│   │   ├── tooltip-shared-elements.ts   Reusable fragments + test suite
│   │   ├── tooltip-event-handlers.ts    Outside-click + word-tooltip event wiring
│   │   ├── tooltip-button-handlers.ts   Save/copy/audio button logic
│   │   ├── tooltip-dropdown-handlers.ts Language-swap dropdown
│   │   ├── floating-menu.ts             mouseup selection menu (lookup/speak/highlight)
│   │   ├── floating-menu-template.ts    Menu HTML
│   │   ├── floating-menu-lang-handlers.ts Language-picker in menu
│   │   ├── highlight-renderer.ts        Wrap text in <span class="vocab-text-highlight">
│   │   ├── highlight-storage.ts         XPath + offset persistence (storage.local)
│   │   ├── keyboard-shortcuts.ts        Configurable key combos replace mouseup
│   │   └── tts-player.ts                HTMLAudioElement playback + error toast
│   └── utils/
│       └── html-escape.ts               HTML-escape utility
│
├── popup/ (977 LOC across 10 files)     Dashboard / Study / Vocabulary tabs (React)
│   ├── App.tsx, main.tsx, index.html
│   ├── components/
│   │   ├── Header.tsx                   Logo + stats bar
│   │   ├── TabNav.tsx                   Dashboard | Study | Vocabulary tabs
│   │   ├── Dashboard.tsx                Stats + quick actions
│   │   ├── StudyView.tsx                Session controller
│   │   ├── VocabularyList.tsx           Searchable saved-words list
│   │   └── study/
│   │       ├── flashcard.tsx            Flip card with reveal toggle
│   │       ├── rating-buttons.tsx       SM-2 quality buttons (predicted intervals)
│   │       ├── study-progress.tsx       StudyProgressHeader, StudyNavigation
│   │       ├── study-states.tsx         StudyEmptyState, SessionComplete
│   │       └── index.ts                 Barrel export
│   └── popup.css                        Popup-specific styles
│
├── options/ (569 LOC across 11 files)   Settings page (React + hooks)
│   ├── Options.tsx, main.tsx, index.html
│   ├── components/
│   │   ├── settings-content.tsx         Section orchestrator (learning/translation/highlight/data/about)
│   │   ├── learning-settings.tsx        Daily goal, audio, notifications, shortcuts
│   │   ├── translation-settings.tsx     Target lang, provider/model selection
│   │   ├── ai-translation-toggle.tsx    AI mode card
│   │   ├── api-key-input.tsx            Secure key UI + connection test
│   │   ├── highlight-settings.tsx       Color presets + custom picker
│   │   ├── data-management.tsx          JSON export/import (destructive)
│   │   └── about-section.tsx            Version + support links
│   ├── hooks/
│   │   ├── use-api-key-management.ts    Storage + connection-test logic
│   │   ├── use-shortcut-recorder.ts     Key-combo capture UI
│   │   └── index.ts                     Barrel
│   └── options.css                      Options-specific styles
│
├── sidepanel/ (705 LOC across 10 files) PDF lookup side panel (Chrome 114+)
│   ├── SidePanel.tsx, main.tsx, index.html
│   ├── components/
│   │   ├── panel-header.tsx             Clear history, nav
│   │   ├── empty-state.tsx              No lookups yet
│   │   ├── error-state.tsx              Lookup failed
│   │   ├── history-list.tsx             Recent lookups with timestamps
│   │   ├── result-card.tsx              Router (Word vs Translation)
│   │   ├── word-result-card.tsx         Display Word + buttons
│   │   └── translation-result-card.tsx  Display TranslationResult + buttons
│   ├── hooks/
│   │   ├── use-sidepanel-data.ts        Listens LOOKUP_WORD / TRANSLATE_TEXT, manages history
│   │   └── use-retranslate.ts           Re-trigger with new lang/provider
│   └── sidepanel.css                    Sidepanel-specific styles
│
├── shared/ (2,664 LOC across 17 files)  Stores, APIs, translation, SM-2, components
│   ├── store.ts                         Zustand: useVocabularyStore, useStatsStore,
│   │                                     useSettingsStore, useUIStore (persisted via chromeStorage)
│   ├── chrome-storage-adapter.ts        Zustand middleware ↔ chrome.storage.local
│   ├── dictionary-api.ts                Free Dictionary API lookup (no auth)
│   ├── free-translation-api.ts          MyMemory free translation
│   ├── translation-service.ts (test)    Orchestrator: free vs LLM route decision
│   ├── llm-provider-config.ts           6 providers metadata (OpenAI, Gemini, Grok, OpenRouter, Groq, Mistral)
│   ├── llm-request-builders.ts          buildTranslationPrompt, buildProviderRequest
│   ├── translation-response-parser.ts   parseProviderResponse, parseTranslationResult (regex)
│   ├── translation-settings.ts          getApiKey, isLLMTranslationEnabled
│   ├── spaced-repetition.ts (test)      SM-2: calculateNextReview, getPredictedIntervals
│   ├── notifications.ts                 showDailyReminder, initNotifications
│   ├── notification-helpers.ts (test)   getNotificationData, getRandomWordPreview
│   ├── tts.ts                           playPronunciation helper
│   └── components/
│       ├── ai-badge.tsx                 "AI" badge indicator
│       ├── donate-bar.tsx               Donation CTA bar
│       ├── footer-credits.tsx           Credits + links
│       ├── icons.tsx                    Heroicon wrappers
│       ├── lang-dropdown.tsx            Language selector
│       ├── stat-item.tsx                Stat display (XP, streak, etc.)
│       ├── toggle.tsx                   Accessible toggle switch
│       └── index.ts                     Barrel export
│
└── styles/
    ├── design-tokens.css                Token definitions (NOT SCANNED — see open questions)
    ├── globals.css                      Tailwind directives + base resets
    └── (203 LOC across 1+ files)
```

---

## LOC Distribution

| Area | Files | LOC | Role |
|------|-------|-----|------|
| content/modules | 17 | 2,958 | Tooltip, menu, highlight, keyboard handling |
| shared | 17 | 2,664 | Stores, translation, SM-2, notification |
| content/* | 2 | 1,107 | Main content script + styles |
| options/components | 8 | 892 | Settings UI |
| sidepanel/components | 7 | 705 | PDF lookup UI |
| popup/components (tabs) | 5+5 | 644+333 | Dashboard, study, vocabulary tabs |
| background/handlers | 7 | 354 | Message handlers |
| shared/components | 8 | 336 | Reusable UI kit |
| sidepanel/hooks | 3 | 304 | Custom hooks for sidepanel |
| options/hooks | 3 | 277 | Custom hooks for options |
| types | 1 | 208 | TypeScript definitions |
| styles | 1 | 203 | Design tokens + globals |

---

## Message Contracts (Content/UI ↔ Background)

| Action | Payload | Direction | Handler |
|--------|---------|-----------|---------|
| `LOOKUP_WORD` | `{ word, context? }` → `Word` | UI → BG | word-handler |
| `LOOKUP_SELECTED` | `{ text }` | Content → BG | word-handler |
| `TRANSLATE_TEXT` | `{ text, targetLanguage? }` → `TranslationResult` | UI/Content → BG | translation-handler |
| `TRANSLATE_SWAP` | `{ text, sourceLangCode, targetLangCode }` | Content → BG | translation-handler |
| `SAVE_WORD` | `{ word: Word }` | UI/Content → BG | word-handler |
| `PLAY_AUDIO` | `{ text, lang? }` → `audioDataUrl` (base64) | Any → BG | audio-handler |
| `UPDATE_REMINDER` | `{ enabled, reminderInterval? }` | Options → BG | notification-handler |
| `OPEN_OPTIONS_PAGE` | `{ hash? }` | Any → BG | options-handler |
| `TEST_NOTIFICATION` | `void` | Options → BG | notification-handler |
| `SHOW_LOADING` | `{ text, isPhrase }` | BG → Content | tooltip-manager |
| `SHOW_TOOLTIP` / `SHOW_TRANSLATION` | `Word` / `TranslationResult` | BG → Content | tooltip-manager |

---

## Key Integration Points

- **Zustand Store** (`shared/store.ts`): Four stores (Vocabulary, Stats, Settings, UI) persisted via chrome.storage.local
- **Chrome Storage Adapter** (`shared/chrome-storage-adapter.ts`): Middleware connecting Zustand ↔ chrome.storage
- **Settings Sync** (`chrome.storage.onChanged`): Content script listens for cross-tab option changes
- **Translation Pipeline** (`shared/translation-service.ts`): Routes free vs LLM based on `useLLMTranslation` setting
- **SM-2 Entry** (`shared/spaced-repetition.ts`): Used in popup/study rating-buttons and flashcard intervals

---

## Test Coverage

| File | Type | Status |
|------|------|--------|
| `shared/translation-service.test.ts` | Unit | Vitest + jsdom |
| `shared/spaced-repetition.test.ts` | Unit | Vitest + jsdom |
| `shared/store.test.ts` | Unit | Vitest + jsdom |
| `shared/notification-helpers.test.ts` | Unit | Vitest + jsdom |
| `content/modules/tooltip-shared-elements.test.ts` | Unit | Vitest + jsdom |
| `playwright.config.ts` | E2E | Chrome Canary, post-build |

---

## Build & Tooling

- **Vite 5** + `@crxjs/vite-plugin` 2.0-beta, `@vitejs/plugin-react`
- **TypeScript 5.6** strict mode
- **ESLint 9** + commitlint (conventional commit validation)
- **Husky** pre-commit hooks (prepare, lint-staged)
- **Obfuscation** (`rollup-plugin-obfuscator` + `javascript-obfuscator` for `npm run build:release`)
- **Sharp** for icon conversion (`scripts/convert-icons.mjs`)

---

## File Naming Conventions

- **kebab-case** for all files (e.g., `tooltip-manager.ts`, `floating-menu-lang-handlers.ts`)
- **Component files** use `.tsx` extension
- **Utility/handler files** use `.ts` extension
- **Test files** use `.test.ts` or `.test.tsx` suffix
- **CSS files** use descriptive names with `-style.css` or module-specific suffix (e.g., `popup.css`)

---

## Open Questions

1. **Firebase dependency** — No imports found; verify if dead code or planned cloud-sync feature
2. **design-tokens.css** — Not scanned; token contract / usage patterns unknown
3. **vitest.setup.ts** — Chrome API mocking extent not mapped
4. **release-manifest.json** — Root vs submodule versions; clarify relationship
