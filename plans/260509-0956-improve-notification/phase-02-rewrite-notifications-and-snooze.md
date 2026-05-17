# Phase 02 — Rewrite Notifications + Snooze Gate

## Context Links

- Parent plan: [plan.md](plan.md)
- Brainstorm: [../reports/brainstorm-260509-0956-improve-notification.md](../reports/brainstorm-260509-0956-improve-notification.md)
- Depends on: [phase-01-types-and-helpers.md](phase-01-types-and-helpers.md)
- Related code:
  - `vocabulary-extension/src/shared/notifications.ts` (entire file, ~230 LOC)
  - `vocabulary-extension/src/background/handlers/notification-handler.ts` (lines 27-87, TEST_NOTIFICATION)
  - `vocabulary-extension/src/shared/notification-helpers.ts` (helpers from phase-01)

## Overview

- Date: 2026-05-09
- Priority: P2
- Implementation status: pending
- Review status: not started
- Description: Rewrite `showDailyReminder` to packed format (title/message/contextMessage + 2 buttons). Gate `handleStudyReminderAlarm` on snooze. Extend `setupNotificationClickHandler` to map button index → snooze action. Verify TEST_NOTIFICATION still bypasses snooze.

## Key Insights

- Brainstorm "Final Solution → Notification layout": title=`📖 {word} {/IPA/}`; message=`{PoS}. {def≤100ch}`; contextMessage=`{translation} · +{N-1} more cards waiting`; 2 buttons.
- Truncation rule: definition ≤100 chars (existing convention); IPA defensive title cap ~50 chars.
- Snooze gate **only in `handleStudyReminderAlarm`** — not in `showDailyReminder`. This guarantees `TEST_NOTIFICATION` (which calls `showDailyReminder` directly via `notification-handler.ts:85`) ignores snooze. Verified via grep: `notification-handler.ts:85` calls `showDailyReminder`, never the alarm path.
- MV3 quirk: `chrome.notifications.onButtonClicked` listener MUST be registered top-level (already at `notifications.ts:199`). Extend the existing listener, do not move into closures.
- Existing `setupNotificationClickHandler` (`notifications.ts:198-210`) opens popup on any button click. Brainstorm changes: button 0 = snooze 1h, button 1 = skip today, body click = unchanged (open popup).
- Notification ID prefix `daily-reminder-` (line 91) lets click handler distinguish reminder buttons from `due-cards`/`streak-risk`/`word-saved` IDs (which never have buttons today).

## Requirements

### Functional

1. `showDailyReminder(dueCount, streak, randomWord?)` rewrites to packed format:
   - title: word missing ⇒ `'Time to Study!'` (existing fallback). Word present + IPA ⇒ `📖 {word} /{IPA}/`. Word present, no IPA ⇒ `📖 {word}`. Truncate IPA so title ≤ 50 chars.
   - message: PoS+def ⇒ `{partOfSpeech}. {def≤100ch}`. No PoS ⇒ `{def≤100ch}`. No word at all ⇒ existing fallbacks (due count / streak / generic).
   - contextMessage: built from `{translation}` and `+{dueCount-1} more cards waiting`. Join with ` · `. Either side optional. Skip if both empty.
   - buttons: `[{title:'Snooze 1h'}, {title:'Skip today'}]` ALWAYS when notification refers to study reminder (i.e., when called from study path or test). Buttons listed even on fallback messages — gives users the snooze controls.
2. `handleStudyReminderAlarm` early-returns when `Date.now() < studyReminderSnoozeUntil`. Auto-clears expired snooze (set to `undefined`) for tidiness.
3. `setupNotificationClickHandler` extended:
   - If `notificationId` starts with `daily-reminder-`:
     - `buttonIndex === 0` → `setStudyReminderSnoozeUntil(Date.now() + 3_600_000)`; clear notif; do NOT open popup.
     - `buttonIndex === 1` → `setStudyReminderSnoozeUntil(getNextLocalMidnight())`; clear notif; do NOT open popup.
   - Else (other notif types): keep existing behavior (open popup on any button click — currently no other notif has buttons, so no functional change).
   - Body click (`onClicked`): unchanged (`chrome.action.openPopup` + clear).
4. `TEST_NOTIFICATION` path verified to call `showDailyReminder` directly — bypasses snooze gate. No code change in `notification-handler.ts`.

### Non-functional

- File size: `notifications.ts` ~230 LOC currently; adding snooze logic + buttons stays under 200 only if we extract. Plan: keep core in `notifications.ts` ≤ 200 LOC by:
  - Extract notification-content building (`buildReminderContent`) into `notification-helpers.ts` (or a new sibling `notification-content-builder.ts` if helpers file grows past 200).
  - Note: `notification-helpers.ts` will grow ~+60 LOC from phase-01. If projected total > 200 LOC → split content builder into `notification-content-builder.ts`.
