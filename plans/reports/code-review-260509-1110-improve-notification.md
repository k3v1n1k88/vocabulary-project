# Code Review — Issue #6 (improve notification + snooze)

- Branch: `feat/improve-notification-issue-6` (submodule `vocabulary-extension/`)
- Diff base: `master` @ `c5423b4`
- Tests: 183/183 pass · `tsc --noEmit` clean · `lint` clean · `build` ok (verified locally; ran `npm test`).

## Scope
- Production: `src/types/index.ts`, `src/shared/notification-helpers.ts`, `src/shared/notification-content-builder.ts` (new), `src/shared/notifications.ts`, `src/options/components/learning-settings.tsx`, `vitest.setup.ts`.
- Tests: `notification-helpers.test.ts` (extended), `notification-content-builder.test.ts` (new), `notifications.test.ts` (new).
- LOC: notifications.ts 244, notification-helpers.ts 205, content-builder 61, learning-settings.tsx 191.

## Overall Assessment
Solid implementation. Pure-content builder is well factored, snooze gate placement is correct, fallbacks are sensible, and tests cover the gate (positive + negative), button dispatch, malformed JSON, and non-finite write rejection. **One real bug** (duplicate listener registration on install) and a few smaller concerns described below — none block merge if owner can patch the install path.

---

## Critical (blocking)

### C1 — Duplicate listener registration on install (`background/service-worker.ts:38, 42`)
`initNotifications()` is invoked twice when the extension is installed/updated:
- Top-level (line 42) — runs on every SW startup.
- `chrome.runtime.onInstalled` callback (line 38) — fires after top-level on first install / update.

Each call runs `setupAlarmHandler()` (`notifications.ts:184`) and `setupNotificationClickHandler()` (`notifications.ts:202`), each of which performs `addListener(...)` unconditionally. Result: on the install/update tick the alarm + button-click listeners get registered **twice in the same SW instance**, so:
- Tapping `Snooze 1h` calls `setStudyReminderSnoozeUntil` twice (RMW collision; second write may overwrite if storage write order interleaves),
- `chrome.notifications.clear` runs twice (idempotent — fine),
- `chrome.action.openPopup()` may be called twice on body click,
- An alarm tick processes twice (extra notification creation possible if `handleStudyReminderAlarm` runs concurrently).

This existed before this PR for `setupAlarmHandler`, but the new button handler now *writes* to storage on click, so the duplicated registration is more harmful than before.

**Recommended fix (one of):**
- Add a module-level `let initialized = false` in `notifications.ts` and early-return from `setupAlarmHandler` / `setupNotificationClickHandler` when set, OR
- Drop the `initNotifications()` call inside `onInstalled` — top-level registration is sufficient for MV3, the install hook only needs to (re)schedule alarms (which `scheduleStudyReminder` already clears + creates idempotently).

Preferred: drop the duplicate call. `onInstalled` should only do things that are install-specific (context menu, default settings, alarm scheduling). Listener wiring belongs at top level only.

---

## High Priority

### H1 — `setStudyReminderSnoozeUntil` RMW races Zustand persist (notification-helpers.ts:182-205, store.ts:239)
The function does `get → mutate → JSON.stringify → set` on `settings-storage`, which is also the persist key for `useSettingsStore`. Concurrent paths:
1. SW button handler calls `setStudyReminderSnoozeUntil(...)` → writes the whole envelope.
2. Options page worker concurrently calls `useSettingsStore.getState().updateSettings({ ... })` → Zustand persist serialises the **in-memory** settings (which may not yet contain the new snooze) and writes the whole envelope.
3. The `chrome.storage.onChanged` listener (`store.ts:239`) hears the first write, calls `setState({ settings: { ...defaultSettings, ...parsed.state.settings } })`, but this is **after** Zustand may have already enqueued its write.

Last-write-wins on the storage key. The most likely user-visible symptom: clicking Snooze in the system notification while the Options page is open and the user has just toggled `notificationsEnabled` — one of the two writes loses.

