# Phase 04 — Unit Tests

## Context Links

- Brainstorm: [`brainstorm-summary.md`](./brainstorm-summary.md) (Success Criteria 5-6)
- Existing test (extend): `vocabulary-extension/src/content/modules/tooltip-shared-elements.test.ts`
- Phases under test: [phase-01](./phase-01-vertical-positioning-fix.md), [phase-02](./phase-02-close-button-html-css.md), [phase-03](./phase-03-dismiss-handlers.md)

## Overview

- **Date:** 2026-05-08
- **Priority:** P2
- **Status:** pending
- **Estimated:** 60min
- **Description:** Cover the new behavior with unit tests using existing vitest + jsdom setup. Three test files: extend `tooltip-shared-elements.test.ts` (close button HTML), new `tooltip-positioning.test.ts` (flip-above logic), new `tooltip-event-handlers.test.ts` (X click, Esc, outside-click regression).

## Key Insights

- Existing test stack: `vitest` + jsdom. See `tooltip-shared-elements.test.ts:1` (`import { describe, it, expect } from 'vitest'`).
- jsdom provides `document`, `window`, `KeyboardEvent`, `MouseEvent`, `getBoundingClientRect`. **`getBoundingClientRect` returns zeros in jsdom by default** — must mock for positioning tests.
- `chrome.runtime` is mocked in existing tests (per task brief). Reuse pattern.
- `tooltip-event-handlers.ts` uses module-scope state (callback refs from `initEventHandlers`). Tests must call `initEventHandlers(getTooltip, removeTooltip, ...)` with mocks before exercising handlers.

## Requirements

### Functional Coverage

**`tooltip-positioning.test.ts` (NEW)**
- `adjustForViewportVertical` — fits below: returns position unchanged.
- `adjustForViewportVertical` — overflows below, room above: flips to above selection.
- `adjustForViewportVertical` — overflows below AND above: clamps to `scrollY + margin`.
- `adjustForViewportVertical` — `selectionRect = null` (saved/fallback path): clamps without flipping.

**`tooltip-shared-elements.test.ts` (EXTEND)**
- `TOOLTIP_ICONS.close` exists and contains `<svg`.
- `createCloseButtonHtml` — contains `vocab-close-btn` class.
- `createCloseButtonHtml` — contains `aria-label="Close"`.
- `createCloseButtonHtml` — contains `type="button"`.
- `createCloseButtonHtml` — contains `title="Close (Esc)"`.

**`tooltip-event-handlers.test.ts` (NEW)**
- Click on `.vocab-close-btn` → calls `removeTooltip`.
- Press `Escape` → calls `removeTooltip`.
- Press other key (e.g. `Enter`, `a`) → does NOT call `removeTooltip`.
- **Regression:** click outside tooltip → does NOT call `removeTooltip` (proves outside-click removal).
- Cleanup function returned by `setupEscapeKeyHandler` removes the listener (verify by calling cleanup, then dispatching Esc, expect no further `removeTooltip` calls).

### Non-Functional

- Tests run under existing `npm test` command (no new tooling).
- Each test file <200 LOC.
- No flakiness — deterministic via mocked `getBoundingClientRect` and direct event dispatch.

## Architecture

### Mocking `getBoundingClientRect` (jsdom)

```ts
// In test setup for tooltip-positioning.test.ts
const mockRect = (overrides: Partial<DOMRect>): DOMRect => ({
  top: 0, left: 0, right: 0, bottom: 0,
  width: 0, height: 0, x: 0, y: 0,
  toJSON: () => ({}),
  ...overrides
} as DOMRect)
```

Set `window.innerHeight` and `window.scrollY` directly:
```ts
Object.defineProperty(window, 'innerHeight', { value: 800, writable: true })
Object.defineProperty(window, 'scrollY', { value: 0, writable: true })
```

### Setup for `tooltip-event-handlers.test.ts`

```ts
let mockTooltip: HTMLDivElement
let removeSpy: ReturnType<typeof vi.fn>

beforeEach(() => {
  mockTooltip = document.createElement('div')
  mockTooltip.id = 'vocabulary-tooltip'
  mockTooltip.innerHTML = `<button class="vocab-close-btn">X</button>`
  document.body.appendChild(mockTooltip)
  removeSpy = vi.fn()
  initEventHandlers(() => mockTooltip, removeSpy, vi.fn(), vi.fn())
})

afterEach(() => {
  mockTooltip.remove()
})
```

## Related Code Files

### Modify

- `vocabulary-extension/src/content/modules/tooltip-shared-elements.test.ts` — add `describe('TOOLTIP_ICONS.close')` and `describe('createCloseButtonHtml')` blocks.

### Create

