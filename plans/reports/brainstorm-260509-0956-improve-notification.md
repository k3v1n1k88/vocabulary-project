# Brainstorm Summary — Improve Notification (Issue #6)

**Date:** 2026-05-09
**Issue:** [k3v1n1k88/vocabulary-extension#6](https://github.com/k3v1n1k88/vocabulary-extension/issues/6)
**Status:** Design approved, ready for `/ck:plan`

---

## Problem Statement

Current notification popup shows only a short title + 1-2 line message. Users want:

1. **Richer content** — IPA pronunciation, meaning/description visible in the notification.
2. **Snooze** — temporarily dismiss notifications for 1 hour or "the rest of today".

## Requirements

### Functional
- Show word + IPA + part-of-speech + definition + translation in study reminder notification.
- Two snooze actions in notification: `Snooze 1h`, `Skip today` (until next local midnight).
- Snooze pauses ONLY study reminders. Other notifications (`word-saved`, `streak-risk`, `due-cards`) unaffected.
- Settings page shows current snooze status + `Resume now` button.
- Manual `TEST_NOTIFICATION` ignores snooze (so user can verify config).
- Word without IPA: graceful fallback (title shows word only).

### Non-functional
- No new dependencies.
- Persists across browser restarts (chrome.storage).
- Cross-platform notification rendering (Win/Mac/Linux).
- Existing test coverage targets maintained.

## Evaluated Approaches

### A. Native packed notification (CHOSEN)
Use Chrome `basic` notification with all 3 text slots filled (title, message, contextMessage) + 2 buttons.

**Pros:** OS-level visibility, lock-screen, simple, no extra UI surface, fits Chrome's 2-button limit.
**Cons:** Limited styling, ~150-char message cap, no clickable audio.

### B. Notification → click → side panel rich card
Minimal notification, rich detail on click in side panel.

**Pros:** Fully styled, scalable to more fields.
**Cons:** Extra click required, doesn't satisfy "show info in the popup" intent, side-panel only works in active browser.

### C. Both (basic notif + rich in-app card on click)
Hybrid.

**Pros:** Best of both.
**Cons:** Most code, contradicts YAGNI for current scope.

**Decision rationale:** A satisfies issue spec directly with minimal code. Issue text emphasizes the *notification itself* should display info — not a follow-up surface.

## Final Solution

### Notification layout

```
┌──────────────────────────────────────────┐
│ 📖 example /ɪɡˈzæmpəl/                  │ title: word + IPA
│ noun. a thing characteristic of its kind │ message: PoS + definition (≤100 chars)
│ ví dụ · +3 more cards waiting           │ contextMessage: translation + due count
│ [ Snooze 1h ]  [ Skip today ]           │ buttons (2 max — Chrome limit)
└──────────────────────────────────────────┘
```

- Use `chrome.notifications.create` with `type: 'basic'`.
- IPA pulled from existing `Word.pronunciation` field. Skipped if missing.
- Part of speech from `Word.partOfSpeech`. Skipped if missing.

### Snooze

- New `UserSettings.studyReminderSnoozeUntil?: number` (timestamp ms).
- `handleStudyReminderAlarm` early-return when `Date.now() < snoozeUntil`.
- Button click handlers:
  - Index 0 (`Snooze 1h`): `snoozeUntil = Date.now() + 3_600_000`.
  - Index 1 (`Skip today`): `snoozeUntil = next local midnight`.
- Settings UI: status banner + `Resume now` (clears snooze).

### File changes

| File | Change |
|------|--------|
| `src/types/index.ts` | Add `studyReminderSnoozeUntil?: number` to `UserSettings`. |
| `src/shared/notification-helpers.ts` | Extend `WordPreview` (add `pronunciation`, `partOfSpeech`); add snooze get/set helpers; helper for "next midnight". |
| `src/shared/notifications.ts` | Rewrite `showDailyReminder` to packed format with buttons; gate `handleStudyReminderAlarm` on snooze; handle button clicks in `setupNotificationClickHandler`. |
| `src/options/components/learning-settings.tsx` | Add snooze status row + Resume button. |
| `src/shared/notification-helpers.test.ts` | Cover new helpers + snooze gating. |
| `src/shared/store.ts` | Persist new settings field (likely auto via Zustand spread). |

## Implementation Considerations

- **MV3 service worker:** button click listener already registered top-level in `setupNotificationClickHandler` — extend it; do not move into async closures.
- **macOS quirk:** `requireInteraction` already gated by `isMacOS()` — keep as is.
- **Truncation:** definition trimmed to 100 chars (existing convention).
- **Storage:** snooze stored alongside settings via existing `settings-storage` key — no new storage key needed (DRY).

## Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Chrome renders contextMessage differently across OSes | Low | Manual smoke test on Win/Mac before release. |
| User snoozes then forgets — misses study | Low | "Skip today" auto-clears at midnight; settings UI shows status. |
| Words with very long IPA push title past Chrome limit | Low | Title cap ~50 chars; truncate IPA defensively. |

## Success Criteria

1. Study reminder shows word + IPA + PoS + definition + translation + due count.
2. `Snooze 1h` button: alarm fires but is silently skipped for 1 hour.
3. `Skip today` button: alarm skipped until next local midnight.
4. Settings page reflects active snooze + allows manual resume.
5. `word-saved`, `streak-risk`, `due-cards` notifications unaffected by snooze.
6. Words without IPA/PoS still render correctly.
7. Unit tests cover snooze gate + button handlers + new helpers; existing tests still pass.

## Out of Scope (Deferred / YAGNI)

- Audio play button in notification (Chrome notif can't host clickable audio).
- Per-notification-type snooze (rejected by user — global study-reminder scope chosen).
- Custom snooze durations (only 1h + today for now).
- Side-panel rich card on click (current click → popup behavior kept).

## Next Steps

1. Run `/ck:plan` with this report as context to produce phased implementation plan.
2. Implement per plan.
3. Smoke-test on Windows + macOS.
4. Update `docs/project-changelog.md` after release.

## Unresolved Questions

- Should "Skip today" copy be localized? (Project supports 12 translation langs but UI itself appears English-only — confirm before plan.)
- Should `due-cards` aggregate notification respect study-reminder snooze, or stay independent? (Current decision: independent. Confirm acceptable.)