The `Resume now` button in `learning-settings.tsx:90` calls `onSettingsUpdate({ studyReminderSnoozeUntil: undefined })` which goes through Zustand and avoids the race in that direction, so the in-page resume is safe. The hazard is only **SW-button-click vs Options-page-Zustand-write** happening within the same ~ms window.

**Mitigations** (pick at least one — current implementation has none):
- Have the SW button handler send a `chrome.runtime.sendMessage` to options page when open, OR
- Read-modify-write under a small advisory lock keyed in storage (heavy), OR
- Document in code-comment that this is best-effort and accept the rare loss (cheapest; user can re-snooze).

This is not a data-corruption bug — both writers merge under `state.settings` — but the *snooze key* itself can flicker. Acceptable for the feature, but please add a comment near `setStudyReminderSnoozeUntil` acknowledging the race so future maintainers don't assume atomicity.

### H2 — Potentially unbounded `contextMessage` from API translation (notification-content-builder.ts:42-50)
`randomWord.translation` is sourced from dictionary/LLM API and is not length-capped before being joined into `contextMessage`. Title and message are capped (50 / 100 chars), but `contextMessage` is not. A long Vietnamese gloss would either be silently truncated by Chrome's notification renderer or look ugly. Suggest capping translation to ~80 chars with `…` ellipsis to match the rest of the formatting discipline.

Not a security issue — `chrome.notifications` does not parse HTML — but it is a UX consistency gap.

---

## Medium Priority

### M1 — `notifications.ts` is 244 LOC (44 over the project's 200-LOC soft cap)
Project rules require modularising files > 200 LOC. Natural split: extract the two `chrome.*.onAlarm/onClicked/onButtonClicked` listener wiring functions (`setupAlarmHandler`, `setupNotificationClickHandler`, `initNotifications`) into `notification-listeners.ts`, leaving `notifications.ts` for `show*` + `schedule*` + `handle*Alarm` only. Acceptable to defer if owner prefers, but please add a note.

`notification-helpers.ts` at 205 LOC is borderline — could leave as-is.

### M2 — `setStudyReminderSnoozeUntil` does a single `JSON.parse → mutate → stringify` without preserving non-`state.settings` keys (notification-helpers.ts:188-200)
The function only walks `parsed.state.settings`. If Zustand ever persists additional top-level keys alongside `state` (e.g., `version` from `persist` middleware), they survive because we only mutate the leaf. Verified: `parsed: { state?: { settings?: ... } } = {}` and `parsed.state = parsed.state || {}` preserve other keys via spread. ✅ Correct as written.

However the test asserting "preserves other settings keys" (`notification-helpers.test.ts:284-298`) does NOT cover the case where Zustand has written `version: 0` at the envelope root. Suggest extending the test:
```ts
await chrome.storage.local.set({
  'settings-storage': JSON.stringify({
    version: 0,
    state: { settings: { dailyGoal: 20 } }
  })
})
await setStudyReminderSnoozeUntil(123)
const parsed = JSON.parse((await chrome.storage.local.get(['settings-storage']))['settings-storage'])
expect(parsed.version).toBe(0)
```

### M3 — `setupNotificationClickHandler` test pollution risk (notifications.test.ts:96-110)
Each `captureButtonHandler()` call invokes `setupNotificationClickHandler()` which calls `addListener` — the mock's call-list grows across tests. Current code uses `vi.mocked(...).mock.calls[calls.length - 1]?.[0]` to grab the **latest** registered listener, which works because `vi.clearAllMocks()` runs in `vitest.setup.ts:beforeEach`. Verified: clearAllMocks fires in the global hook before each `it`. ✅ Safe.

But the hook order matters: if anyone changes `vitest.setup.ts` to drop `clearAllMocks`, these tests will silently break (older listeners would get picked). Worth a one-line comment in the test file noting the dependency.