- `vocabulary-extension/src/content/modules/tooltip-positioning.test.ts` — new file.
- `vocabulary-extension/src/content/modules/tooltip-event-handlers.test.ts` — new file.

### Delete

- None.

## Implementation Steps

1. **Extend `tooltip-shared-elements.test.ts`:**
   a. Add `expect(TOOLTIP_ICONS.close).toContain('<svg')` to existing `TOOLTIP_ICONS` describe block.
   b. Update import to include `createCloseButtonHtml`.
   c. Add new `describe('createCloseButtonHtml', ...)` with 4 assertions: class, aria-label, type, title.
2. **Create `tooltip-positioning.test.ts`:**
   a. Import `adjustForViewportVertical` from positioning module.
   b. Setup helper to set `window.innerHeight` and `window.scrollY`.
   c. Test 1: `position={left:100, top:100}`, `height=200`, `selectionRect={top:50}`, viewport 800h → no overflow → returns input unchanged.
   d. Test 2: `position={left:100, top:700}`, `height=200`, `selectionRect={top:680}`, viewport 800h → overflows; flipped top = `680 - 200 - 10 = 470` → returns `{left:100, top:470}`.
   e. Test 3: `position={left:100, top:700}`, `height=900` (huge), `selectionRect={top:680}`, viewport 800h → flip would be `-230` < `scrollY+margin=10` → clamps to `{left:100, top:10}`.
   f. Test 4: `selectionRect=null`, overflows → clamps to `{left:100, top:scrollY+margin}`.
3. **Create `tooltip-event-handlers.test.ts`:**
   a. Setup per Architecture above.
   b. Test 1: `setupCloseButtonHandler()`; click `.vocab-close-btn`; expect `removeSpy` called once.
   c. Test 2: `setupEscapeKeyHandler()`; dispatch `keydown` Escape on document; expect `removeSpy` called once.
   d. Test 3: `setupEscapeKeyHandler()`; dispatch `keydown` Enter; expect `removeSpy` NOT called.
   e. Test 4 (regression): no outside-click handler exists. Dispatch `click` on `document.body` outside tooltip; expect `removeSpy` NOT called. Also assert `setupOutsideClickHandler` is NOT exported from module: `expect((handlersModule as any).setupOutsideClickHandler).toBeUndefined()`.
   f. Test 5: cleanup function. Call returned cleanup; dispatch Escape again; expect `removeSpy` call count unchanged.
4. **Run tests:**
   ```
   cd vocabulary-extension
   npm test
   ```
   Expect: all new tests pass; all existing tests still pass.
5. **If failing:** read failure output, fix tests OR fix implementation (if test reveals real bug). Do NOT skip or `.skip` tests.

## Todo List

- [ ] Update `tooltip-shared-elements.test.ts` import to include `createCloseButtonHtml`
- [ ] Add `TOOLTIP_ICONS.close` assertion
- [ ] Add `createCloseButtonHtml` describe block (4 assertions)
- [ ] Create `tooltip-positioning.test.ts` with viewport mocking helper
- [ ] Add 4 `adjustForViewportVertical` test cases (fits / flips / clamps / null rect)
- [ ] Create `tooltip-event-handlers.test.ts` with `initEventHandlers` setup
- [ ] Add close-button click test
- [ ] Add Escape keydown test
- [ ] Add non-Escape keydown negative test
- [ ] Add outside-click regression test (asserts handler removed AND no closure on outside click)
- [ ] Add cleanup-removes-listener test
- [ ] Run `npm test` — all green
- [ ] Verify each test file <200 LOC

## Success Criteria

- All new tests pass.
- All existing tests still pass (no regression).
- `npm test` exits 0.
- `setupOutsideClickHandler` provably absent from module exports.
- Each test file deterministic (no flake on 5 consecutive runs).

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| jsdom `KeyboardEvent` differs from real browsers | Low | Med | Use `new KeyboardEvent('keydown', { key: 'Escape' })` — supported by jsdom |
| `getBoundingClientRect` mock leaks between tests | Med | Low | Use `beforeEach`/`afterEach` to set/restore; or use vi.spyOn |
| Module-scope state in `tooltip-event-handlers.ts` leaks across tests | Med | Med | Re-call `initEventHandlers` in each `beforeEach` to reset callbacks |
| Test file exceeds 200 LOC | Low | Low | Split into `tooltip-event-handlers-close.test.ts` + `-escape.test.ts` if needed |

## Security Considerations

- No production code reached by tests; no auth/secrets touched.

## Next Steps

- Phase 05 (manual QA) starts only after this phase passes.
- If tests reveal a bug in Phase 01-03 implementation, fix in the relevant phase before proceeding.
