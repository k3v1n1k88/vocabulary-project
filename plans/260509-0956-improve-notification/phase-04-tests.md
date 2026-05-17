# Phase 04 — Tests

## Context Links

- Parent plan: [plan.md](plan.md)
- Brainstorm: [../reports/brainstorm-260509-0956-improve-notification.md](../reports/brainstorm-260509-0956-improve-notification.md)
- Depends on: [phase-01-types-and-helpers.md](phase-01-types-and-helpers.md), [phase-02-rewrite-notifications-and-snooze.md](phase-02-rewrite-notifications-and-snooze.md)
- Independent of: [phase-03-settings-ui-snooze-banner.md](phase-03-settings-ui-snooze-banner.md) (UI smoke covered in phase-03; banner not unit-tested here per YAGNI)
- Code under test:
  - `vocabulary-extension/src/shared/notification-helpers.ts`
  - `vocabulary-extension/src/shared/notifications.ts`
  - `vocabulary-extension/src/background/handlers/notification-handler.ts`
- Test infra: Vitest + jsdom + `vitest-chrome`. Existing test file: `vocabulary-extension/src/shared/notification-helpers.test.ts` (175 LOC).

## Overview

- Date: 2026-05-09
- Priority: P2
- Implementation status: pending
- Review status: not started
- Description: Extend existing `notification-helpers.test.ts` (and add `notifications.test.ts` if needed for alarm/button-click coverage) covering: extended `WordPreview`, `getNextLocalMidnight`, snooze get/set round-trip, alarm gate, button-click dispatch, TEST_NOTIFICATION bypass. Maintain ≥80% on critical paths.

## Key Insights

- Helper-level tests stay in `notification-helpers.test.ts` (collocated convention, line 1-9 already imports from `./notification-helpers`).
- Alarm + button-click handlers live in `notifications.ts` and need a new file `notifications.test.ts` (does not exist yet) — pattern follows `translation-service.test.ts` style.
- `vitest-chrome` provides `chrome.storage.local`, `chrome.notifications`, `chrome.alarms` mocks. `vi.stubGlobal` for fetch where needed (not needed here).
- Fake timers: `vi.useFakeTimers()` + `vi.setSystemTime(new Date('2026-05-09T15:30:00'))` for deterministic midnight + 1h math. Always `vi.useRealTimers()` in `afterEach`.
- `setStudyReminderSnoozeUntil` does read-modify-write — mock `chrome.storage.local.get` to return seed + assert `chrome.storage.local.set` payload via `expect(...).toHaveBeenCalledWith(...)`.
- `TEST_NOTIFICATION` bypass test: assert `chrome.notifications.create` IS called when `showDailyReminder` invoked directly with non-empty snooze in storage — proves gate is in alarm path, not in `showDailyReminder`.

## Requirements

### Functional

Cover seven test groups:

1. `getRandomWordPreview` — new fields `pronunciation` + `partOfSpeech` propagate from `Word` → `WordPreview` (both due-card branch + random branch).
2. `getNextLocalMidnight` — pure fn; deterministic via fake timers; spans DST day acceptable (1 case mid-year, 1 case crossing month boundary).
3. `getStudyReminderSnoozeUntil` / `setStudyReminderSnoozeUntil` — round-trip via mocked storage; covers undefined-clears-key, missing envelope auto-creates, malformed JSON returns undefined safely.
4. `handleStudyReminderAlarm` — when snooze in future: `chrome.notifications.create` NOT called. When snooze expired: notification fires AND snooze auto-cleared.
5. `setupNotificationClickHandler` — button index 0 → snooze = now + 3_600_000; button index 1 → snooze = next local midnight; non-`daily-reminder-` IDs preserve legacy `openPopup` behavior; body click unchanged.
6. `TEST_NOTIFICATION` bypass — direct `showDailyReminder` call creates notification even with active snooze in storage. (Indirectly: confirm `handleTestNotification` calls `showDailyReminder` not the alarm path.)
7. Regression — all existing tests in `notification-helpers.test.ts` still pass unchanged.

### Non-functional

