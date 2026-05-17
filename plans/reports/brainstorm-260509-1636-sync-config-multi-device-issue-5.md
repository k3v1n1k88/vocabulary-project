# Brainstorm: Sync Config Across Devices (Issue #5)

**Date:** 2026-05-09
**Issue:** [k3v1n1k88/vocabulary-extension#5](https://github.com/k3v1n1k88/vocabulary-extension/issues/5)
**Branch:** master → feat/sync-config-issue-5

---

## Problem Statement

Users running the extension on multiple devices (work laptop, home desktop, etc.) must reconfigure settings on each device. No cross-device sync today — `chrome.storage.local` is device-scoped.

**User pain:** Re-entering target language, LLM provider + API key, daily goal, theme, shortcuts, highlight color, reminders on every device. High friction → likely cause of churn.

---

## Scope (Locked via AskUserQuestion)

| Question | Decision |
|----------|----------|
| What syncs? | **Settings only.** Vocabulary, stats, highlights stay local. |
| API keys synced? | **Yes** — ride Google's encrypted sync infra (same model as bookmarks). |
| Migration? | **Auto on first read** — zero user action. |

Out of scope: vocabulary sync, user accounts, Firebase, encrypted-at-rest passphrase.

---

## Approaches Evaluated

### A. `chrome.storage.sync` swap (CHOSEN)
- Built-in MV3 API, free, no backend
- Quotas: 8KB/item, 100KB total, 512 items — **settings JSON ~0.4-2KB → 4-10× headroom**
- Riding existing Chrome sync = same security as bookmarks/history
- Effort: ~1 day

### B. Firebase + custom backend (rejected)
- Phase 2 plan in roadmap — unclear if dependency is dead code
- Requires: auth UI, encryption client-side, conflict resolution, GDPR compliance
- Effort: 4-6 weeks. **Massive over-engineering for 2KB of preferences.**

### C. Hybrid: sync settings now, defer vocab cloud sync (deferred)
- Same as A for this issue. Vocab cloud sync is a separate Phase 2 effort, not blocked by this work.

**Decision rationale:** YAGNI/KISS dominate. Issue is "sync **config**" not full data. chrome.storage.sync ships in days, not weeks, and doesn't preclude future cloud features.

---

## Recommended Solution

### Architecture

```
useSettingsStore  →  chromeSyncStorage (NEW)  →  chrome.storage.sync
                                              ↘ fallback (one-time read) → chrome.storage.local
```

Other stores (`useVocabularyStore`, `useStatsStore`) **unchanged** — stay on `chrome.storage.local` (size + privacy).

### Migrate-on-Read

```
chromeSyncStorage.getItem(name):
  1. Read sync → if present, return
  2. Else read local (legacy) → if present, copy to sync, return
  3. Else null
```

No version flags, no first-run hooks. Existing v1.0.5 users seamlessly upgrade.

### Cross-Tab + Cross-Device Sync

`chrome.storage.onChanged` fires with `(changes, areaName)`. Existing listener at `store.ts:239-251` doesn't filter `areaName` — fix to filter `areaName === 'sync'` for settings store.

### Files Touched

| File | Change |
|------|--------|
| `src/shared/chrome-sync-storage-adapter.ts` | NEW (~60 LOC) |
| `src/shared/store.ts` | Settings store uses new adapter; onChanged area filter |
| `src/content/modules/settings-manager.ts` | Read from `chrome.storage.sync` |
| `src/shared/store.test.ts` | Migration + sync change tests |
| `docs/system-architecture.md` | Update Storage Schema section |

Manifest `"storage"` permission already present.

---

## Implementation Considerations & Risks

| Risk | Mitigation |
|------|------------|
| User has Chrome sync disabled | sync API still persists locally — no-op until sign-in. Graceful. |
| Quota exceeded (long API key, future settings bloat) | try/catch on setItem → fall back to local + console.warn |
| Conflict: same setting changed on 2 devices simultaneously | Chrome's last-write-wins by timestamp. Acceptable for prefs. |
| Stale local copy after migration | Harmless tombstone. Optional sweep in v2.0. |
| onChanged listener fires for both areas | Filter `areaName === 'sync'` for settings handler |

---

## Success Metrics

- Setting changed on Device A appears on Device B within Chrome's sync interval (~seconds)
- Existing v1.0.5 users see zero data loss on upgrade
- If Chrome sync off → extension still works locally
- No regression in same-device cross-tab settings sync
- Test coverage maintained or improved
- Vocabulary/stats unaffected

---

## Next Steps

1. Plan creation: `260509-1636-sync-config-multi-device-issue-5/`
2. Implementation phases:
   - Phase 1: Sync storage adapter + migration logic
   - Phase 2: Wire settings store + onChanged area filter
   - Phase 3: Content script settings-manager update
   - Phase 4: Tests + docs update
3. Manual cross-device verification on signed-in Chrome profile
4. PR linking to issue #5

---

## Unresolved Questions

1. **Cleanup of legacy `settings-storage` in local after sync migration?** Recommend defer to v2.0; harmless tombstone.
2. **Should `useUIStore` (activeTab) ever sync?** Current verdict: no — UI state is per-device session. Out of scope.
3. **Telemetry on migration success rate?** No analytics infra today; not required for this feature.
