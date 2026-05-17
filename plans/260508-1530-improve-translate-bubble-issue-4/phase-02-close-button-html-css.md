# Phase 02 — Close Button HTML + CSS

## Context Links

- Brainstorm: [`brainstorm-summary.md`](./brainstorm-summary.md) (Fix 2)
- Shared elements: `vocabulary-extension/src/content/modules/tooltip-shared-elements.ts:1-160`
- Templates: `vocabulary-extension/src/content/modules/tooltip-templates.ts:1-168`
- Error template: `vocabulary-extension/src/content/modules/tooltip-error-template.ts:1-85`
- CSS: `vocabulary-extension/src/content/content-style.css` (tooltip block at line 250+)

## Overview

- **Date:** 2026-05-08
- **Priority:** P2
- **Status:** pending
- **Estimated:** 45min
- **Description:** Add accessible X (close) button to all 4 tooltip templates (word, translation, loading, error). Pure HTML + CSS — no event wiring (Phase 03 owns that).

## Key Insights

- `TOOLTIP_ICONS` (lines 12-44) currently has: speaker, speakerSmall, bookmark, copy, settings, error, chevronDown, robot. **No `close` icon** — must add.
- The 4 templates all open with `<div class="vocab-tooltip-content...">` then a `<div class="vocab-header">` block. Close button should sit absolute-positioned inside `.vocab-tooltip-content` (top-right), not inside the header — keeps templates clean and the button stays consistent across content variants.
- Existing icons use `viewBox="0 0 24 24"` with `stroke="currentColor"` — match this convention.
- Error template (`tooltip-error-template.ts:71-83`) uses `vocab-error-card` class — close button must work for that variant too.

## Requirements

### Functional

- All 4 tooltip templates render an X button at the top-right of the container.
- Button has class `.vocab-close-btn` (single hook for Phase 03 event handlers).
- Hit area: 24×24px minimum (a11y target).
- `aria-label="Close"`, `title="Close (Esc)"` for discoverability.
- `type="button"` (prevents form submit if ever embedded in form).

### Non-Functional

- Visible focus ring (keyboard nav).
- Hover state (color/background change).
- No CSS specificity conflict with host page (uses `vocab-` namespace).
- Dark mode: matches existing tooltip dark mode if present in CSS.
- File LOC: `tooltip-shared-elements.ts` currently 160 — adding ~20 LOC stays well under 200.

## Architecture

### Component Layering

```
TOOLTIP_ICONS.close (new SVG)
  └─ createCloseButtonHtml()  [new helper in tooltip-shared-elements.ts]
       └─ embedded in: createTooltipHTML, createTranslationTooltipHTML,
                       createLoadingHTML, createErrorHTML
```

### HTML Output Shape

```html
<button type="button" class="vocab-close-btn" aria-label="Close" title="Close (Esc)">
  <!-- TOOLTIP_ICONS.close (X SVG) -->
</button>
```

### CSS Position Strategy

- Container `.vocab-tooltip-content` already has `padding: 16px` and `position` is implicit static. Set `position: relative` on `.vocab-tooltip-content` (verify no break to existing layout).
- `.vocab-close-btn` → `position: absolute; top: 8px; right: 8px;`.
- Account for the button's 24×24 footprint by ensuring header content has enough right-padding (`.vocab-header { padding-right: 32px }` or similar) so word/badge row doesn't overlap with X.

## Related Code Files

### Modify

- `vocabulary-extension/src/content/modules/tooltip-shared-elements.ts` — add `close` to `TOOLTIP_ICONS`; add `createCloseButtonHtml()` helper.
- `vocabulary-extension/src/content/modules/tooltip-templates.ts` — embed `createCloseButtonHtml()` at start of each template's `<div class="vocab-tooltip-content">`. Affects `createTooltipHTML` (line 25), `createTranslationTooltipHTML` (line 98), `createLoadingHTML` (line 148).
- `vocabulary-extension/src/content/modules/tooltip-error-template.ts` — embed at start of `vocab-error-card` div in `createErrorHTML` (line 71).
- `vocabulary-extension/src/content/content-style.css` — add `.vocab-close-btn` block; add `position: relative` to `.vocab-tooltip-content` (line 256); add right-padding to `.vocab-header` to avoid overlap.

### Create

- None.

### Delete

- None.

## Implementation Steps

1. **Locate icon convention** (already done — see Key Insights). Add `close` to `TOOLTIP_ICONS` with `viewBox="0 0 24 24"`, `stroke="currentColor"`, two diagonal `<line>` elements forming X. Width/height 16.
2. **Add `createCloseButtonHtml()`** in `tooltip-shared-elements.ts` (after `createSettingsButtonHtml`, before `createErrorCodeBadgeHtml`):
   ```ts
   export function createCloseButtonHtml(): string {
     return `<button type="button" class="vocab-close-btn" aria-label="Close" title="Close (Esc)">
       ${TOOLTIP_ICONS.close}
     </button>`
   }
   ```