### M4 — Banner does not auto-refresh (`learning-settings.tsx:19`)
Comment correctly notes this is intentional (YAGNI). But user could leave the Options page open past a snooze expiry and still see the banner showing "paused until …" forever. Acceptable trade-off given the explicit `mountedAt` snapshot and the inline comment, but worth surfacing in the journal/changelog as a known UX limitation.

---

## Low Priority

### L1 — Magic strings duplicated
- `'settings-storage'` appears in `notification-helpers.ts`, `store.ts`, `notification-handler.ts`. Already pre-existing; not in scope, but extracting a const from the persist key would prevent future drift.
- `REMINDER_ID_PREFIX = 'daily-reminder-'` is private to `notifications.ts` — fine.

### L2 — `showDailyReminder` permissions check (notifications.ts:73)
`chrome.permissions.contains({ permissions: ['notifications'] })` — `notifications` is declared as a required permission in `manifest.json` (verify), so this check always returns true unless the user manually revoked. Cheap call though, leave as defensive guard. ✅

### L3 — Title length cap unicode-unsafe (notification-content-builder.ts:23)
`withIpa.length <= TITLE_MAX_LEN` counts UTF-16 code units; emoji `📖` is 2 units; IPA characters are 1. Resulting display width is unpredictable across OSes anyway, so not worth fixing.

### L4 — `definition.slice(0, 100)` produces no ellipsis (content-builder.ts:34)
A 101-char definition just truncates at 100 with no `…`. Cheap UX win to append `'…'` when truncated. Optional.

### L5 — Test `'fires and auto-clears stale snooze'` (notifications.test.ts:54-70)
Verifies that `'studyReminderSnoozeUntil' in parsed.state.settings === false`. Good. But the implementation (`notifications.ts:154`) only auto-clears **before** `showDailyReminder` is invoked. If the storage write fails (caught in `setStudyReminderSnoozeUntil`), the notification still fires — meaning a user could enter a state where the stale snooze is still in storage but they have already seen the reminder. Test does not cover the storage-failure branch. Low priority because the catch-and-warn pattern is consistent with the rest of the file.

---

## Edge Cases (scout)

- **`reminderInterval` undefined → 60-min default** (notifications.ts:39): correct.
- **Concurrent SW-restart during snooze**: `setStudyReminderSnoozeUntil` writes are atomic per-call (single `chrome.storage.local.set`), so SW restart cannot tear a write. ✅
- **`notificationsEnabled` toggled off mid-snooze**: `handleStudyReminderAlarm` early-returns on `!notificationsEnabled` regardless of snooze, so behavior is correct. ✅
- **Snooze written via Resume button while SW alarm is firing**: in-flight `handleStudyReminderAlarm` already read `data.settings.studyReminderSnoozeUntil` before the resume; could see stale value and fire one more notification. Acceptable.
- **DST forward jump in `getNextLocalMidnight`**: native `Date.setHours(24, 0, 0, 0)` handles DST correctly per spec — verified via test "rolls over month boundary". ✅
- **TEST_NOTIFICATION bypass**: confirmed `showDailyReminder` is unconditional and only the alarm path gates on snooze. Division is documented in the JSDoc comment (`notifications.ts:64-66`) — clear and safe. ✅

---

## Positive Observations

- Pure content-builder with comprehensive test matrix (12 cases covering full word, no IPA, no PoS, fallback chain, truncation thresholds) — exemplary.
- JSDoc comment on `showDailyReminder` explicitly calls out why it bypasses the snooze gate — saves future maintainers the head-scratching.
- `getStudyReminderSnoozeUntil` rejects non-finite values defensively (string `'oops'` test case).
- `setStudyReminderSnoozeUntil(NaN)` is rejected before storage write — good input validation.
- `mountedAt` snapshot via `useState(() => Date.now())` correctly avoids the React-purity violation of calling `Date.now()` directly in render.
- New `vitest.setup.ts` mocks are thorough and avoid bleed-through via `clearAllMocks`.
- No `any`, `as unknown as`, or weakened types introduced. ✅
- No PII / secrets / stack traces leaking through notification text. ✅

