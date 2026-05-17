# Phase 03 — Dismiss Handlers (Drop Outside-Click, Add Esc + X Click)

## Context Links

- Brainstorm: [`brainstorm-summary.md`](./brainstorm-summary.md) (Fix 2)
- Event handlers: `vocabulary-extension/src/content/modules/tooltip-event-handlers.ts:1-162`
- Manager: `vocabulary-extension/src/content/modules/tooltip-manager.ts:1-178`

## Overview

- **Date:** 2026-05-08
- **Priority:** P2
- **Status:** pending
- **Estimated:** 45min
- **Description:** Delete `setupOutsideClickHandler` entirely. Add `setupCloseButtonHandler()` (binds click to `.vocab-close-btn`) and `setupEscapeKeyHandler()` (document-level keydown). Update `tooltip-manager.ts` to wire new handlers and clean both up on `removeTooltip`.

## Key Insights

- Verified via Grep: `setupOutsideClickHandler` is referenced **only** at:
  - `tooltip-event-handlers.ts:50` (definition)
  - `tooltip-manager.ts:13` (import)
  - `tooltip-manager.ts:62` (call site)
  Safe to delete the function entirely.
- Current cleanup pattern: `cleanupOutsideClick` is a single ref of type `(() => void) | null` (manager line 23). New design has **two** cleanups (close-button click + Esc keydown) → use an array or two refs.
- Esc keydown listener must be on `document` (tooltip is not focusable; user may not have focus inside tooltip when pressing Esc).
- Close button click handler needs `e.stopPropagation()` defensively — even though we removed outside-click, future re-introduction of any document-level click handler shouldn't re-close the tooltip.

## Requirements

### Functional

- Click on `.vocab-close-btn` inside the tooltip → tooltip removed.
- Press Escape (anywhere in document) while tooltip is open → tooltip removed.
- Click anywhere outside the tooltip → tooltip persists (regression fix).
- Esc listener does not leak after tooltip removed (cleanup on every `removeTooltip`).
- Multiple tooltip lifecycles work correctly (open → close → open → close, no zombie listeners).

### Non-Functional

- Esc listener must NOT swallow Esc events for other UI on host page when tooltip is closed.
- Esc behavior: only acts when tooltip is currently mounted.
- File LOC: `tooltip-event-handlers.ts` currently 162 — net change: −10 LOC delete + ~25 LOC add ≈ 177. **Trim if reaches 200.**

## Architecture

### Handler Lifecycle

```
mountTooltip(el, selectionRect):
  appendChild
  cleanups = [
    setupCloseButtonHandler(),
    setupEscapeKeyHandler()
  ]

removeTooltip():
  el.remove()
  cleanups.forEach(fn => fn())
  cleanups = []
```

### `setupCloseButtonHandler` Signature

```ts
export function setupCloseButtonHandler(): (() => void) {
  const tooltip = getTooltip()
  if (!tooltip) return () => {}
  const btn = tooltip.querySelector('.vocab-close-btn')
  if (!btn) return () => {}
  const handler = (e: Event) => {
    e.stopPropagation()
    removeTooltip()
  }
  btn.addEventListener('click', handler)
  return () => btn.removeEventListener('click', handler)
}
```

### `setupEscapeKeyHandler` Signature

```ts
export function setupEscapeKeyHandler(): (() => void) {
  const handler = (e: KeyboardEvent) => {
    if (e.key === 'Escape') {
      removeTooltip()
    }
  }
  document.addEventListener('keydown', handler)
  return () => document.removeEventListener('keydown', handler)
}
```

## Related Code Files

### Modify

- `vocabulary-extension/src/content/modules/tooltip-event-handlers.ts` — delete `setupOutsideClickHandler` (lines 47-59); add `setupCloseButtonHandler` and `setupEscapeKeyHandler`.
- `vocabulary-extension/src/content/modules/tooltip-manager.ts` — replace `cleanupOutsideClick: (() => void) | null` with `cleanups: Array<() => void>`; replace import; update `mountTooltip` and `removeTooltip` accordingly.

### Create

- None.

### Delete

- None (function deletion only, not files).

## Implementation Steps

