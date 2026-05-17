# Phase 01 — Vertical Positioning Fix

## Context Links

- Brainstorm: [`brainstorm-summary.md`](./brainstorm-summary.md) (Fix 1)
- Existing positioning module: `vocabulary-extension/src/content/modules/tooltip-positioning.ts:1-79`
- Existing manager: `vocabulary-extension/src/content/modules/tooltip-manager.ts:1-178`
- CSS file: `vocabulary-extension/src/content/content-style.css` (tooltip block at line 250+)

## Overview

- **Date:** 2026-05-08
- **Priority:** P2
- **Status:** pending
- **Estimated:** 60min
- **Description:** Fix bottom-of-page cutoff. Flip tooltip above the selection rect when no room below; clamp if neither side fits. Cap container height with CSS. Measure actual rendered tooltip height via `requestAnimationFrame` after mount, re-apply on content updates.

## Key Insights

- Current `adjustForViewport()` only handles **horizontal** overflow (lines 55-69). Vertical is unhandled — root cause of issue.
- `calculateTooltipPosition()` (line 28) computes `rect.bottom + scrollY + 10` from the selection — but the rect is **lost** after return. Vertical adjustment needs the selection rect (top of selection = where to flip to). Solution: have `calculateTooltipPosition()` return both position **and** the source selection rect (or null when fallback).
- Tooltip height is **unknown** before mount (content varies — loading spinner vs full word card vs error). Must measure after `appendChild` via `requestAnimationFrame` then apply correction.
- Content updates (`updateTooltipWithWord`, `updateTooltipWithTranslation`, error path in `showErrorTooltip`) change inner HTML and thus height — must re-measure on each.
- CSS cap `max-height: min(60vh, 500px)` keeps the tooltip bounded so flipped-above never exceeds the available space above selection in absolute worst cases.

## Requirements

### Functional

