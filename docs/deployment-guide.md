# Deployment Guide

**Vocabulary Builder Chrome Extension v1.0.5**

---

## Build Modes

### Development Build

```bash
cd vocabulary-extension
npm run build
```

**Output:** `dist/` folder (unobfuscated, sourcemaps included)
**Use case:** Local testing, debugging, developer load

**Contents:**
- `manifest.json` (generated from `src/manifest.ts`)
- `popup.html`, `options.html`, `sidepanel.html` (entry points)
- `popup.js`, `options.js`, `sidepanel.js` (bundled React)
- `content-script.js`, `service-worker.js` (bundled handlers/modules)
- `icons/` (extension icons)
- `.map` files (sourcemaps for debugging)

### Release Build

```bash
cd vocabulary-extension
npm run build:release
```

**Output:** `dist/` folder (obfuscated, minified, no sourcemaps)
**Use case:** Submission to Chrome Web Store or distribution

**Tooling:**
- `rollup-plugin-obfuscator` + `javascript-obfuscator` with `compact: true`
- Sourcemaps disabled for release
- Code size reduced ~30–40%

---

## Local Development & Testing

### Prerequisites

- Node.js 18+
- npm
- Chrome browser (v90+ for core features, v114+ for side panel)

### Setup

```bash
cd vocabulary-extension
npm install
```

### Run Dev Server

```bash
npm run dev
```

**Output:** Starts Vite dev server; watches for file changes and rebuilds incrementally. Outputs to `dist/`.

### Load Extension in Chrome

1. Open Chrome
2. Navigate to `chrome://extensions/`
3. Enable **Developer mode** (toggle in top right)
4. Click **Load unpacked**
5. Select the `dist` folder from this project
6. Extension appears in toolbar; pin it for easy access

### Live Reload

After `npm run dev`, changes to source files auto-rebuild. **Refresh** the extension in `chrome://extensions/` (or press Cmd+R / Ctrl+R on extension detail page).

### Debug Popup/Options/SidePanel

1. Click extension icon → Inspect popup (press Ctrl+Shift+I / Cmd+Shift+I while popup open)
2. Right-click extension → Options → DevTools (automatic)
3. Chrome menu → More tools → Developer tools, then switch to relevant tab

### Debug Content Script

1. Right-click webpage → Inspect
2. Open DevTools console
3. Select content script from dropdown (near top-left of console)
4. Type commands; interact with `window` scope (tooltip/menu/highlight modules are available)

### Debug Service Worker

1. `chrome://extensions/`
2. Find Vocabulary Builder extension
3. Click **Inspect views → service worker**
4. DevTools opens for service worker context; set breakpoints, inspect state

---

## Testing

### Unit Tests

```bash
npm run test
# or
npm run test:unit
```

Runs Vitest with jsdom; watches for changes.

**Files tested:**
- `shared/translation-service.test.ts`
- `shared/spaced-repetition.test.ts`
- `shared/store.test.ts`
- `shared/notification-helpers.test.ts`
- `content/modules/tooltip-shared-elements.test.ts`

### Test Coverage

```bash
npm run test:coverage
```

Outputs coverage report to `coverage/` (target: >80% on critical paths).

### E2E Tests (Playwright)

```bash
npm run build
npm run test:e2e
```

Builds extension first, then runs Playwright tests against built `dist/`.

**Config:** `playwright.config.ts` (Chrome Canary, headless by default)

**Run with UI:**

```bash
npm run test:e2e:ui
```

Opens Playwright Inspector for debugging.

### Linting & Type Check

```bash
npm run lint                  # ESLint
npx tsc --noEmit             # TypeScript strict check
```

Run before commit (enforced by Husky pre-commit hook).

---

## Pre-Commit & CI/CD

### Husky Pre-Commit Hooks

Configured via `.husky/`:

1. **Lint-staged** — Lints only staged files via ESLint
2. **Commitlint** — Validates commit message format (conventional commits)

**Run manually:**

```bash
npm run prepare              # Install Husky hooks
npx husky install           # Ensure hooks active
```

### Commit Message Format

```
type(scope): description

Optional body.

Optional footer.
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`

Example:
```
feat(tooltip): add AI badge to translation results

Displays LLM provider name when AI translation used.
Closes #42
```

**Validation:** `commitlint` enforces format; commit fails if non-compliant.

### GitHub Actions Workflows

Located in `.github/workflows/`:

#### **test.yml** — On PR / push to main

1. Checkout code
2. Install Node.js 18
3. `npm ci` (clean install)
4. `npm run lint` (ESLint)
5. `npx tsc --noEmit` (Type check)
6. `npm run test:unit` (Vitest)
7. `npm run build` (Build check)
8. `npm run test:e2e` (Playwright on Chrome Canary)

**Failure:** Blocks merge if any step fails.

#### **release.yml** — On version tag (manual trigger)

1. Build release: `npm run build:release`
2. Create GitHub release with `dist/` as artifact
3. (Optional) Auto-publish to Chrome Web Store (requires API credentials)

---

## Version Management

### Bump Version