- TS strict, no `any`.
- Keep `isMacOS()` gate on `requireInteraction` unchanged.
- Keep `priority: 2`.

## Architecture

```
chrome.alarms.onAlarm
        |
        v
handleStudyReminderAlarm
        |  reads getStudyReminderSnoozeUntil
        |  if now < snoozeUntil: return
        |  if now >= snoozeUntil and snoozeUntil set: setStudyReminderSnoozeUntil(undefined)
        |
        v
showDailyReminder(dueCount, streak, randomWord)
        |  buildReminderContent -> {title, message, contextMessage}
        v
chrome.notifications.create({type:'basic', ..., buttons:[Snooze 1h, Skip today]})

chrome.notifications.onButtonClicked  (top-level)
   id starts with 'daily-reminder-' ?
      yes -> idx 0: setStudyReminderSnoozeUntil(now + 3600000)
             idx 1: setStudyReminderSnoozeUntil(getNextLocalMidnight())
             clear, return
      no  -> existing behavior (openPopup + clear)

chrome.notifications.onClicked (body)
   openPopup + clear  (UNCHANGED)

TEST_NOTIFICATION -> showDailyReminder directly  (NO snooze gate)
```

## Related Code Files

### Modify

- `vocabulary-extension/src/shared/notifications.ts`:
  - Rewrite `showDailyReminder` (lines 58-103).
  - Gate `handleStudyReminderAlarm` (lines 151-158).
  - Extend `setupNotificationClickHandler` (lines 198-210).
  - Import `buildReminderContent` from `./notification-content-builder`.
  - Import `getNextLocalMidnight`, `setStudyReminderSnoozeUntil` from `./notification-helpers`.

### Create

- `vocabulary-extension/src/shared/notification-content-builder.ts` — pure content-building module. <!-- Updated: Validation Session 1 - Q3 decided strict separation, mandatory new file -->
  - Exports `buildReminderContent(dueCount: number, streak: number, randomWord?: WordPreview): { title: string; message: string; contextMessage?: string }`.
  - No Chrome API calls. No storage access. Pure function.
  - Imports `WordPreview` type from `./notification-helpers`.

### Delete

- None.

### Verify (no change expected)

- `vocabulary-extension/src/background/handlers/notification-handler.ts` — confirm `handleTestNotification` still calls `showDailyReminder` directly (line 85). No edit.

## Implementation Steps

1. **Add `buildReminderContent`** in NEW `src/shared/notification-content-builder.ts`: <!-- Updated: Validation Session 1 - moved out of notification-helpers.ts -->
   - Pure function. No chrome calls. Inputs: `dueCount`, `streak`, `randomWord?: WordPreview`. Returns `{title: string; message: string; contextMessage?: string}`.
   - Title logic:
     - `randomWord?.word` missing → `'Time to Study!'`.
     - With word: build `📖 {word}` then append ` /{ipa}/` if `pronunciation` present and total ≤ 50 chars; otherwise drop IPA.
   - Message logic:
     - `randomWord` present:
       - `def100 = randomWord.definition.slice(0, 100)`.
       - `partOfSpeech` present → `{partOfSpeech}. {def100}`; else → `def100`.
     - `randomWord` absent → existing fallbacks: `dueCount > 0` → "You have N cards waiting for review!"; else streak > 0 → "Keep your N-day streak going!"; else "Start building your vocabulary today!".
   - contextMessage logic:
     - Parts: `randomWord?.translation` (skip if empty); `dueCount > 1` → `+{dueCount-1} more cards waiting`.
     - Join non-empty parts with ` · `. If empty, return `undefined`.
2. **Rewrite `showDailyReminder`** (`notifications.ts`):
   - Replace body lines 70-89 with `const { title, message, contextMessage } = buildReminderContent(dueCount, streak, randomWord)`.
   - Pass to `chrome.notifications.create`:
     ```ts
     {
       type: 'basic',
       iconUrl: chrome.runtime.getURL('icons/icon-128.png'),
       title,
       message,
       ...(contextMessage ? { contextMessage } : {}),
       buttons: [{ title: 'Snooze 1h' }, { title: 'Skip today' }],
       priority: 2,
       ...(isMacOS() ? {} : { requireInteraction: true })
     }
     ```
3. **Gate `handleStudyReminderAlarm`**:
   - Top of function, after `getNotificationData()`:
     - `const snoozeUntil = data.settings?.studyReminderSnoozeUntil`
     - `const now = Date.now()`
     - `if (snoozeUntil && now < snoozeUntil) return`
     - `if (snoozeUntil && now >= snoozeUntil) { await setStudyReminderSnoozeUntil(undefined) }` (auto-clear stale)
   - Or read snooze separately via `getStudyReminderSnoozeUntil()` if `data.settings` shape doesn't already include it after phase-01 update. Prefer reading from already-fetched `data.settings` (single storage read) for perf.
