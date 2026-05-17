# Phase 05 — Manual QA

## Context Links

- Brainstorm: [`brainstorm-summary.md`](./brainstorm-summary.md) (Success Criteria 1-4, 7)
- Behavior matrix: brainstorm-summary.md → "Behavior Matrix (Before → After)"

## Overview

- **Date:** 2026-05-08
- **Priority:** P2
- **Status:** pending
- **Estimated:** 30min
- **Description:** Manual smoke test on real pages after all unit tests pass. Confirms behavior matrix from brainstorm. No code changes.

## Key Insights

- Unit tests cover logic; manual QA covers integration with real DOM, real selection rects, real CSS, and host-page interaction edge cases.
- Wikipedia is the canonical "long article" target — pages reliably exceed viewport.
- Dark mode must be verified visually because CSS variables differ.

## Requirements

### Test Environment

- Build extension: `cd vocabulary-extension && npm run build`.
- Load unpacked into Chrome from `vocabulary-extension/dist/`.
- Test pages:
  - **Long page:** https://en.wikipedia.org/wiki/Lexicography (long article — bottom paragraph easily reachable).
  - **Short page:** https://example.com (minimal content — top-of-viewport selection).
  - **Dark page:** any site with `prefers-color-scheme: dark` or a known dark-themed page.

### QA Checklist

#### Vertical positioning (Phase 01)

- [ ] **Bottom selection (long page):** Scroll to bottom of Wikipedia article. Select a word in last paragraph. Click Translate. **Expected:** bubble flips ABOVE the selection, fully visible (no cutoff).
- [ ] **Top selection (regression):** Reload page. Select a word in first paragraph. Click Translate. **Expected:** bubble renders BELOW selection (existing behavior unchanged).
- [ ] **Middle selection (regression):** Scroll to middle. Select a word. Click Translate. **Expected:** bubble renders BELOW.
- [ ] **Long translation:** Select an entire long sentence and Translate. **Expected:** bubble bounded by `min(60vh, 500px)`; if content exceeds, internal scroll appears.
- [ ] **Content update re-measure:** Trigger word lookup (loading → loaded). **Expected:** bubble adjusts on content load if height grew (no flash, no cutoff).

#### Dismiss (Phases 02 + 03)

- [ ] **Outside click persists:** Open bubble. Click on empty area of page (outside bubble). **Expected:** bubble stays.
- [ ] **Outside click on text:** Open bubble. Click on a different word in the article (without selecting). **Expected:** bubble stays.
- [ ] **X button click:** Open bubble. Click the X in top-right. **Expected:** bubble closes.
- [ ] **Escape closes:** Open bubble. Press Escape. **Expected:** bubble closes.
- [ ] **Escape after focus moved:** Open bubble. Click into the URL bar. Press Escape. **Expected:** bubble closes (Escape is document-level).
- [ ] **New selection regression:** Select word A → Translate (bubble appears). Select word B → click Translate (or floating menu). **Expected:** old bubble replaced by new (no zombie bubble).

#### A11y (Phase 02)

- [ ] **Tab to X:** Open bubble. Press Tab. **Expected:** focus reaches the X button (visible focus ring).
- [ ] **Enter activates X:** With X focused, press Enter. **Expected:** bubble closes.
- [ ] **Title tooltip:** Hover X for 1s. **Expected:** OS tooltip shows "Close (Esc)".
- [ ] **Screen reader:** Enable screen reader (NVDA on Windows / VoiceOver on Mac). Tab to X. **Expected:** announces "Close, button".

#### Visual (Phase 02)

- [ ] **Light mode visual:** X button visible top-right, doesn't overlap word/badge row. Hover changes background.
- [ ] **Dark mode visual:** Open dark-themed page. Trigger bubble. **Expected:** X button visible against dark background (if dark mode CSS exists; else file follow-up issue).
- [ ] **Long word/text:** Select a very long compound word. **Expected:** header wraps or truncates without overlapping X button.

#### Resilience

- [ ] **Multi-cycle:** Open/close bubble 10× rapidly via Esc. **Expected:** no console errors, no leaked listeners (DevTools → Memory → snapshot before/after).
- [ ] **Console clean:** No errors or warnings in console during any of the above.

## Sign-Off Conditions

All checkboxes above ticked AND no console errors AND visual issues filed as separate follow-up if not blockers.

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Host page's own CSS breaks `.vocab-close-btn` styling | Low | Low | `vocab-` namespace + high z-index already in place |
| Long word overlaps X due to insufficient header padding | Med | Low | Adjusted via `padding-right: 32px` in Phase 02 — verify here |
| Esc conflicts with host page (e.g., Wikipedia's own Esc handler) | Low | Med | Don't `stopPropagation`; host handler still fires |
| Dark mode reveals invisible X button | Med | Med | If found, file as Phase 02 follow-up (not a blocker for issue #4 close) |

## Security Considerations

- N/A — manual testing only, no code changes.

## Next Steps

<!-- Updated: Validation Session 1 - submodule commit boundary explicit -->
- After sign-off: open PR **inside the `vocabulary-extension` submodule** with conventional commit `fix(tooltip): flip-above vertical overflow + manual dismiss (X + Esc) (#4)`.
- After submodule PR merges: in the parent repo (`vocabulary-project`), bump the submodule pointer (`git add vocabulary-extension && git commit -m "chore: bump vocabulary-extension to <sha>"`).
- Update `docs/project-changelog.md` (parent repo) with bug fix entry referencing the submodule SHA.
- Close issue #4 on the submodule's GitHub repo (`k3v1n1k88/vocabulary-extension`).

## Unresolved Questions

<!-- Updated: Validation Session 1 correction - no dark mode in stylesheet -->
- Dark mode CSS coverage: **none exists** in `content-style.css` (re-grepped at implementation). Tracked as a separate follow-up issue (whole-stylesheet dark mode, not just close-btn).
- Touch device behavior (mobile Chrome on Android): not explicitly tested. Out of scope for this issue but worth tracking.
