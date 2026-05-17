# Phase 01 — Types & Helpers

## Context Links

- Parent plan: [plan.md](plan.md)
- Brainstorm: [../reports/brainstorm-260509-0956-improve-notification.md](../reports/brainstorm-260509-0956-improve-notification.md)
- Code standards: `docs/code-standards.md`
- Related code:
  - `vocabulary-extension/src/types/index.ts:74-89` (`UserSettings`)
  - `vocabulary-extension/src/types/index.ts:2-18` (`Word.pronunciation` line 5, `Word.partOfSpeech` line 7)
  - `vocabulary-extension/src/shared/notification-helpers.ts:6-10` (`WordPreview`)

## Overview

- Date: 2026-05-09
- Priority: P2
- Implementation status: pending
- Review status: not started
- Description: Foundation phase. Add `studyReminderSnoozeUntil` to `UserSettings`, extend `WordPreview` with `pronunciation` + `partOfSpeech`, add three snooze/midnight helpers. No behavior change yet.

## Key Insights

- Brainstorm Decision A: Use Chrome `basic` notification with 3 text slots — needs `pronunciation` + `partOfSpeech` plumbed through `WordPreview` (current shape only has `word/definition/translation`).
- Brainstorm "Storage": persist snooze in existing `settings-storage` Zustand key (no new chrome.storage key, DRY). `useSettingsStore` (`src/shared/store.ts:220-235`) auto-persists any `UserSettings` field.
- Brainstorm "Skip today" = next local midnight. Helper must compute it deterministically (testable).
- Snooze field is `?: number` (timestamp ms). `undefined` = active. Reading uses storage parse path identical to `parseSettings`.

## Requirements

### Functional

- `UserSettings.studyReminderSnoozeUntil?: number` (ms epoch).
- `WordPreview.pronunciation?: string`, `WordPreview.partOfSpeech?: string`.
- `getNextLocalMidnight(now?: number): number` — returns ms epoch of next local 00:00:00.000.
- `getStudyReminderSnoozeUntil(): Promise<number | undefined>` — reads from `settings-storage` via existing `parseSettings`.
- `setStudyReminderSnoozeUntil(ts: number | undefined): Promise<void>` — read-modify-write `settings-storage` JSON, preserve `state` envelope, persist via `chrome.storage.local.set`.

### Non-functional

- TypeScript strict, no `any`.
- File size ≤ 200 LOC (`notification-helpers.ts` currently 147; new file ~+40 LOC fine).
- No new dependencies.
- DRY: reuse `parseSettings` for read.

## Architecture

```
+-------------------+
| UserSettings      |  +-- studyReminderSnoozeUntil?: number
+-------------------+

+-------------------+
| WordPreview       |  +-- pronunciation?: string
|                   |  +-- partOfSpeech?: string
+-------------------+

notification-helpers.ts
   getNextLocalMidnight(now)        pure
   getStudyReminderSnoozeUntil()    chrome.storage.local.get -> parseSettings
   setStudyReminderSnoozeUntil(ts)  read-modify-write same key
```

Data flow (write): caller -> `setStudyReminderSnoozeUntil` -> get raw `settings-storage` -> JSON.parse -> mutate `state.settings.studyReminderSnoozeUntil` -> JSON.stringify -> `chrome.storage.local.set`. Zustand store hydrates from same key on next read; `chrome.storage.onChanged` listener (`store.ts:239`) keeps in-memory mirror in sync.

## Related Code Files

### Modify

- `vocabulary-extension/src/types/index.ts` — add field to `UserSettings`.
- `vocabulary-extension/src/shared/notification-helpers.ts` — extend `WordPreview`; add 3 helpers; map `Word.pronunciation` + `Word.partOfSpeech` into preview in `getRandomWordPreview`.

### Create

- None.

### Delete

- None.

## Implementation Steps