4. **Extend `setupNotificationClickHandler`**:
   - In `onButtonClicked` listener:
     ```ts
     if (notificationId.startsWith('daily-reminder-')) {
       if (buttonIndex === 0) {
         await setStudyReminderSnoozeUntil(Date.now() + 3_600_000)
       } else if (buttonIndex === 1) {
         await setStudyReminderSnoozeUntil(getNextLocalMidnight())
       }
       chrome.notifications.clear(notificationId)
       return
     }
     // Fall-through: existing behavior (preserve back-compat for any future button on other notifs)
     if (buttonIndex === 0) chrome.action.openPopup()
     chrome.notifications.clear(notificationId)
     ```
   - Make outer arrow `async` to support `await`.
   - Listener registration MUST stay synchronous + top-level (don't wrap in async IIFE).
5. **Verify TEST_NOTIFICATION**: open `notification-handler.ts`; confirm line 85 still `await showDailyReminder(...)`. No edit. Add inline code comment near `showDailyReminder` declaration: `// NOTE: snooze gate lives in handleStudyReminderAlarm; this fn is unconditional so TEST_NOTIFICATION + future direct callers can fire.`
6. **Type check + lint**: `cd vocabulary-extension && npx tsc --noEmit && npm run lint`.
7. **Manual smoke** (deferred to QA): trigger via Options page Test button → confirm 3 text rows + 2 buttons render on Win/Mac.

## Todo List

- [ ] Create new file `src/shared/notification-content-builder.ts` with `buildReminderContent` pure helper
- [ ] Rewrite `showDailyReminder` body to use `buildReminderContent` + buttons
- [ ] Add snooze gate at top of `handleStudyReminderAlarm`
- [ ] Auto-clear stale snooze when `now >= snoozeUntil`
- [ ] Extend button-click listener with `daily-reminder-` prefix branch
- [ ] Convert `onButtonClicked` listener to async to support await
- [ ] Verify TEST_NOTIFICATION still bypasses (no code change in `notification-handler.ts`)
- [ ] Add inline comment on `showDailyReminder` clarifying gate location
- [ ] `npx tsc --noEmit` clean
- [ ] `npm run lint` clean

## Success Criteria

- `showDailyReminder` produces 3 text rows + 2 buttons when word + all metadata present.
- Word missing → falls back to legacy text but still shows snooze buttons.
- Snooze 1h click → next alarm tick within 1h is skipped silently.
- Skip today click → next alarm tick before next local midnight is skipped silently.
- After snooze expiry, next alarm tick auto-clears `studyReminderSnoozeUntil` and fires normally.
- `TEST_NOTIFICATION` always renders, even mid-snooze.
- `due-cards-check` alarm path untouched (verify `handleDueCardsCheckAlarm` line 163 unchanged).
- Type-check + lint pass.

Validation: unit tests in phase-04; manual smoke on Test button.

## Risk Assessment

| Risk | Severity | Likelihood | Mitigation |
|------|----------|------------|------------|
| Forget to make button listener `async` → race on `setStudyReminderSnoozeUntil` write | Med | Med | Test: snooze click → immediately re-trigger alarm → must be skipped. Add unit test. |
| Move listener registration into async closure (MV3 service-worker dies) | High | Low | Code review checklist; comment above listener: "MV3: keep registration top-level synchronous." |
| Notification ID prefix collision (`daily-reminder-` vs future IDs) | Low | Low | Document prefix as reserved in `notifications.ts` comment. |
| `contextMessage` rendering differs across OSes (Windows action center may hide it) | Low | Med | Brainstorm risk #1 — manual smoke on Win + Mac; acceptable degradation since data also in title/message. |
| `chrome.notifications.create` rejects `buttons` array on unsupported platforms | Low | Low | All MV3 desktop Chromes support 2 buttons; documented Chrome behavior. |
| Snooze gate reads stale `settings-storage` (Zustand wrote but storage hasn't fsync'd) | Low | Low | `chrome.storage.local.set` is durable per Chrome guarantees; alarm fires on minute scale. |
| Auto-clear write inside alarm handler races user resume | Low | Low | Both writes converge on `undefined`; idempotent. |

## Security Considerations

- No new permissions.
- Notification icon already permitted.
- Snooze write through existing storage path; no expanded surface.
- No untrusted user input in notification text — `Word.word`/`pronunciation`/`partOfSpeech`/`definition`/`vietnameseTranslation` already user-or-API-sourced and rendered by Chrome's native notifications (no XSS surface — Chrome strips HTML).

## Next Steps

- Phase-03 (UI) can run in parallel; consumes same `studyReminderSnoozeUntil` field.
- Phase-04 tests cover: `buildReminderContent` matrix; snooze gate; button-click handler dispatch.
- Post-merge QA: smoke test on Win + Mac (brainstorm risk #1).
