# Phase 03 — Settings UI: Snooze Banner + Resume

## Context Links

- Parent plan: [plan.md](plan.md)
- Brainstorm: [../reports/brainstorm-260509-0956-improve-notification.md](../reports/brainstorm-260509-0956-improve-notification.md)
- Depends on: [phase-01-types-and-helpers.md](phase-01-types-and-helpers.md)
- Independent of: phase-02 (can run parallel after phase-01).
- Related code:
  - `vocabulary-extension/src/options/components/learning-settings.tsx` (~171 LOC, fits new banner under 200 cap)
  - `vocabulary-extension/src/shared/store.ts:200-235` (`useSettingsStore`)
  - `vocabulary-extension/src/shared/notification-helpers.ts` (helpers from phase-01)

## Overview

- Date: 2026-05-09
- Priority: P2
- Implementation status: pending
- Review status: not started
- Description: Add a snooze status banner to `learning-settings.tsx`. When `settings.studyReminderSnoozeUntil` is in the future and `notificationsEnabled`, show: `🔕 Reminders paused until {locale time/date} [Resume now]`. Resume button clears the field.

## Key Insights

- Brainstorm "Settings UI" item: status row + Resume button.
- `settings` prop already typed `UserSettings` (line 5) — once phase-01 adds the field, TS picks it up automatically.
- Resume should write through `onSettingsUpdate` (line 6) which routes to `useSettingsStore.updateSettings` — keeps single write path. This means we do NOT call `setStudyReminderSnoozeUntil` helper here; we use the Zustand action. (Helper exists for the background button-click path which has no React/Zustand context.)
- Banner placement: inside the `notificationsEnabled` block (after line 119) so it only shows when reminders are enabled overall. If user disables notifications, snooze status is moot.
- Locale formatting: use `new Date(ts).toLocaleString()` — no extra dependency.
- Auto-refresh: timestamp may pass while options page open. Acceptable to render once on mount; user can refresh. YAGNI: no `setInterval` re-render.

## Requirements

### Functional

1. When `settings.notificationsEnabled === true` AND `settings.studyReminderSnoozeUntil` is defined AND `Date.now() < studyReminderSnoozeUntil`:
   - Render banner: `🔕 Reminders paused until {Date.toLocaleString(snoozeUntil)}` + `[Resume now]` button.
2. Resume button → `onSettingsUpdate({ studyReminderSnoozeUntil: undefined })`.
3. When snooze expired (`Date.now() >= studyReminderSnoozeUntil`): do not render banner. (Background alarm handler from phase-02 will auto-clear field on next tick.)
4. When snooze undefined or notifications disabled: do not render banner.

### Non-functional

- No new dependency.
- No new component file (banner is small, ~10-15 LOC inline). Keep `learning-settings.tsx` under 200 LOC.
- Tailwind classes consistent with existing styling (`bg-white rounded-xl ...`).
- TS strict, no `any`.

## Architecture

```
learning-settings.tsx
   reads settings.studyReminderSnoozeUntil (UserSettings)
        |
        v
   isSnoozeActive = settings.studyReminderSnoozeUntil
                    && Date.now() < settings.studyReminderSnoozeUntil
        |
        v
   if isSnoozeActive: render banner (inside notificationsEnabled block)
        |
        v
   Resume click -> onSettingsUpdate({ studyReminderSnoozeUntil: undefined })
        |
        v
   useSettingsStore -> chrome.storage.local (settings-storage)
        |
        v
   chrome.storage.onChanged -> background reads on next alarm tick (no snooze)
```

## Related Code Files

### Modify

- `vocabulary-extension/src/options/components/learning-settings.tsx`:
  - Insert snooze banner block inside `{settings.notificationsEnabled && ( ... )}` (between lines 119 and the closing of that block) OR as a separate row immediately under the notification toggle, before the "Reminder Interval" select. Choose: under the toggle, before interval select, so user sees status first.

