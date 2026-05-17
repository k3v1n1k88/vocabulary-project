---
title: "Improve notification (Issue #6)"
description: "Rich notification content + study-reminder snooze (1h / Skip today)"
status: completed
priority: P2
effort: 5h
branch: feat/improve-notification-issue-6
tags: [notification, ux, chrome-extension, issue-6]
created: 2026-05-09
completed: 2026-05-09
---

# Improve Notification (Issue #6)

## Context

- Issue: https://github.com/k3v1n1k88/vocabulary-extension/issues/6
- Brainstorm: [../reports/brainstorm-260509-0956-improve-notification.md](../reports/brainstorm-260509-0956-improve-notification.md)
- Submodule (code root): `vocabulary-extension/`

## Goal

1. Pack rich content (word + IPA + PoS + definition + translation + due count) into Chrome `basic` notification with 2 buttons.
2. Add snooze for study reminders only (1h or until next local midnight). Settings UI shows status + Resume.

## Phases

| # | Phase | Status | Effort | Description |
|---|-------|--------|--------|-------------|
| 01 | [Types & helpers](phase-01-types-and-helpers.md) | completed | 1h | Extend `UserSettings`, `WordPreview`; add snooze get/set + midnight helper |
| 02 | [Notifications + snooze gate](phase-02-rewrite-notifications-and-snooze.md) | completed | 1.5h | Rewrite `showDailyReminder`; add snooze gate in alarm; button-click handler |
| 03 | [Settings UI snooze banner](phase-03-settings-ui-snooze-banner.md) | completed | 1h | Banner + Resume button in `learning-settings.tsx` |
| 04 | [Tests](phase-04-tests.md) | completed | 1.5h | Helper unit tests + snooze gate tests + button handler tests |

## Dependencies

```
phase-01 (foundation: types + helpers)
   ├── phase-02 (notifications.ts uses helpers + new field)
   ├── phase-03 (UI uses helpers + new field, parallel with 02)
   └── phase-04 (tests cover all impl phases; helper tests can start with 01)
```

- phase-02 and phase-03 can run in parallel after phase-01.
- phase-04 final pass requires phase-02 and phase-03 done.

## Success Criteria

1. Notification renders title + message + contextMessage + 2 buttons; degrades gracefully when IPA/PoS/translation missing.
2. `Snooze 1h` and `Skip today` set `studyReminderSnoozeUntil`; `handleStudyReminderAlarm` early-returns while active.
3. `due-cards`, `word-saved`, `streak-risk` notifications NOT affected by snooze.
4. `TEST_NOTIFICATION` always fires (bypasses snooze — gate is in alarm handler, not in `showDailyReminder`).
5. Settings UI shows snooze banner with localized timestamp + working Resume.
6. `npx tsc --noEmit` passes; `npm run lint` clean; `npm run test:unit` green.

## Out of Scope

- Localization of button copy.
- Audio in notification.
- Per-notification-type snooze.
- Side panel surface.
- Refactoring alarm scheduling.

## Unresolved Questions

_All resolved in Validation Session 1 (see Validation Log below)._

## Validation Log

### Session 1 — 2026-05-09
**Trigger:** `/ck:plan validate` before implementation kickoff.
**Questions asked:** 4
**Verification tier:** Standard (4 phases) — file paths and line refs all verified clean.

#### Questions & Answers

1. **[Assumption]** Plan keeps button copy hardcoded English ('Snooze 1h' / 'Skip today'). UI is currently English-only. Confirm?
   - Options: English-only (Recommended) | Add i18n now | Icons/symbols only
   - **Answer:** English-only
   - **Rationale:** No i18n infra exists; adding it is a separate epic. Hardcoded English matches all current UI strings.

2. **[Scope]** Should `due-cards-check` aggregate notification (5+ cards due) respect study-reminder snooze?
   - Options: Independent — snooze study only (Recommended) | Snooze affects both
   - **Answer:** Independent — snooze study only
   - **Rationale:** Snooze targets the per-interval reminder. Backlog alert is a distinct intent. Field name `studyReminderSnoozeUntil` correctly scopes the semantic.

3. **[Architecture]** `notifications.ts` is already 230 LOC (>200 cap). Where should `buildReminderContent` live?
   - Options: Append to notification-helpers.ts (Recommended) | New notification-content-builder.ts
   - **Answer:** New notification-content-builder.ts
   - **Rationale:** User chose strict file-size separation. Pure content building isolated from storage helpers. Phase-02 must CREATE this file (not conditional).

4. **[Risk]** DST edge: 'Skip today' on spring-forward = 23h, on fall-back = 25h. Accept or clamp?
   - Options: Accept native behavior (Recommended) | Clamp to 24h max
   - **Answer:** Accept native behavior
   - **Rationale:** `setHours(24,0,0,0)` semantic = "next local midnight" — DST drift matches user expectation.

#### Confirmed Decisions
- Button copy: hardcoded English.
- Snooze scope: study-reminder alarm only; due-cards-check independent.
- `buildReminderContent`: new file `src/shared/notification-content-builder.ts` (mandatory, not conditional).
- DST: native `setHours(24,0,0,0)`.

#### Action Items
- [x] Plan-02 architecture updated: `notification-content-builder.ts` is now in **Create** list (not conditional Create).
- [ ] Phase-02 imports `buildReminderContent` from `./notification-content-builder` instead of `./notification-helpers`.

#### Impact on Phases
- **Phase 02:** File-creation list changes from "conditional Create" → "Create" for `notification-content-builder.ts`. Import path updated.
- **Phase 04:** `buildReminderContent` test moves from `notification-helpers.test.ts` to new `notification-content-builder.test.ts`.
