---
status: pending
issue: https://github.com/k3v1n1k88/vocabulary-extension/issues/4
date: 2026-05-08
---

# Brainstorm Summary — Improve Translate Bubble (Issue #4)

## Problem Statement

Two distinct UX bugs in the translate/lookup tooltip ("bubble"):

1. **Bottom-of-page cutoff** — selecting paragraph near viewport bottom causes the bubble to render below the visible area. `tooltip-positioning.ts:adjustForViewport()` only handles horizontal overflow; vertical is unhandled.
2. **Auto-dismiss on outside click** — `tooltip-event-handlers.ts:setupOutsideClickHandler()` removes the tooltip on any document click, forcing user to re-trigger. No manual close control exists.

Issue scope: applies to **both** the word lookup tooltip and the translation tooltip (shared code path via `tooltip-manager.ts`).

## Decisions

| Decision | Choice |
|---|---|
| Overflow strategy | Flip above selection when no room below; clamp if no room either way |
| Dismiss model | X button + Escape key only; remove outside-click dismiss |
| Scope | Both word lookup tooltip + translation tooltip |
| Floating menu | Unchanged (still outside-click dismiss — correct UX for transient menu) |
| Lingering tooltip on new selection without action | Accept as known v1 limitation; address if reported |

## Evaluated Approaches

### Vertical overflow

| Approach | Verdict |
|---|---|
| **Flip above + bounded max-height** | ✅ Chosen — standard pattern, predictable |
| Estimate height from constants | ❌ Fragile for long translations |
| Auto-scroll page to fit | ❌ Intrusive, moves user content unexpectedly |
| Pure max-height + scrollbar | ❌ Bad UX — scrolling inside tooltip |

### Dismiss

| Approach | Verdict |
|---|---|
| **X + Esc, drop outside-click** | ✅ Chosen — matches issue request |
| Keep outside-click + add X | ❌ X becomes redundant |
| Outside-click after grace period | ❌ Unpredictable timing |

## Recommended Solution

### Fix 1 — Vertical viewport adjustment

**File:** `vocabulary-extension/src/content/modules/tooltip-positioning.ts`

- CSS cap: `max-height: min(60vh, 500px)` on tooltip container.
- New helper `adjustForViewportVertical(position, height, selectionRect)`:
  - If `top + height > scrollY + innerHeight - margin` → flip above
  - If flipped also overflows top → clamp to `scrollY + margin`
- After mount, measure tooltip via `requestAnimationFrame` + `getBoundingClientRect()`, then apply vertical adjustment.
- Re-run on content updates (loading → loaded → error) since tooltip height changes.

### Fix 2 — Manual dismiss (X + Escape)

**Files:**
- `tooltip-shared-elements.ts` — new `createCloseButtonHtml()` helper.
- `tooltip-templates.ts` — embed X in `createTooltipHTML`, `createTranslationTooltipHTML`, `createLoadingHTML`, `createErrorHTML` headers.
- `tooltip-event-handlers.ts` — replace `setupOutsideClickHandler()` with `setupCloseButtonHandler()` + `setupEscapeKeyHandler()`. Esc listener must be document-scoped (tooltip not focusable).
- `tooltip-manager.ts` — wire new handlers.
- CSS — top-right X button, 24×24, hover state, `aria-label="Close"`, focusable.

## Files To Modify

```
vocabulary-extension/src/content/modules/
├── tooltip-positioning.ts        (vertical adjustment logic)
├── tooltip-manager.ts            (re-measure on mount + content update)
├── tooltip-shared-elements.ts    (createCloseButtonHtml helper)
├── tooltip-templates.ts          (embed X in 4 templates)
├── tooltip-event-handlers.ts     (Esc + close button; drop outside-click)
└── (CSS file — locate during implementation)
```

Tests: `tooltip-shared-elements.test.ts` must cover new close button.

## Behavior Matrix (Before → After)

| Trigger | Before | After |
|---|---|---|
| Selection near bottom → translate | Bubble cut off | Flips above selection |
| Click outside tooltip | Closes | No effect |
| Click X button | (no button) | Closes |
| Press Escape | No effect | Closes |
| New selection → trigger lookup | Replaces | Replaces (unchanged) |
| New selection, no action | Old tooltip closed | Old tooltip lingers (v1 limit) |

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Tooltip "jump" on flip | Measure via `requestAnimationFrame` before paint — 1-frame max |
| Lingering tooltip on idle new selection | Accepted v1 limitation; defer to feedback |
| Esc listener leak | Tie cleanup to `removeTooltip()` like existing outside-click cleanup |
| A11y regression | X button: `aria-label`, focusable, keyboard-activatable |
| CSS conflict on host pages | Reuse existing `vocab-` namespace; high z-index already set |

## Success Criteria

1. Selecting text near page bottom → bubble flips above and is fully visible.
2. Bubble persists when clicking outside.
3. X button + Esc both close the bubble.
4. New selection + click Translate replaces old bubble (no regression).
5. All existing unit tests pass.
6. New tests cover: vertical flip logic, close button click, Escape key.
7. Manual smoke test: works on long article (e.g. Wikipedia bottom paragraph).

## Next Steps

- (Pending user choice) Run `/ck:plan` to produce phased implementation plan with TODO breakdown.
- Phase suggestion: (1) positioning fix → (2) close button + Esc → (3) tests → (4) manual QA on long pages.

## Unresolved Questions

- Which CSS file owns tooltip styles? Locate during implementation phase.
- Should X button have a tooltip/title `"Close (Esc)"` for discoverability? Default yes unless conflicts with design tokens.
- Is there an existing icon convention for X (SVG inline vs Heroicons)? Check `tooltip-shared-elements.ts` icon patterns.