- TS strict, no `any` (use `as unknown as Type` if mock cast needed).
- Coverage target ≥80% on `notification-helpers.ts` and `notifications.ts` snooze/button paths (per `docs/code-standards.md` line 201).
- Test file size ≤ 200 LOC each. If `notifications.test.ts` projects > 200, split into `notifications.snooze.test.ts` + `notifications.click-handler.test.ts`.
- No real network/IO; all chrome APIs mocked.
- Deterministic: every time-dependent test uses `vi.setSystemTime`.

## Architecture

```
notification-helpers.test.ts  (extend existing)
  +-- describe getRandomWordPreview            <- pronunciation + partOfSpeech
  +-- describe getNextLocalMidnight            <- vi.setSystemTime
  +-- describe getStudyReminderSnoozeUntil     <- mock chrome.storage.local.get
  +-- describe setStudyReminderSnoozeUntil     <- mock get + set; assert payload
  +-- describe buildReminderContent (phase-02) <- pure matrix: word/IPA/PoS/translation present|missing
  +-- (existing 5 describes preserved)

notifications.test.ts  (new)
  +-- describe handleStudyReminderAlarm
  |     +-- skips when snoozeUntil > now
  |     +-- fires + auto-clears when snoozeUntil <= now
  |     +-- skips when notificationsEnabled false (regression)
  +-- describe setupNotificationClickHandler
  |     +-- daily-reminder- + idx 0 -> snooze = now + 1h
  |     +-- daily-reminder- + idx 1 -> snooze = next midnight
  |     +-- non daily-reminder- id -> openPopup (legacy)
  |     +-- onClicked body -> openPopup (legacy)
  +-- describe showDailyReminder bypasses snooze
        +-- direct call creates notification regardless of stored snooze
        +-- handleTestNotification calls showDailyReminder (spy)
```

Mock seed pattern:

```ts
const seedSettings = (s: Partial<UserSettings>) => {
  const value = JSON.stringify({ state: { settings: s } })
  vi.mocked(chrome.storage.local.get).mockResolvedValue({ 'settings-storage': value })
}
```

## Related Code Files

### Modify

- `vocabulary-extension/src/shared/notification-helpers.test.ts`:
  - Add tests for new `pronunciation` + `partOfSpeech` in `getRandomWordPreview`.
  - Add `describe getNextLocalMidnight`.
  - Add `describe getStudyReminderSnoozeUntil`.
  - Add `describe setStudyReminderSnoozeUntil`.

### Create

- `vocabulary-extension/src/shared/notification-content-builder.test.ts` (new): <!-- Updated: Validation Session 1 - buildReminderContent moved to its own file -->
  - Covers `buildReminderContent` matrix: word/IPA/PoS/translation present|missing × dueCount/streak fallbacks.
- `vocabulary-extension/src/shared/notifications.test.ts` (new):
  - Covers `handleStudyReminderAlarm`, `setupNotificationClickHandler`, `showDailyReminder` bypass.
  - Pattern: import functions; mock `chrome.*` via `vitest-chrome` autoload (already in `vitest.setup.ts`).
  - If `handleStudyReminderAlarm` is not exported — add `export` (small refactor inside phase-02 file scope; or test via alarm listener trigger). Prefer: export named for testability; document "test-only export".

### Delete

- None.

## Implementation Steps

1. **Extend `getRandomWordPreview` tests** (`notification-helpers.test.ts`):
   - Update fixture `words` (line 129-133): add `pronunciation: '/ˈæpəl/'`, `partOfSpeech: 'noun'` on at least 2 entries.
   - Add: `it('includes pronunciation when present', ...)` — assert `result?.pronunciation` matches.
   - Add: `it('includes partOfSpeech when present', ...)` — assert `result?.partOfSpeech` matches.
   - Add: `it('omits pronunciation when missing on word', ...)` — pick word w/o IPA, assert `result?.pronunciation` undefined.

