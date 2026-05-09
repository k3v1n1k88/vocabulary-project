# Code Standards & Conventions

**Vocabulary Builder Chrome Extension**

---

## Guiding Principles

Follow **YAGNI** (You Aren't Gonna Need It), **KISS** (Keep It Simple, Stupid), and **DRY** (Don't Repeat Yourself).

- Write code for current requirements, not speculative futures
- Favor clarity and readability over clever optimizations
- Extract duplicated logic into reusable utilities early

---

## File Naming & Organization

### Naming Convention

- **All files:** kebab-case (e.g., `tooltip-manager.ts`, `floating-menu-lang-handlers.ts`, `api-key-input.tsx`)
- **Components:** `.tsx` extension
- **Utilities/handlers:** `.ts` extension
- **Tests:** `.test.ts` or `.test.tsx` suffix (e.g., `store.test.ts`)
- **Styles:** descriptive with `-style.css` or module suffix (e.g., `popup.css`, `content-style.css`)
- **Directories:** kebab-case (e.g., `shared/`, `content/modules/`, `options/components/`)

### Rationale

Kebab-case filenames are unambiguous when LLM tools (Grep, Glob) scan the codebase. File names like `tooltip-manager.ts` immediately convey purpose without reading content.

### File Size Guideline

**Target:** Keep individual code files under 200 LOC.

- When a file approaches 200 LOC, split into smaller, focused modules
- Example: `tooltip-manager.ts` (manage DOM) + `tooltip-positioning.ts` (viewport logic) instead of one 400+ LOC file
- **Exception:** Configuration files (`.json`, `.css`, `.env`), test fixtures, and vendored code may exceed limit
- **Review trigger:** Any file exceeding 200 LOC in PR → discuss split strategy

Current codebase status:
- Most files are well under 200 LOC
- `tooltip-manager.ts`, `floating-menu.ts`, `highlight-renderer.ts` in the 100–180 LOC range (acceptable)
- No files currently exceed 200 LOC (well-structured)

---

## Language & Type System

### TypeScript

- **Mode:** `strict: true` in `tsconfig.json`
- **No `any`:** Avoid `any` type; use `unknown` + type guard if truly dynamic
- **Exhaustive checks:** Use discriminated unions for message types, state patterns
- **Non-null assertions:** Minimize `!` operator; prefer optional chaining `?.` and nullish coalescing `??`

### Single Source of Truth (SoT)

- **Types:** `vocabulary-extension/src/types/index.ts` — all shared types live here
  - `Word`, `FlashcardData`, `UserStats`, `LLMProvider`, `UserSettings`, `MessageType`, `Message`, `PdfLookupResult`
  - No scattered type definitions; import from `@/types`
- **Enums:** Message actions (e.g., `MessageType.LOOKUP_WORD`) defined in types SoT
- **Constants:** Reusable config (LLM providers, language codes) in dedicated files (`llm-provider-config.ts`, etc.)

---

## Code Quality

### Error Handling

- **Try-catch blocks:** Wrap async operations (fetch, storage) and browser API calls
- **Error messages:** Log the error path, not just the exception; include context
- **Graceful degradation:** Fallback to free translation if LLM fails; show error toast instead of silent failure
- **Example (from codebase):**
  ```typescript
  try {
    const response = await fetch(url);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
  } catch (error) {
    console.error('Translation fetch failed:', error);
    // Fallback or user-facing error message
  }
  ```

### Security Standards

- **No sensitive data in logs:** Never log API keys, user tokens, or plaintext passwords
- **Storage:** API keys stored in `chrome.storage.local` (user-accessible but encrypted by Chrome at rest)
- **Content Security Policy (CSP):** Defined in manifest.ts; inline scripts forbidden
- **XSS protection:** HTML escaping via `content/utils/html-escape.ts` for tooltip content
- **CORS:** All external API calls use host_permissions in manifest.ts

### Logging & Debugging

- Use `console.error()` for exceptions, `console.warn()` for degradation, `console.log()` for development only
- Remove debug logs before commit (use `.test` or `.dev` branches for temporary logging)
- Never log full API responses; summarize for debugging

---

## Component & Module Patterns

### React Components

- **Functional components only** (no class components)
- **Hook composition** over render props or HOCs
- **Single responsibility:** One component = one visual or logical unit
- **Props spread:** Avoid `{...props}` unless explicitly delegating DOM attributes
- **Example:** `flash-card.tsx` owns flip state; `rating-buttons.tsx` owns SM-2 quality selection

### Zustand Store

- **One store per domain:** `useVocabularyStore`, `useStatsStore`, `useSettingsStore`, `useUIStore`
- **Slice pattern:** Organize actions by domain (not required but helpful for large stores)
- **Middleware:** Use `chromeStorage` adapter for persistence; avoid manual storage calls in components
- **Selectors:** Use selectors to minimize re-renders
  ```typescript
  const wordCount = useVocabularyStore((s) => s.words.length);
  ```

### Message Handlers

- **File per handler:** `audio-handler.ts`, `notification-handler.ts`, `word-handler.ts`, etc.
- **Handler signature:**
  ```typescript
  export const handleActionName = async (
    message: Message,
    sender: chrome.runtime.MessageSender
  ): Promise<ResponseType> => {
    // validate, execute, return
  };
  ```
- **Registration:** Import and register in `service-worker.ts` routing switch
- **Error handling:** Catch and return error object with descriptive message

### Content Script Modules