1. **Types (`src/types/index.ts`):** Append `studyReminderSnoozeUntil?: number // ms epoch; undefined = active reminders` to `UserSettings` interface (after `highlightColor`, line 88).
2. **WordPreview (`src/shared/notification-helpers.ts`):** Add `pronunciation?: string` and `partOfSpeech?: string` to `WordPreview` interface (lines 6-10).
3. **Update `NotificationData.words` shape:** Extend tuple type at line 19 to include `pronunciation?: string; partOfSpeech?: string` so `getRandomWordPreview` can read them.
4. **Update `getRandomWordPreview`:** When building return object (lines 96-103, 107-112), include `pronunciation: wordData.pronunciation` and `partOfSpeech: wordData.partOfSpeech`.
5. **Update `parseVocabData`:** No code change needed (passes `data?.state?.words` straight through), but TS type for return `words` must include new optional fields — update `NotificationData['words']` shape at line 19.
6. **Add `getNextLocalMidnight(now?: number): number`:**
   ```ts
   export function getNextLocalMidnight(now: number = Date.now()): number {
     const d = new Date(now)
     d.setHours(24, 0, 0, 0)  // sets next-day 00:00:00.000 in local TZ
     return d.getTime()
   }
   ```
7. **Add `getStudyReminderSnoozeUntil`:** read `settings-storage` via `chrome.storage.local.get`, run through `parseSettings`, return `settings?.studyReminderSnoozeUntil`. Cast settings type to include new field (extend `NotificationData['settings']` shape at lines 13-16 to add `studyReminderSnoozeUntil?: number`).
8. **Add `setStudyReminderSnoozeUntil(ts: number | undefined)`:**
   - `chrome.storage.local.get(['settings-storage'])`
   - Parse current value; if missing, build minimal `{state: {settings: {}}}` envelope.
   - Set `parsed.state.settings.studyReminderSnoozeUntil = ts` (delete key when `ts === undefined` to keep storage tidy).
   - `chrome.storage.local.set({'settings-storage': JSON.stringify(parsed)})`.
   - Wrap in try/catch; log via `console.warn` on failure.
9. **Type check:** `cd vocabulary-extension && npx tsc --noEmit`.

## Todo List

- [ ] Add `studyReminderSnoozeUntil?: number` to `UserSettings`
- [ ] Extend `WordPreview` with `pronunciation?` + `partOfSpeech?`
- [ ] Extend `NotificationData['words']` element shape with same fields
- [ ] Extend `NotificationData['settings']` shape with `studyReminderSnoozeUntil?`
- [ ] Map `pronunciation` + `partOfSpeech` in `getRandomWordPreview` returns (both branches)
- [ ] Add `getNextLocalMidnight(now?: number): number`
- [ ] Add `getStudyReminderSnoozeUntil(): Promise<number | undefined>`
- [ ] Add `setStudyReminderSnoozeUntil(ts: number | undefined): Promise<void>`
- [ ] Run `npx tsc --noEmit` clean
- [ ] Run `npm run lint` clean

## Success Criteria

- Type-check passes; lint clean.
- Existing tests still green (no behavior changes to existing helpers).
- New helpers exported; signatures match brainstorm spec.

Validation: `cd vocabulary-extension && npx tsc --noEmit && npm run lint && npm run test:unit`.

## Risk Assessment

| Risk | Severity | Likelihood | Mitigation |
|------|----------|------------|------------|
| `setHours(24,...)` DST edge: spring-forward day | Low | Low | Date constructor handles DST; document, accept ~23h/25h drift on DST days (matches user expectation of "next local midnight"). |
| Read-modify-write race vs Zustand persist | Med | Low | Snooze button click rare (≤1/day); RMW window tiny. If race ever observed, bounce snooze write through Zustand store instead. Out-of-scope for v1. |
| Existing `getRandomWordPreview` callers depend on narrow `WordPreview` | Low | Low | Adding optional fields is backward-compatible. Verified callers: `notifications.ts:74-89`, `notification-handler.ts:53-73`. Both still work. |
| `notification-handler.ts` builds `WordPreview` inline (DRY violation, lines 53-73) | Low | High | Out-of-scope. Note for follow-up: could call `getRandomWordPreview` instead. Brainstorm explicitly keeps current path. |

## Security Considerations

- No PII in snooze timestamp — pure ms number.
- Storage already gated by Chrome's local storage encryption.
- No new permissions needed.
- Validate `ts` is finite number before write (defensive; optional).

## Next Steps

- Unblocks phase-02 (uses extended `WordPreview` + new helpers).
- Unblocks phase-03 (UI uses `getStudyReminderSnoozeUntil` + `setStudyReminderSnoozeUntil(undefined)` for Resume).
- Helper unit tests can start in parallel under phase-04.