2. **Add `describe getNextLocalMidnight`**:
   ```ts
   describe('getNextLocalMidnight', () => {
     beforeEach(() => vi.useFakeTimers())
     afterEach(() => vi.useRealTimers())

     it('returns next local 00:00 from mid-day', () => {
       vi.setSystemTime(new Date(2026, 4, 9, 15, 30, 0))  // 2026-05-09 15:30 local
       const ts = getNextLocalMidnight()
       const d = new Date(ts)
       expect(d.getFullYear()).toBe(2026)
       expect(d.getMonth()).toBe(4)
       expect(d.getDate()).toBe(10)
       expect(d.getHours()).toBe(0)
       expect(d.getMinutes()).toBe(0)
     })

     it('rolls over month boundary', () => {
       vi.setSystemTime(new Date(2026, 4, 31, 23, 59, 0))  // last day of May
       const d = new Date(getNextLocalMidnight())
       expect(d.getMonth()).toBe(5)  // June
       expect(d.getDate()).toBe(1)
     })

     it('accepts explicit `now` arg', () => {
       const explicit = new Date(2026, 0, 1, 12, 0, 0).getTime()
       const d = new Date(getNextLocalMidnight(explicit))
       expect(d.getDate()).toBe(2)
     })
   })
   ```

3. **Add `describe getStudyReminderSnoozeUntil`**:
   - Mock `chrome.storage.local.get` to return seeded `settings-storage`.
   - Cases:
     - returns undefined when `settings-storage` missing.
     - returns undefined when settings has no `studyReminderSnoozeUntil`.
     - returns the stored number when present.
     - returns undefined on malformed JSON (defensive).

4. **Add `describe setStudyReminderSnoozeUntil`**:
   - Mock `chrome.storage.local.get` + `chrome.storage.local.set`.
   - Cases:
     - writes ts to `state.settings.studyReminderSnoozeUntil`; preserves other settings keys (seed `notificationsEnabled: true`, assert preserved).
     - `set(undefined)` → key removed (not present in resulting JSON).
     - missing envelope → builds `{state:{settings:{studyReminderSnoozeUntil:ts}}}`.
     - assert `chrome.storage.local.set` called with `{'settings-storage': expect.any(String)}` and parsed payload matches.

5. **Add `describe buildReminderContent`** (helper from phase-02):
   - Matrix:
     - full word (ipa + pos + translation, due=3, streak=5) → `title='📖 example /ɪɡˈzæmpəl/'`, `message='noun. ...'`, `contextMessage='ví dụ · +2 more cards waiting'`.
     - word without IPA → title `'📖 example'`, no IPA.
     - word without PoS → message is plain definition.
     - word without translation, due=1 → contextMessage undefined (no translation, no "+0").
     - word without translation, due=5 → contextMessage `'+4 more cards waiting'`.
     - no word, due=3 → title `'Time to Study!'`, message `'You have 3 cards waiting for review!'`.
     - no word, due=0, streak=4 → message `'Keep your 4-day streak going!'`.
     - no word, due=0, streak=0 → generic message.
     - definition >100 chars → truncated to 100.
     - very long IPA pushing title >50 chars → IPA dropped.

6. **Create `notifications.test.ts`**:

   ```ts
   import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest'
   import { showDailyReminder, /* exported handlers */ } from './notifications'
   ```

   - **`describe handleStudyReminderAlarm`:**
     - seed snooze = `Date.now() + 60_000` → invoke handler → `expect(chrome.notifications.create).not.toHaveBeenCalled()`.
     - seed snooze = `Date.now() - 60_000` → invoke → `expect(chrome.notifications.create).toHaveBeenCalled()` AND `chrome.storage.local.set` called with payload where `studyReminderSnoozeUntil` removed.
     - seed `notificationsEnabled: false` → no fire (regression).

   - **`describe setupNotificationClickHandler`:**
     - call `setupNotificationClickHandler()`; capture listener via `chrome.notifications.onButtonClicked.addListener.mock.calls[0][0]`.
     - invoke with `('daily-reminder-123', 0)` → assert `chrome.storage.local.set` payload sets snooze to `Date.now() + 3_600_000` (use fake timers).
     - invoke with `('daily-reminder-123', 1)` → assert payload = `getNextLocalMidnight()` value.
     - invoke with `('other-id', 0)` → assert `chrome.action.openPopup` called; storage NOT set.
     - body click listener via `chrome.notifications.onClicked.addListener.mock.calls[0][0]` → call with any id → `chrome.action.openPopup` called (unchanged).

   - **`describe showDailyReminder bypasses snooze`:**
     - seed snooze = `Date.now() + 3_600_000` → call `showDailyReminder(2, 5, {word:'apple', definition:'fruit'})` directly → `chrome.notifications.create` IS called. Proves gate not in `showDailyReminder`.
     - spy on `showDailyReminder`; invoke `handleTestNotification` from `notification-handler.ts` → spy called once. (Optional: this is integration; can defer if isolation tricky.)

