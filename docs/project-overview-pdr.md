# Project Overview & Product Development Requirements

**Vocabulary Builder Chrome Extension v1.0.5**

---

## Vision & Purpose

Empower learners to acquire vocabulary naturally while browsing the web through integrated word lookup, contextual learning, and spaced-repetition flashcards. Support multi-language translation with optional AI enhancement for rich, contextual definitions.

---

## Target Users

1. **Language learners** (ESL/EFL students, polyglots) studying via reading online content
2. **Academic researchers** cross-referencing terminology across languages
3. **Content creators** building vocabulary in foreign languages for work/travel
4. **Self-directed learners** seeking low-friction study tools integrated into daily browsing

---

## Feature List

### Core Features (v1.0.5 Shipped)

| Feature | Scope | Status |
|---------|-------|--------|
| **Word Lookup** | Right-click any word → Free Dictionary API lookup with phonetic / part-of-speech / definitions | Live |
| **Multi-Language Translation** | 12 target languages: VI, ZH, JA, KO, ES, FR, DE, PT, RU, TH, ID, AR | Live |
| **AI Translation** | Optional high-quality translation via 6 LLM providers (OpenAI, Gemini, Grok, OpenRouter, Groq, Mistral) | Live |
| **Side Panel** | PDF lookup results in Chrome 114+ side panel | Live |
| **Flashcards & SM-2** | Study with spaced repetition (quality 1–5 → next interval) | Live |
| **Gamification** | Streaks, XP, levels, daily-goal progress tracking | Live |
| **Audio Pronunciation** | Google Text-to-Speech for word audio playback | Live |
| **Keyboard Shortcuts** | Configurable hotkey mode (alternative to mouseup selection) | Live |
| **Study Reminders** | Configurable notification intervals via Chrome alarms | Live |
| **Offline-First** | All data in `chrome.storage.local`; no cloud sync required | Live |
| **Highlight & Persist** | Right-click → highlight; persisted by page URL + XPath | Live |
| **Dark Mode** | Theme toggle in settings & UI (Tailwind-based) | Live |

---

## Non-Functional Requirements

| Requirement | Description | Status |
|-------------|-------------|--------|
| **Chrome MV3** | Manifest V3 only; no background page, only service worker | Live |
| **Performance** | Tooltip appears <200ms; highlight restoration <500ms on page load | Implemented |
| **Storage** | Supports 10,000+ words in `chrome.storage.local` (10 MB soft limit) | Designed |
| **Privacy** | No cloud sync by default; data stays on device; API keys encrypted in storage | Live |
| **Accessibility** | WCAG 2.1 AA for popup/options UI; keyboard-navigable study mode | Partial |
| **Cross-Tab Sync** | Settings updates via `chrome.storage.onChanged` listener propagate immediately | Live |
| **Browser Compat** | Chrome 114+ (side panel); 90+ (core features) | Tested |

---

## Technical Stack

- **Frontend:** React 18 + TypeScript 5.6, Zustand (state), Tailwind CSS + Heroicons
- **Build:** Vite 5 + CRXJS 2.0-beta, code obfuscation for release
- **Chrome APIs:** `runtime`, `storage.local/session`, `contextMenus`, `sidePanel`, `notifications`, `alarms`, `tabs`, `action`
- **Testing:** Vitest + jsdom (unit), Playwright + Chrome Canary (e2e)
- **Code Quality:** ESLint 9, TypeScript strict, Husky + commitlint (conventional commits)
- **Tooling:** Sharp (icon conversion), rollup-plugin-obfuscator

---

## Functional Requirements Detail

### FR-1: Word Lookup
- User right-clicks word on any webpage → context menu "Look up / Translate" → tooltip appears near cursor
- Lookup via Free Dictionary API (no auth required)
- Display: word, pronunciation, part-of-speech, definitions, usage examples
- Optional: selected phrase/multi-word context passed to backend

### FR-2: Translation (Free & AI)
- Free path: MyMemory API for fast, no-cost translation
- AI path: User configures LLM provider (OpenAI, Gemini, etc.) + API key → higher-quality synonyms/antonyms/context
- `translateText` service routes based on `useLLMTranslation` toggle + provider config
- Output: translated text + extracted synonyms/antonyms/example notes

### FR-3: Save & Study
- "Save to Vocabulary" button → word + translation persisted to `chrome.storage.local`
- Flashcard flip shows definition; click rating (1–5) → SM-2 algo computes next review date
- Study view navigates word-by-word, tracks progress per session

### FR-4: Sidebar & PDF Lookup
- `LOOKUP_WORD` message routed to side panel (Chrome 114+)
- Display recent lookups in history list; re-translate with different lang/provider
- Empty state if no lookups yet

### FR-5: Reminders & Gamification
- Daily notification at user-specified time (via `chrome.alarms`)
- XP, streak, level tracking tied to review sessions
- Dashboard shows stats + quick motivational message

### FR-6: Keyboard Shortcuts
- Alt+M / Cmd+Shift+M (configurable) as alternative to mouseup selection
- Shortcuts recorded via key-combo capture UI in options

### FR-7: Highlight & Persistence
- Right-click → "Highlight" wraps selection in `<span class="vocab-text-highlight">`
- Stored per page (normalized URL) + XPath + offset
- On reload: restore highlights via XPath resolution + reapply styles

---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **User Retention** | >40% DAU after 2 weeks | Analytics / user feedback |
| **Words Saved per User** | Average 50+ words/month | Chrome storage aggregates |
| **Study Sessions/User** | Average 2+ sessions/week | Notification engagement |
| **AI Translation Adoption** | >25% users configure LLM keys | Settings telemetry |
| **Zero Critical Bugs** | v1.0.5 release stable | Test suite pass rate 100% |
| **Load Time** | Tooltip <200ms, page load <500ms | Playwright perf benchmarks |

---

## Scope Boundaries

### In Scope
- Single extension per user (no multi-profile sync within extension)
- Offline-first: no mandatory cloud backend
- Translation via 6 LLM + 1 free provider
- Support Chrome 114+ for full feature set

### Out of Scope (v1.0.5)
- Cloud sync / cross-device vocabulary sharing
- Voice input for lookups
- Mobile app (browser extension only)
- Advanced NLP (sentiment, POS tagging beyond API)
- Collaborative study groups

---

## Acceptance Criteria (v1.0.5)

- [x] All core features shipped and documented
- [x] Test suite: >80% coverage on critical paths (translation, SM-2, store)
- [x] Popup/options/sidepanel render without errors
- [x] Tooltip appears within 200ms of right-click
- [x] Highlights persist across page reloads
- [x] Settings sync across tabs via `storage.onChanged`
- [x] No sensitive data (API keys) logged to console
- [x] ESLint + TypeScript strict pass with 0 errors
- [x] E2E tests pass on Chrome Canary

---

## Open Questions

1. **Firebase dependency** — Listed in `package.json` v10.14.1 but no imports found. Is this dead code, a planned feature (cloud sync), or conditionally imported?
2. **release-manifest.json duplication** — Files exist at both project root and `vocabulary-extension/`; which drives the release process?
3. **Design tokens mapping** — `src/styles/design-tokens.css` indexed but not opened; token contract / usage patterns unknown.
4. **vitest-chrome mocking extent** — `vitest.setup.ts` not scanned; depth of chrome API mock surface unknown.

---

## Version History

- **v1.0.5** (current) — Stable release with all core features, multi-language + AI translation, SM-2, gamification
- **v1.0.0** — Initial public release
