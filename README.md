# Vocabulary Builder Chrome Extension

A Chrome extension for learning vocabulary with flashcards, spaced repetition (SM-2 algorithm), AI-powered translation, and context menu word lookup. Learn while you browse.

**Current version:** 1.0.5 | **Status:** Stable (shipped to Chrome Web Store)

---

## Quick Links

- **[Full Project Documentation](./docs/)** — Start here for architecture, code standards, deployment
- **[Submodule README](./vocabulary-extension/README.md)** — Feature details, tech stack, getting started
- **Architecture Map:** [system-architecture.md](./docs/system-architecture.md)
- **Code Standards:** [code-standards.md](./docs/code-standards.md)
- **Roadmap:** [project-roadmap.md](./docs/project-roadmap.md)

---

## What Is This Project?

This is the **source repository** for the Vocabulary Builder Chrome extension. The actual codebase (React 18 + TypeScript + Vite + CRXJS) lives in the `vocabulary-extension/` git submodule.

**Project structure:**
```
vocabulary-project/               (You are here)
├── vocabulary-extension/         (Git submodule - actual codebase)
│   ├── src/
│   ├── package.json
│   ├── README.md
│   └── ...
├── docs/                         (Comprehensive documentation)
│   ├── project-overview-pdr.md
│   ├── codebase-summary.md
│   ├── code-standards.md
│   ├── system-architecture.md
│   ├── deployment-guide.md
│   ├── design-guidelines.md
│   └── project-roadmap.md
├── plans/                        (Development plans & research reports)
│   └── reports/
├── .github/workflows/            (CI/CD pipelines)
└── README.md                     (This file)
```

---

## Getting Started

### 1. Clone the Repository

```bash
git clone <repo-url> vocabulary-project
cd vocabulary-project
git submodule update --init --recursive
```

### 2. Install Dependencies

```bash
cd vocabulary-extension
npm install
```

### 3. Start Development Server

```bash
npm run dev
```

Outputs a `dist/` folder. Load it as an unpacked extension in Chrome:
1. Open `chrome://extensions/`
2. Enable "Developer mode"
3. Click "Load unpacked"
4. Select the `dist` folder

### 4. View Documentation

All documentation is in `./docs/`:

| File | Purpose |
|------|---------|
| [project-overview-pdr.md](./docs/project-overview-pdr.md) | Vision, features, requirements, success metrics |
| [codebase-summary.md](./docs/codebase-summary.md) | File-by-file map of 95 source files (~11.7k LOC) |
| [code-standards.md](./docs/code-standards.md) | Naming, TypeScript, testing, commit conventions |
| [system-architecture.md](./docs/system-architecture.md) | 5 surfaces, message contracts, data flow, Zustand store |
| [deployment-guide.md](./docs/deployment-guide.md) | Build, test, release, Chrome Web Store submission |
| [design-guidelines.md](./docs/design-guidelines.md) | Design system, components, colors, typography, A11y |
| [project-roadmap.md](./docs/project-roadmap.md) | Phase roadmap, backlog, success metrics, timeline |

---

## Features (v1.0.5)

| Feature | Status | Details |
|---------|--------|---------|
| **Word Lookup** | Live | Right-click any word → Free Dictionary API |
| **Multi-Language Translation** | Live | 12 languages (VI, ZH, JA, KO, ES, FR, DE, PT, RU, TH, ID, AR) |
| **AI Translation** | Live | 6 LLM providers: OpenAI, Gemini, Grok, OpenRouter, Groq, Mistral |
| **Flashcards & SM-2** | Live | Spaced repetition algorithm (quality 1–5 ratings) |
| **Gamification** | Live | XP, streaks, levels, daily goals, progress tracking |
| **Audio Pronunciation** | Live | Google TTS for word pronunciation |
| **Keyboard Shortcuts** | Live | Configurable hotkeys (Alt+M / Cmd+Shift+M) |
| **PDF Side Panel** | Live | Chrome 114+ side panel for PDF lookups |
| **Daily Reminders** | Live | Configurable notifications via chrome.alarms |
| **Highlight & Persist** | Live | Right-click → highlight saved per page URL |
| **Dark Mode** | Live | System preference + user toggle |
| **Data Export/Import** | Live | JSON backup & restore |
| **Cross-Device Settings Sync** | Live | Settings + API keys sync via chrome.storage.sync; vocabulary/stats remain local |

---

## Architecture Snapshot

```
┌─────────────────────────────────────────────────┐
│         React UIs (4 Surfaces)                  │
│  Popup | Options | SidePanel | Content Script  │
└────────────────┬────────────────────────────────┘
                 │
                 ├─ chrome.runtime.sendMessage
                 │
        ┌────────▼─────────────┐
        │  Service Worker      │
        │  (Message Router)    │
        └────────┬─────────────┘
                 │
        ┌────────▼──────────────────┐
        │ Handlers (6 types)        │
        │ word, translation, audio, │
        │ notification, options, pdf│
        └────────┬──────────────────┘
                 │
        ┌────────▼──────────────────┐
        │ Services & APIs           │
        │ - Translation pipeline    │
        │ - SM-2 algorithm          │
        │ - Dictionary API          │
        │ - LLM providers           │
        │ - Chrome storage          │
        └────────┬──────────────────┘
                 │
        ┌────────▼──────────────────┐
        │ Zustand Stores (4)        │
        │ Vocabulary, Stats,        │
        │ Settings, UI State        │
        │ (Persisted to chrome.store)
        └───────────────────────────┘
```

---

## Key Directories

### Source Code (`vocabulary-extension/src/`)