- When selection is near viewport bottom and tooltip would overflow below, flip the tooltip to render **above** the selection (with the same 10px gap).
- If the tooltip is too tall to fit above either, clamp `top` to `scrollY + margin` (10px) so the top edge stays visible (max-height ensures it doesn't grow off-screen).
- No effect when tooltip fits below (existing behavior preserved — regression check).
- Apply on initial mount and after every content update (loading → loaded → error).

### Non-Functional

- No visible "jump" — measure within the same frame as mount via `requestAnimationFrame`.
- No new dependencies.
- File stays under 200 LOC (currently 79).

## Architecture

### Data Flow

```
calculateTooltipPosition()
  ├─ returns { position, selectionRect | null }
adjustForViewport(position, maxWidth)         ── horizontal (existing, unchanged)
mountTooltip(el)
  └─ requestAnimationFrame(() => {
       const rect = el.getBoundingClientRect()
       const adjusted = adjustForViewportVertical(currentPos, rect.height, selectionRect)
       el.style.top = adjusted.top + 'px'
     })
```

### Helper Signature

```ts
export function adjustForViewportVertical(
  position: TooltipPosition,
  tooltipHeight: number,
  selectionRect: DOMRect | null,
  margin = 10
): TooltipPosition
```

Logic:
1. Compute viewport bottom: `viewportBottom = scrollY + innerHeight - margin`.
2. If `position.top + tooltipHeight <= viewportBottom` → return `position` unchanged.
3. Else, if `selectionRect` provided → try flipped: `flippedTop = selectionRect.top + scrollY - tooltipHeight - margin`.
4. If `flippedTop >= scrollY + margin` → return `{ left, top: flippedTop }`.
5. Else clamp: return `{ left, top: scrollY + margin }`.

## Related Code Files

### Modify

- `vocabulary-extension/src/content/modules/tooltip-positioning.ts` — add `adjustForViewportVertical`; refactor `calculateTooltipPosition` to also return the source `selectionRect: DOMRect | null`; update or replace `getFinalTooltipPosition` to no longer be the only entry point (manager will call vertical adjust separately post-mount).
- `vocabulary-extension/src/content/modules/tooltip-manager.ts` — store `currentSelectionRect` per show call (or pass through); add post-mount `requestAnimationFrame` measure-and-adjust in `mountTooltip`; re-measure on `updateTooltipWithWord` / `updateTooltipWithTranslation` / error path.
- `vocabulary-extension/src/content/content-style.css` — add `max-height: min(60vh, 500px); overflow: auto;` to `.vocab-tooltip-content` (around line 256), confirm doesn't conflict with `width: max-content`.

### Create

- None.

### Delete

- None.

## Implementation Steps

1. **Refactor `calculateTooltipPosition()`** to return `{ position: TooltipPosition; selectionRect: DOMRect | null }`. Saved-position and fallback paths return `selectionRect: null` (no flip-above target).
2. **Add `adjustForViewportVertical(position, height, selectionRect, margin?)`** per logic above. Use `window.scrollY`/`window.innerHeight` from DOM (matches existing module style — no SSR concerns since content script).
3. **Update `getFinalTooltipPosition`** signature: return `{ position, selectionRect }` so the manager can call vertical adjustment post-mount. Keep horizontal `adjustForViewport` call inline as today.
4. **Wire into `tooltip-manager.ts`:**
   a. In each `showXxx()` call site, capture `{ position, selectionRect }` from positioning module.
   b. In `createTooltipElement`, set initial `top` from `position.top` (current behavior).
   c. In `mountTooltip(el, selectionRect)`, after `appendChild`, call `requestAnimationFrame(() => measureAndAdjust(el, selectionRect))`.
   d. `measureAndAdjust` reads `el.getBoundingClientRect().height`, computes vertical-adjusted top, sets `el.style.top`. Store `selectionRect` at module scope so update paths can re-use.
   e. Add same RAF call at end of `updateTooltipWithWord`, `updateTooltipWithTranslation`, and the existing-tooltip branch of `showErrorTooltip`.
5. **CSS:** edit `content-style.css` `.vocab-tooltip-content` block (line 256) — add `max-height: min(60vh, 500px); overflow-y: auto;` (overflow only kicks in when content exceeds the cap).
6. **Compile check** — run `npm run build` (or `tsc --noEmit`) to confirm no type errors.

## Todo List

- [ ] Refactor `calculateTooltipPosition` return shape to include `selectionRect`
- [ ] Add `adjustForViewportVertical` helper in `tooltip-positioning.ts`
- [ ] Update `getFinalTooltipPosition` return signature
- [ ] Add module-scope `currentSelectionRect` ref in `tooltip-manager.ts`
- [ ] Add `measureAndAdjust(el)` helper in `tooltip-manager.ts`
- [ ] Wire RAF measurement into `mountTooltip`
- [ ] Wire re-measure into `updateTooltipWithWord`
- [ ] Wire re-measure into `updateTooltipWithTranslation`
- [ ] Wire re-measure into `showErrorTooltip` (existing-tooltip branch)
- [ ] Add `max-height: min(60vh, 500px); overflow-y: auto` to `.vocab-tooltip-content`
- [ ] Run `npm run build` — confirm zero type/compile errors
- [ ] Verify file LOC counts: `tooltip-positioning.ts` <200, `tooltip-manager.ts` <200

## Success Criteria

- Selecting text near viewport bottom → tooltip flips above selection and is fully visible.
- Selecting text near viewport top → tooltip renders below (no regression).
- Selecting in middle of viewport → tooltip renders below (no regression).
- Tooltip with very long translation does not exceed `min(60vh, 500px)` and scrolls internally.
- No visible "jump" between mount and adjusted position (1-frame correction).
- `npm run build` passes; no TypeScript errors.

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Visible position jump on flip | Med | Low | Use RAF before paint — single-frame correction |
| Selection rect stale after content update | Med | Med | Store rect at module scope on first show; re-use on updates (selection hasn't moved) |
| `getBoundingClientRect()` returns 0 height before paint | Low | Med | RAF guarantees post-layout measurement |
| Saved position from floating menu has no rect → cannot flip | Low | Low | Acceptable: floating menu position is user-chosen, no overflow expected |
| `overflow-y: auto` conflicts with dropdowns inside tooltip | Med | Med | Audit dropdowns (`vocab-source-lang-dropdown`, `vocab-target-lang-dropdown`) — they may need `position: fixed` or escape via portal. **If conflict found, escalate before merge.** |

## Security Considerations

- No new user input surfaces; pure DOM measurement and styling.
- No XSS surface change.

## Next Steps

- Phase 02 can run in parallel (no file overlap).
- Phase 03 must wait until Phase 02 lands (close button rendered before handler binds).
- Phase 04 tests this phase's `adjustForViewportVertical` helper.