3. **Import + embed in `tooltip-templates.ts`** — add to imports from `./tooltip-shared-elements`. Insert `${createCloseButtonHtml()}` immediately after `<div class="vocab-tooltip-content...">` opening tag in each of the 3 templates.
4. **Embed in `tooltip-error-template.ts`** — add `createCloseButtonHtml` to existing `tooltip-shared-elements` import. Insert immediately after `<div class="vocab-tooltip-content vocab-error-card">` opening tag.
<!-- Updated: Validation Session 1 - use --vocab-primary-500 (correct token); add dark-mode block -->
5. **CSS edits in `content-style.css`:**
   - `.vocab-tooltip-content` (line 256): add `position: relative;`.
   - `.vocab-header` (line 297): add `padding-right: 32px;` (or alternative — verify doesn't break existing layout).
   - New block `.vocab-close-btn` (light mode):
     ```css
     .vocab-close-btn {
       position: absolute;
       top: 8px;
       right: 8px;
       width: 24px;
       height: 24px;
       display: inline-flex;
       align-items: center;
       justify-content: center;
       background: transparent;
       border: none;
       border-radius: var(--vocab-radius-sm);
       color: var(--vocab-gray-500);
       cursor: pointer;
       padding: 0;
       transition: background 0.15s, color 0.15s;
     }
     .vocab-close-btn:hover {
       background: var(--vocab-gray-100);
       color: var(--vocab-gray-800);
     }
     .vocab-close-btn:focus-visible {
       outline: 2px solid var(--vocab-primary-500);
       outline-offset: 2px;
     }
     ```
   - **Dark-mode block:** SKIPPED at implementation. Re-grep confirmed `content-style.css` has **zero** `prefers-color-scheme: dark` blocks (the `@media` near line 270 is `(min-width: 1200px)`, responsive sizing). Adding the only dark-mode rule for one button would violate YAGNI. Tracked as separate follow-up.
6. **Compile check** — `npm run build` and verify visually in dev (Phase 03 wires the click; for now button is inert but rendered).

## Todo List

- [ ] Add `close` SVG to `TOOLTIP_ICONS` in `tooltip-shared-elements.ts`
- [ ] Add `createCloseButtonHtml()` helper
- [ ] Embed `createCloseButtonHtml()` in `createTooltipHTML`
- [ ] Embed `createCloseButtonHtml()` in `createTranslationTooltipHTML`
- [ ] Embed `createCloseButtonHtml()` in `createLoadingHTML`
- [ ] Embed `createCloseButtonHtml()` in `createErrorHTML`
- [ ] Add `position: relative` to `.vocab-tooltip-content` in CSS
- [ ] Add `padding-right: 32px` (or equivalent) to `.vocab-header`
- [ ] Add `.vocab-close-btn` CSS block (base + hover + focus-visible) using `--vocab-primary-500`/`--vocab-gray-*` tokens (no `--vocab-primary` — does not exist)
- [ ] ~~Dark-mode overrides~~ — skipped (no existing dark-mode CSS in stylesheet; tracked as follow-up)
- [ ] Run `npm run build` — confirm no type errors
- [ ] Visual smoke check: button renders top-right of all 4 tooltip variants
- [ ] Verify file LOC: `tooltip-shared-elements.ts` <200

## Success Criteria

- All 4 tooltip variants render an X button visible at top-right.
- Button is keyboard-focusable (Tab order) with visible focus ring.
- Hover changes background/color.
- Header content does not visually overlap the close button.
- `aria-label="Close"` present and `title="Close (Esc)"` shown on hover.
- Existing tooltip layouts unchanged (no regression in word/translation/loading/error rendering).

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| `position: relative` on `.vocab-tooltip-content` breaks existing absolute children | Low | Med | Audit existing children — dropdowns currently use inline `display:none` then probably static; verify in dev |
| Header content overlaps with X on narrow tooltip variants | Med | Low | Add `padding-right: 32px` to `.vocab-header`; visual check |
| CSS variables (`--vocab-gray-500` etc.) don't exist | Med | Low | Fallback values inline (`var(--x, fallback)`); check `design-tokens.css` for actual names |
| Dark mode: button invisible on dark background | Low | Med | No dark-mode CSS exists in stylesheet today; tracked as separate follow-up issue (re-verified at implementation, Validation Session 1 correction) |
| Button captures touch events meant for tooltip body | Low | Low | Standard 24×24 hit area is fine; not a custom gesture surface |

## Security Considerations

- Static HTML, no user input interpolated into close button. No XSS surface.
- `type="button"` prevents accidental form submission.

## Next Steps

- Phase 03 wires click handler to `.vocab-close-btn` and Esc key globally.
- Phase 04 tests `createCloseButtonHtml()` HTML shape (presence of class, aria-label, SVG).
