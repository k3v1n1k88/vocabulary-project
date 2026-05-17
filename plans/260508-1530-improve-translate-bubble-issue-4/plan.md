---
title: "Improve Translate Bubble (Issue #4)"
description: "Fix bottom-of-page cutoff via flip-above + replace outside-click dismiss with X button + Escape"
status: in-progress
priority: P2
effort: 4h
branch: fix/tooltip-bubble-issue-4
tags: [bugfix, ui, tooltip, content-script, a11y]
created: 2026-05-08
---

# Plan — Improve Translate Bubble (Issue #4)

## Context

- GitHub issue: https://github.com/k3v1n1k88/vocabulary-extension/issues/4
- Brainstorm (approved): [`brainstorm-summary.md`](./brainstorm-summary.md)
- Scope: word lookup tooltip + translation tooltip (shared path via `tooltip-manager.ts`).
- Out of scope: floating menu (still outside-click dismiss — correct UX); lingering tooltip on idle new selection (accepted v1 limit).

## Approved Design (from brainstorm)

1. **Vertical overflow** → flip above selection when overflow below; clamp if neither fits. CSS cap `max-height: min(60vh, 500px)`. Measure post-mount via `requestAnimationFrame`, re-measure on content updates.
2. **Dismiss model** → remove outside-click handler entirely. Add X button (top-right of tooltip) + Escape key. Both close the bubble.

## Phases

| #  | Phase | File | Status | Effort |
|----|-------|------|--------|--------|
| 01 | Vertical positioning fix (flip above + measure) | [phase-01-vertical-positioning-fix.md](./phase-01-vertical-positioning-fix.md) | completed | 60min |
| 02 | Close button HTML + CSS (4 templates) | [phase-02-close-button-html-css.md](./phase-02-close-button-html-css.md) | completed | 45min |
| 03 | Dismiss handlers (drop outside-click, add Esc + X click) | [phase-03-dismiss-handlers.md](./phase-03-dismiss-handlers.md) | completed | 45min |
| 04 | Unit tests (positioning, close button, Esc, regression) | [phase-04-tests.md](./phase-04-tests.md) | completed | 60min |
| 05 | Manual QA on long page + a11y check | [phase-05-manual-qa.md](./phase-05-manual-qa.md) | pending | 30min |

Total: ~240min (4h).

## Dependency Graph

```
phase-01 ──┐
           ├──> phase-04 (tests cover both) ──> phase-05 (manual QA)
phase-02 ──┴──> phase-03
```

- Phase 01 and Phase 02 can run in parallel (different files).
- Phase 03 depends on Phase 02 (close button must exist before handler binds to it).
- Phase 04 depends on phases 01 + 03 (tests verify final behavior).
- Phase 05 depends on Phase 04 (run only after green tests).

## File Ownership (no overlap across parallel phases)

- Phase 01 owns: `tooltip-positioning.ts`, `tooltip-manager.ts` (positioning hook only), `content-style.css` (max-height block).
- Phase 02 owns: `tooltip-shared-elements.ts`, `tooltip-templates.ts`, `tooltip-error-template.ts`, `content-style.css` (close-btn block).
- Phase 03 owns: `tooltip-event-handlers.ts`, `tooltip-manager.ts` (handler wiring only).
- Phases 01 + 03 both touch `tooltip-manager.ts` → run sequentially after Phase 02, OR coordinate edit regions.

## Constraints

- All files <200 LOC (per `code-standards.md`).
- No new dependencies.
- Conventional commits: `fix(tooltip):` prefix.
- All existing tests must remain passing.

## Validation Log

### Verification Results (2026-05-08)
- Tier: Full (5 phases)
- Claims checked: 22 (file paths, symbols, line numbers, CSS tokens, LOC)
- Verified: 21 | Failed: 1 | Unverified: 0
- Failures:
  - Phase 02 step 5: CSS var `--vocab-primary` does not exist; actual token is `--vocab-primary-500`. Plan's `var(--vocab-primary, #3b82f6)` fallback would always activate. → Fixed via Decision 1.
- Notes: file LOC counts match within ±1; all symbol line numbers match; `.vocab-tooltip-content` confirmed at line 256; dark-mode block confirmed present at line 270.

### Decisions (Validation Session 1)

1. **CSS token for close button** → Use `--vocab-primary-500` (and `--vocab-gray-500/100/900` already correct). Phase 02 step 5 updated.
2. **Dark mode coverage** → ~~Add now~~ **Reverted at implementation**: re-grep found ZERO `prefers-color-scheme` blocks in `content-style.css` (the line-270 `@media` is `(min-width: 1200px)`, responsive sizing). No existing dark-mode CSS anywhere — adding only-`.vocab-close-btn` overrides would be the file's first and only dark-mode rule, violating YAGNI. Defer to a separate stylewide dark-mode follow-up issue.
3. **Overflow strategy on `.vocab-tooltip-content`** → Keep `max-height: min(60vh, 500px)` + `overflow-y: auto` (scrollable). Proceed without pre-audit; if dropdown clipping surfaces during Phase 05 QA, escalate before merge per existing Phase 01 risk mitigation.
4. **Submodule commit boundary** → `vocabulary-extension/` is a submodule. All `fix(tooltip):` commits happen inside the submodule on a feature branch; after merge, parent repo (`vocabulary-project`) bumps the submodule pointer. Issue #4 closes via submodule PR.

### Implementation Outcomes (2026-05-08)

- Branch (submodule): `fix/tooltip-bubble-issue-4`
- Tests: **144 / 144** passing (was 128; +16 new on tooltip behavior)
- Build (`npm run build`): clean, zero TypeScript errors
- File LOC: all <200 (`tooltip-positioning.ts`:148, `tooltip-manager.ts`:130, `tooltip-event-handlers.ts`:194, `tooltip-shared-elements.ts`:172, `tooltip-templates.ts`:172, `tooltip-error-template.ts`:86)
- `setupOutsideClickHandler` symbol fully removed from codebase

#### Code-review fixes applied before merge

- **H1 (high)**: close-button click handler was inert after every `innerHTML` replace (loading → loaded, error reuse-position branch, dropdown re-translate). **Fix**: switched to delegated listener on the tooltip root element (which survives `innerHTML` swaps). Regression tests added (`tooltip-event-handlers.test.ts`: "still fires after innerHTML replace" + "fires when target is a child of the close button").
- **M1 (medium)**: `adjustForViewportVertical` flip branch could land off-screen when selection scrolled below viewport. **Fix**: added upper-bound check (`flippedTop + tooltipHeight <= viewportBottom`). Regression test added (`tooltip-positioning.test.ts`: "clamps when selection is below viewport").
- **M2 (medium, deferred)**: in-tooltip language dropdowns may clip when the new `overflow-y: auto` engages on tall translation results. Aligns with Validation Decision 3 (no pre-audit) — verify in Phase 05 QA; escalate if reproducible.