1. **Pre-flight Grep** — re-verify `setupOutsideClickHandler` has no other callers (already done; redo before commit to be safe):
   ```
   Grep "setupOutsideClickHandler" vocabulary-extension/
   ```
2. **In `tooltip-event-handlers.ts`:**
   a. Delete `setupOutsideClickHandler` (current lines 47-59).
   b. Add `setupCloseButtonHandler` per signature above (placed in same neighborhood — near top, after `initEventHandlers`).
   c. Add `setupEscapeKeyHandler` per signature above (immediately after).
3. **In `tooltip-manager.ts`:**
   a. Update import block (lines 11-17) — remove `setupOutsideClickHandler`, add `setupCloseButtonHandler` and `setupEscapeKeyHandler`.
   b. Replace `let cleanupOutsideClick: (() => void) | null = null` (line 23) with `let cleanups: Array<() => void> = []`.
   c. Update `mountTooltip` (line 59) — replace `cleanupOutsideClick = setupOutsideClickHandler()` with:
      ```ts
      cleanups = [
        setupCloseButtonHandler(),
        setupEscapeKeyHandler()
      ]
      ```
   d. Update `removeTooltip` (line 167) — replace `if (cleanupOutsideClick) { cleanupOutsideClick(); cleanupOutsideClick = null }` block (lines 172-175) with:
      ```ts
      cleanups.forEach(fn => fn())
      cleanups = []
      ```
4. **Compile check** — `npm run build`. Confirm zero type errors and zero references to removed export.
5. **Manual smoke test (dev mode):**
   - Open tooltip → click outside → tooltip stays.
   - Open tooltip → click X → tooltip closes.
   - Open tooltip → press Esc → tooltip closes.
   - Open → close → open → close (5 cycles): no console errors, listener count stable (DevTools).

## Todo List

- [ ] Re-Grep `setupOutsideClickHandler` to confirm only 3 references (def + import + call)
- [ ] Delete `setupOutsideClickHandler` from `tooltip-event-handlers.ts`
- [ ] Add `setupCloseButtonHandler` to `tooltip-event-handlers.ts`
- [ ] Add `setupEscapeKeyHandler` to `tooltip-event-handlers.ts`
- [ ] Update import block in `tooltip-manager.ts`
- [ ] Replace `cleanupOutsideClick` ref with `cleanups: Array<() => void>` in `tooltip-manager.ts`
- [ ] Update `mountTooltip` to register both handlers into `cleanups`
- [ ] Update `removeTooltip` to iterate `cleanups`
- [ ] Run `npm run build` — confirm no type errors
- [ ] Manual smoke test: outside click does NOT close
- [ ] Manual smoke test: X click closes
- [ ] Manual smoke test: Esc closes
- [ ] DevTools: open/close 5×, confirm no leaked listeners (event listener count stable)
- [ ] Verify file LOC: `tooltip-event-handlers.ts` <200, `tooltip-manager.ts` <200

## Success Criteria

- Tooltip persists across all click locations outside itself.
- X button click → tooltip removed.
- Esc keypress → tooltip removed.
- No `setupOutsideClickHandler` symbol remains in codebase.
- No leaked Esc listener after tooltip closed (verified via DevTools listener count).
- Existing handlers (audio, save, copy, AI hint, dropdowns) still function — no regression.

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Esc listener leak — accumulates per show cycle | Med | High | Cleanup on `removeTooltip`; verify with DevTools listener count after 5 cycles |
| Esc swallows host-page Esc handlers (e.g., modal close) | Low | Med | Don't `stopPropagation` on Esc — only call `removeTooltip`; host page still receives event |
| User on input field types Esc to clear → tooltip steals it | Low | Low | Acceptable — Esc with tooltip open SHOULD close tooltip; user can re-focus input |
| Close button click bubbles to outside-click handler that doesn't exist anymore | None | None | Outside-click handler removed in this phase; `stopPropagation` is defensive |
| Race: tooltip removed by other code path while Esc handler in-flight | Low | Low | `removeTooltip` is idempotent (checks `if (tooltip)` line 168) |

## Security Considerations

- Document-level keydown listener does not capture or log keystrokes.
- No new privileged API surface.

## Next Steps

- Phase 04 tests this phase's handlers (X click, Esc, outside-click regression).
- Phase 05 manual QA confirms behavior on real pages.