- **Separate concerns:** `settings-manager.ts` owns settings sync, `tooltip-manager.ts` owns DOM, `highlight-renderer.ts` owns highlight logic
- **Initialization:** Each module exports an `init()` or setup function called from `content-script.ts`
- **Cleanup:** Provide `destroy()` or `remove()` functions for teardown (optional if DOM auto-cleaned)
- **Example flow:**
  ```typescript
  // content-script.ts
  await settingsManager.init();
  await tooltipManager.init();
  await highlightRenderer.restoreHighlights();
  ```

### Shared Components

- **No side effects:** Reusable components in `shared/components/` are pure presentational
- **Props over config:** Pass config via props, not global settings
- **Example:** `lang-dropdown.tsx` accepts `value` and `onChange`, doesn't read store directly

---

## Translation & Internationalization

### Language Support

- **Target languages:** 12 supported (VI, ZH, JA, KO, ES, FR, DE, PT, RU, TH, ID, AR)
- **Language codes:** ISO 639-1 format; reference in `llm-provider-config.ts`
- **UI text:** English only (no i18n framework; simplicity > localization for now)

### Translation Pipeline

1. **Free route:** MyMemory API (no auth, single-word optimized)
2. **LLM route:** 6 providers (OpenAI, Gemini, Grok, OpenRouter, Groq, Mistral)
3. **Decision:** `translateText` → if `useLLMTranslation` and `apiKey` present → LLM; else → free
4. **Response parsing:** Regex-based extraction of synonyms/antonyms/notes from LLM output
5. **Fallback:** If LLM fails, retry with free API or return error

---

## Spaced Repetition (SM-2) Convention

### Entry Points

- **Settings:** Options page → Learning → daily goal trigger
- **Study:** Popup → Study tab → rating buttons (1–5) trigger SM-2 calculation
- **Calculation:** `calculateNextReview(quality)` returns next review date; `getPredictedIntervals` shows buttons labels

### Algorithm

- Quality 1–2: Reset interval to 1 day, repetitions to 0
- Quality 3–5: Increment repetitions, adjust EF (easiness factor), compute interval
- SM-2 formula: `interval = interval * EF` (with adjustments per quality)

### Storage

- Word + SM-2 state in `FlashcardData` (vocabulary store)
- Persisted to `chrome.storage.local` via Zustand adapter

---

## Testing Standards

### Unit Tests (Vitest + jsdom)

- **Coverage target:** >80% on critical paths (translation, SM-2, store, notification helpers)
- **Test file location:** Co-located with source (e.g., `store.test.ts` next to `store.ts`)
- **Mock strategies:**
  - Mock fetch via `vi.stubGlobal('fetch', ...)`
  - Mock chrome APIs via `vitest-chrome` setup
  - Mock Zustand stores via `vi.mock('./store')`
- **Example pattern:**
  ```typescript
  describe('calculateNextReview', () => {
    it('should reset interval for quality < 3', () => {
      const result = calculateNextReview(card, 2);
      expect(result.interval).toBe(1);
    });
  });
  ```

### E2E Tests (Playwright)

- **Run after build:** `npm run build && playwright test`
- **Focus:** Key user flows (lookup → save → study → rate)
- **Config:** `playwright.config.ts` (Chrome Canary, headed mode for debugging)

### Pre-Commit Checks (Husky)

- **Lint:** `npm run lint` (ESLint 9 with strict warnings)
- **Type check:** `npx tsc --noEmit`
- **Staged files:** commitlint enforces conventional commit format

---

## Commit Message Format

Follow **Conventional Commits** (enforced via commitlint):

```
type(scope): brief description

Optional body: explain why, not what (code shows what).

Optional footer: Breaking-change, Fixes #123
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`

Examples:
- `feat(translation): add Grok provider integration`
- `fix(tooltip): correct viewport positioning for RTL text`
- `test(sm2): increase coverage to 85%`
- `docs(readme): update quickstart instructions`

**No AI references** in commit messages (e.g., no "Claude implemented X" or "AI-assisted refactor").

---

## ESLint & Formatting

- **ESLint 9** with strict ruleset
- **No formatter config** (Prettier disabled; rely on ESLint auto-fix)
- **Plugin focus:** `@typescript-eslint`, `react-hooks`, `react-refresh`
- **Command:** `npm run lint` (reports errors); `npm run lint --fix` (auto-fixes)
- **CI enforcement:** GitHub Actions fail if lint errors detected

---

## Versioning & Release

### Version Bumping

- `npm run version:patch` (e.g., 1.0.4 → 1.0.5)
- `npm run version:minor` (e.g., 1.0.5 → 1.1.0)
- `npm run version:major` (e.g., 1.0.5 → 2.0.0)
- Updates `package.json` and auto-synced to manifest.ts via `import packageJson from '../package.json'`

### Build Modes

- **Dev:** `npm run build` (unobfuscated, sourcemaps)
- **Release:** `npm run build:release` (obfuscated, minified, no sourcemaps)

---

## Configuration Files

- **`tsconfig.json`:** `strict: true`, `moduleResolution: bundler`, path aliases (`@/` → `src/`)
- **`vite.config.ts`:** CRXJS plugin, React plugin, Vitest config
- **`.env.example`:** Template for API keys (not committed; users create `.env.local`)
- **`prettier.config.cjs`:** (if used; currently minimal config)

---

## Open Questions / TODO

1. **design-tokens.css mapping** — Verify token contract and Tailwind integration
2. **vitest-chrome mocking** — Full extent of chrome API stubs unknown; review `vitest.setup.ts`
3. **Firebase dependency audit** — Confirm if dead code or planned feature