```bash
npm run version:patch         # 1.0.5 → 1.0.6
npm run version:minor         # 1.0.5 → 1.1.0
npm run version:major         # 1.0.5 → 2.0.0
```

**Effect:**
- Updates `package.json` version field
- `src/manifest.ts` auto-syncs via `import packageJson from '../package.json'`
- No git tag created (use `git tag` manually if needed)

### Publish Release (Manual Process)

1. Bump version: `npm run version:patch`
2. Build release: `npm run build:release`
3. Create git commit: `git add . && git commit -m "release: v1.0.6"`
4. Tag: `git tag v1.0.6`
5. Push: `git push origin main && git push origin v1.0.6`
6. GitHub Actions triggers `release.yml` workflow
7. Download `dist.zip` from GitHub release
8. Submit to Chrome Web Store (manual or via API)

---

## Chrome Web Store Submission

### Prerequisites

1. Google Play Developer Account ($5 one-time fee)
2. Extension ZIP from release build
3. Extension icon (128x128 PNG)
4. Store listing assets (screenshots, description)

### Submission Steps

1. Create new item in Chrome Web Store Developer Console
2. Upload extension ZIP (`dist.zip`)
3. Fill in metadata:
   - **Name:** Vocabulary Builder
   - **Description:** (from `package.json` or README)
   - **Screenshots:** 1280x800 or 640x400 PNG/JPG
   - **Category:** Education / Productivity
   - **Languages:** English
4. Review permissions (manifest.ts lists required permissions)
5. Submit for review (24–72 hours typical)
6. Once approved, publish to all users

### Permissions Disclosure

Users will see these required permissions on install:

- **storage** — Save vocabulary locally
- **contextMenus** — Right-click lookup
- **activeTab** — Access current page for lookup
- **notifications** — Daily study reminders
- **alarms** — Schedule reminders
- **sidePanel** — PDF lookup panel
- **Host permissions** — Query Free Dictionary, MyMemory, OpenAI, Gemini, Grok, etc.

---

## Troubleshooting Build / Deployment

### Issue: "Cannot find module '@crxjs/vite-plugin'"

**Solution:**
```bash
npm install
# or if node_modules corrupted:
rm -rf node_modules && npm install
```

### Issue: Build fails with TypeScript errors

```bash
npx tsc --noEmit
```

Review errors and fix; see `code-standards.md` for TypeScript guidelines.

### Issue: ESLint reports errors

```bash
npm run lint -- --fix
# Review auto-fixed issues, manually fix others
```

### Issue: Service worker not updating after code change

1. Stop Vite dev server
2. Go to `chrome://extensions/`
3. Disable and re-enable Vocabulary Builder extension
4. Check service worker console (`Inspect views → service worker`)

### Issue: Content script changes not reflecting

1. Close and reload all tabs open to web pages
2. OR go to `chrome://extensions/`, click reload on Vocabulary Builder
3. Refresh the web page

### Issue: E2E tests fail with "Chrome Canary not found"

Install Playwright browsers:
```bash
npx playwright install
```

### Issue: API keys rejected in options page

1. Ensure key format is correct (e.g., OpenAI keys start with `sk-`)
2. Check rate limits / API account status on provider's dashboard
3. Verify network requests in DevTools (Network tab, filter by provider domain)

---

## Performance Checklist

Before release, verify:

- [ ] Tooltip appears <200ms after right-click
- [ ] Highlighting restored <500ms on page load (measure via Performance tab)
- [ ] Settings sync across tabs <100ms (test via 2 windows, change setting, observe propagation)
- [ ] Flashcard flip animation smooth (60 FPS, no jank)
- [ ] No console errors or warnings (check all 4 contexts: popup, options, sidepanel, content)
- [ ] Storage usage <10 MB for 1000+ words (verify via `chrome://extensions/ → Storage` or DevTools → Application)
- [ ] Bundle size <2 MB (check `dist/` folder size)

---

## Rollback Procedure

If v1.0.6 has critical bug and needs rollback:

1. **From GitHub:** Go to releases, download previous stable version ZIP
2. **From Chrome Web Store:** Unpublish v1.0.6, select previous version as current
3. **Locally:** `git checkout v1.0.5` and rebuild

---

## Security Before Release

- [ ] No API keys in source code (use `.env.local`, not checked in)
- [ ] No sensitive logs in console
- [ ] XSS check: test tooltip with `<img src=x onerror=alert('XSS')>` (should escape)
- [ ] CSP violations: no inline scripts in popup/options HTML
- [ ] CORS: all API calls use host_permissions in manifest.ts

---

## Monitoring & Analytics (Future)

Current v1.0.5 has no built-in analytics. Suggested future additions:

- Error tracking via Sentry (optional)
- Install/usage metrics (opt-in, privacy-preserving)
- Crash reporting with user consent

See `project-roadmap.md` for post-v1.0 roadmap.

---

## Open Questions

1. **Chrome Web Store API credentials** — Where to store / rotate?
2. **Auto-update mechanism** — Chrome handles extension updates automatically; verify cadence with Web Store
3. **Staged rollout** — Does Chrome Web Store support percentage-based rollout? (Yes, via developer console)