7. **Test-only export** (if needed): in `notifications.ts`, export `handleStudyReminderAlarm` and `handleDueCardsCheckAlarm` (currently file-private). Add comment `// exported for testability`. KISS — no internal-only mechanism.

8. **Run**: `cd vocabulary-extension && npm run test:unit`. Iterate until green.

9. **Coverage check** (optional): `npm run test:unit -- --coverage` → confirm `notification-helpers.ts` ≥85%, `notifications.ts` snooze paths ≥80%.

10. **Lint + typecheck**: `npx tsc --noEmit && npm run lint`.

## Todo List

- [ ] Extend `getRandomWordPreview` fixture with `pronunciation` + `partOfSpeech`
- [ ] Assert new fields propagate (3 cases)
- [ ] Add `describe getNextLocalMidnight` (3 cases, fake timers)
- [ ] Add `describe getStudyReminderSnoozeUntil` (4 cases)
- [ ] Add `describe setStudyReminderSnoozeUntil` (4 cases)
- [ ] Add `describe buildReminderContent` (10-case matrix)
- [ ] Create `notifications.test.ts`
- [ ] Add alarm-gate tests (3 cases)
- [ ] Add button-click handler tests (4 cases)
- [ ] Add `showDailyReminder` bypass test
- [ ] Export `handleStudyReminderAlarm` if needed for tests
- [ ] Run `npm run test:unit` green
- [ ] Run `npx tsc --noEmit` clean
- [ ] Run `npm run lint` clean
- [ ] Coverage ≥80% on touched files

## Success Criteria

- All new tests pass; all existing tests still pass.
- `npm run test:unit` exits 0.
- `npx tsc --noEmit` clean.
- `npm run lint` clean.
- Coverage on `notification-helpers.ts` ≥85%; `notifications.ts` snooze + button paths ≥80%.
- No real chrome API calls (all mocked); no real timers (fake timers in time-sensitive tests).
- Test files ≤200 LOC each; split if exceeded.

Validation: full test suite green in CI; manual review of test output for correct gate/bypass behaviors.

## Risk Assessment

| Risk | Severity | Likelihood | Mitigation |
|------|----------|------------|------------|
| `vitest-chrome` mock for `chrome.storage.local.get/set` returns shape mismatch | Med | Med | Inspect `vitest.setup.ts`; fall back to `vi.spyOn(chrome.storage.local, 'get').mockResolvedValue(...)`. |
| Top-level listener registration fires once per import → cross-test pollution | High | Med | `beforeEach: vi.clearAllMocks()`; capture listener once via `addListener.mock.calls[0][0]`. If duplication: dynamic re-import via `vi.resetModules()`. |
| `handleStudyReminderAlarm` private → must export for test (touches phase-02 file) | Low | High | Add export with comment; coordinate with phase-02 author or make export part of phase-02 deliverable. |
| Fake timers leak into other tests | Med | Low | Strict `afterEach(vi.useRealTimers)` in every describe using timers. |
| `Date.now()` vs `vi.setSystemTime` drift inside RMW | Low | Low | Both honor fake timers when `useFakeTimers()` is active. |
| Tests for DOM-free helpers in jsdom env unnecessarily slow | Low | Low | Acceptable; suite already uses jsdom. |
| `chrome.action.openPopup` not stubbed by `vitest-chrome` | Med | Med | Stub manually: `chrome.action = { openPopup: vi.fn() } as any`. Document in setup. |

## Security Considerations

- Tests only — no code change to production paths beyond adding `export`.
- No real API keys or PII in fixtures (use placeholder words like `apple`, `example`).
- Mock data must not leak between tests (`beforeEach: vi.clearAllMocks()`).
- Avoid logging full storage payloads in test output (potential CI log noise).

## Next Steps

- Once green: phase complete; bundle into PR with phase-01/02/03 commits.
- Post-merge: run Playwright E2E (existing `playwright.config.ts`) to confirm no popup regression.
- Update `docs/project-changelog.md` per docs-management rule (separate task).
- If coverage gaps detected for snooze gate post-merge → add integration test against built service worker (out-of-scope for v1).