---

## TypeScript Strictness Audit

Searched diff for `any`, `as unknown as`, `@ts-ignore`, `@ts-expect-error`: zero hits in production code.
Test file uses `as unknown as ReturnType<typeof vi.fn>` on `chrome.permissions.contains` (notifications.test.ts:171) — justified, the global mock is typed as the chrome SDK shape and the `mockResolvedValueOnce` extension requires a vi-fn cast. Acceptable in tests.
Test casts `as { 'settings-storage'?: string }` (notifications.test.ts:29) for reading the latest mock-call payload — acceptable.

`buildReminderContent` return type uses a discriminated optional (`contextMessage?`); `notifications.ts:79` destructures all three then conditionally spreads — correct nullability handling.

---

## Security

- No XSS surface: `chrome.notifications.create` does not parse HTML; values are rendered as plain text by the OS notification renderer. Word/translation/pronunciation strings from API → notification text is safe.
- Storage write security: `setStudyReminderSnoozeUntil` writes only a number (validated) into the existing `state.settings` object. Cannot inject arbitrary keys via attacker-controlled input because the field name is hard-coded.
- Permission check (`chrome.permissions.contains`) before `chrome.notifications.create` — defensive, no bypass.
- No PII leaks: notification body shows only the user's own saved word + definition + translation, no tokens, no IDs.

✅ No security blockers.

---

## API Contract Audit

- `WordPreview.pronunciation`/`partOfSpeech` are optional — all consumers (`buildReminderContent`, `getRandomWordPreview`) handle undefined. ✅
- `NotificationData.settings` includes the new `studyReminderSnoozeUntil` — backwards compatible (older stored settings without the key just yield `undefined`). ✅
- `UserSettings.studyReminderSnoozeUntil?: number` with `?` — no DB migration needed. ✅
- `chrome.notifications.create` button array contract: index 0 = Snooze 1h, index 1 = Skip today. Handler matches. ✅

---

## Recommended Actions (priority order)

1. **C1** — Remove the duplicate `initNotifications()` in `service-worker.ts:38`, OR add an `initialized` guard in `notifications.ts`. **Blocking**.
2. **H1** — Add a code-comment near `setStudyReminderSnoozeUntil` documenting the RMW-vs-Zustand race as best-effort.
3. **H2** — Cap `randomWord.translation` length (e.g., 80 chars) in `buildContextMessage` for visual consistency.
4. **M1** — Plan a follow-up split of `notifications.ts` into `notifications.ts` + `notification-listeners.ts` (defer is OK).
5. **M2** — Extend the `setStudyReminderSnoozeUntil` test to assert envelope-root keys (e.g., Zustand `version`) are preserved.
6. **L4** — Optional: append `…` to truncated definitions.

---

## Metrics

- Type Coverage: high (no `any` introduced).
- Test Coverage: 183/183 pass; 39 new test cases across three files for new code paths.
- Linting Issues: 0.
- LOC over soft cap: 1 file (`notifications.ts` at 244, +44 over).

## Unresolved Questions

1. Should `setStudyReminderSnoozeUntil` actively notify the open Options page via `chrome.runtime.sendMessage` to mitigate the H1 race, or is best-effort acceptable for this feature?
2. Is the duplicate `initNotifications()` call in `onInstalled` intentional (e.g., to re-arm after a `chrome.runtime.reload()`)? If so, the listener idempotency guard is the cleaner fix; if not, dropping the second call is preferred.
3. `notifications.ts` is 244 LOC — split now in this PR or follow-up?

**Status:** DONE_WITH_CONCERNS
**Summary:** Implementation is correct, well-tested, and security-clean. One real bug (duplicate listener registration on install in `service-worker.ts:38, 42` — fires snooze write twice on button click) plus one race-condition documentation gap (H1) should be addressed before merge.
**Concerns/Blockers:** C1 (duplicate listener) is the only blocker; H1, H2, M1 are recommended follow-ups.