### Create

- None.

### Delete

- None.

## Implementation Steps

1. **Compute snooze state** at top of `LearningSettings` body (after `function LearningSettings(...) {`):
   ```ts
   const snoozeUntil = settings.studyReminderSnoozeUntil
   const isSnoozeActive = !!snoozeUntil && Date.now() < snoozeUntil
   const snoozeUntilLabel = snoozeUntil ? new Date(snoozeUntil).toLocaleString() : ''
   ```
2. **Render banner** inside `{settings.notificationsEnabled && ( ... )}` block, immediately before the existing "Reminder Interval" `<label>`:
   ```tsx
   {isSnoozeActive && (
     <div className="mb-4 flex items-center justify-between rounded-lg bg-amber-50 border border-amber-200 px-3 py-2">
       <div className="text-sm text-amber-900">
         🔕 Reminders paused until <span className="font-medium">{snoozeUntilLabel}</span>
       </div>
       <button
         onClick={() => onSettingsUpdate({ studyReminderSnoozeUntil: undefined })}
         className="text-sm text-primary-700 hover:text-primary-800 font-medium"
       >
         Resume now
       </button>
     </div>
   )}
   ```
3. **Verify** banner only renders when both `notificationsEnabled` (gated by outer block) AND `isSnoozeActive` are true.
4. **Visual check** in dev build: open Options → Learning → toggle on reminders → manually inject `studyReminderSnoozeUntil` via DevTools storage → reload → confirm banner. Click Resume → confirm banner disappears + storage cleared.
5. **Type check + lint**: `cd vocabulary-extension && npx tsc --noEmit && npm run lint`.

## Todo List

- [ ] Compute `isSnoozeActive` + `snoozeUntilLabel` at top of component
- [ ] Render banner inside `notificationsEnabled` block, before Reminder Interval
- [ ] Wire Resume button to `onSettingsUpdate({ studyReminderSnoozeUntil: undefined })`
- [ ] Confirm banner hidden when notifications disabled
- [ ] Confirm banner hidden when snooze expired
- [ ] `npx tsc --noEmit` clean
- [ ] `npm run lint` clean
- [ ] File still under 200 LOC

## Success Criteria

- Banner appears with correct localized timestamp when snooze active.
- Banner hidden when: notifications off, snooze undefined, snooze expired.
- Resume click clears `studyReminderSnoozeUntil` + banner disappears immediately (Zustand re-renders on store update).
- File ≤ 200 LOC.
- Type-check + lint pass.

Validation: visual smoke + integration with phase-02 (snooze a notif → open Options → see banner → click Resume → trigger alarm → notif fires).

## Risk Assessment

| Risk | Severity | Likelihood | Mitigation |
|------|----------|------------|------------|
| Banner shows stale state (snooze expired but user still sees banner) | Low | Med | Acceptable; user reload refreshes. YAGNI: no auto-tick. |
| `toLocaleString()` differs across locales (DST/timezone confusion) | Low | Low | Native API; no fix needed. |
| Resume click doesn't propagate to background alarm in flight | Low | Low | Alarm fires every N min; next tick re-reads storage and finds `undefined`. |
| File grows past 200 LOC after banner | Low | Low | Banner ~12 LOC; current file 171 LOC → ~183 LOC final. |
| Tailwind `amber-50` not in palette | Low | Low | Default Tailwind palette includes amber. Confirm via project's `tailwind.config` (out-of-scope — fall back to `yellow-50` if missing). |

## Security Considerations

- No new permissions.
- No PII in banner text (timestamp only).
- No untrusted input rendered (timestamp from local storage).
- Resume button is a no-arg action — no XSS/injection surface.

## Next Steps

- Phase-04 covers integration tests if feasible (banner render gate); primarily verified via component-level smoke.
- After all phases merge: update `docs/project-changelog.md` (post-implementation, separate task).