- **`background/`** — Service worker + 6 message handlers (word, translation, audio, notification, options, PDF)
- **`content/`** — Content script + tooltip, menu, highlight modules (19 files)
- **`popup/`** — React UI: Dashboard, Study, Vocabulary tabs
- **`options/`** — React UI: Settings page (learning, translation, data, about)
- **`sidepanel/`** — React UI: PDF lookup side panel
- **`shared/`** — Zustand stores, translation service, SM-2 algorithm, reusable components
- **`types/`** — TypeScript definitions (single source of truth)
- **`styles/`** — Tailwind + design tokens

### Documentation (`docs/`)

High-level guides for architecture, standards, and deployment.

### Development Plans (`plans/`)

Research reports and implementation plans from scout agents.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **UI** | React 18 + TypeScript 5.6 |
| **Build** | Vite 5 + CRXJS 2.0-beta |
| **State** | Zustand 4.5.5 (chrome.storage persistence) |
| **Styling** | Tailwind CSS + Heroicons |
| **Testing** | Vitest + jsdom (unit), Playwright (e2e) |
| **Quality** | ESLint 9, TypeScript strict, Husky (pre-commit) |
| **Chrome APIs** | storage, runtime, contextMenus, sidePanel, notifications, alarms, tabs |

---

## Development Workflow

```bash
# Install
cd vocabulary-extension && npm install

# Develop (watch mode)
npm run dev

# Type check
npx tsc --noEmit

# Lint
npm run lint

# Test
npm run test              # Unit tests (watch)
npm run test:unit         # Unit tests (run once)
npm run test:coverage     # Coverage report
npm run test:e2e          # E2E tests (requires build)

# Build
npm run build             # Dev build
npm run build:release     # Obfuscated release build

# Version management
npm run version:patch     # 1.0.5 → 1.0.6
npm run version:minor     # 1.0.5 → 1.1.0
npm run version:major     # 1.0.5 → 2.0.0
```

---

## Committing Code

Uses **Conventional Commits** (enforced via commitlint):

```bash
git commit -m "feat(translation): add Grok provider integration"
git commit -m "fix(tooltip): correct viewport positioning for RTL text"
git commit -m "test(sm2): increase coverage to 85%"
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`

---

## Testing & Quality Assurance

### Run All Tests

```bash
npm run test:unit       # Unit tests (~5 files)
npm run test:coverage   # Coverage report (target: >80%)
npm run test:e2e        # E2E tests (Playwright)
npm run lint            # ESLint + type check
```

### Test Coverage Goals

- **Translation service:** >90%
- **SM-2 algorithm:** >90%
- **Zustand store:** >80%
- **Notification helpers:** >85%
- **Overall:** >80%

### CI/CD Pipelines

- **test.yml** — Runs on every PR / push (lint, type check, unit tests, E2E)
- **release.yml** — Runs on version tags (builds release, publishes artifact)

---

## Performance Targets

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Tooltip latency | <200ms | DevTools Performance tab, right-click word |
| Highlight restore | <500ms | DevTools Performance, page load |
| Settings sync | <100ms | Two windows, change setting, observe |
| Bundle size | <2 MB | `du -sh dist/` |
| Storage usage | <10 MB for 1000+ words | `chrome://extensions/ → Storage` |

---

## Debugging

### Popup / Options UI
1. Click extension icon → right-click → Inspect popup
2. DevTools opens; use Console, Network, Performance tabs

### Service Worker
1. `chrome://extensions/`
2. Find extension → Inspect views → service worker
3. DevTools opens; set breakpoints

### Content Script
1. Right-click webpage → Inspect
2. Switch "Scope" dropdown to content script
3. Type commands; interact with DOM

---

## Deployment

### To Chrome Web Store

1. Bump version: `npm run version:patch`
2. Build release: `npm run build:release`
3. Create git commit & tag
4. Push (triggers GitHub Actions)
5. Download artifact from release
6. Submit ZIP to Chrome Web Store Developer Console (24–72 hour review)

See [deployment-guide.md](./docs/deployment-guide.md) for detailed steps.

---

## Open Questions & TODOs

From scout report & documentation audit:

1. **Firebase dependency** — Is it dead code or planned cloud-sync feature? ([project-roadmap.md](./docs/project-roadmap.md#blocker-firebase-dependency-clarification))
2. **release-manifest.json** — Root vs submodule versions; clarify single source of truth
3. **design-tokens.css** — Verify token contract, Tailwind integration
4. **vitest-chrome mocking** — Full extent of chrome API stubs unknown
5. **A11y compliance** — WCAG 2.1 AA full audit + fixes needed

See [project-roadmap.md](./docs/project-roadmap.md) for backlog & priorities.

---

## Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/your-feature`)
3. Commit changes (`git commit -m 'feat: your feature'`)
4. Push to branch (`git push origin feature/your-feature`)
5. Open Pull Request

**Requirements:**
- Tests pass (`npm run test`)
- Lint passes (`npm run lint`)
- Conventional commit message
- Describe "why" in PR body, not "what" (code shows what)

---

## Support & Community

- **Issues:** GitHub Issues (bugs, feature requests)
- **Discussions:** GitHub Discussions (questions, ideas)
- **Email:** vanntl@vng.com.vn

---

## License

MIT License — See LICENSE file for details

---

## Project Status

**v1.0.5 is stable and shipped to Chrome Web Store.** Actively maintained. See [project-roadmap.md](./docs/project-roadmap.md) for Phase 2 plans (cloud sync, advanced features).

**Last updated:** 2026-05-08
