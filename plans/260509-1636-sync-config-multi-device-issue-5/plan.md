---
title: "Sync Config Multi-Device (Issue #5)"
status: completed
priority: P2
created: 2026-05-09
issue: https://github.com/k3v1n1k88/vocabulary-extension/issues/5
brainstorm: ../reports/brainstorm-260509-1636-sync-config-multi-device-issue-5.md
blockedBy: []
blocks: []
---

# Sync Config Multi-Device (Issue #5)

## Summary

Settings sync across user's devices via `chrome.storage.sync`. Vocabulary, stats, highlights remain device-local. Migrate-on-read pattern: existing users seamlessly upgrade. No backend, no auth, no Firebase.

## Why

Users on multiple devices currently re-configure settings (target language, LLM provider, API keys, daily goal, theme, shortcuts, highlight color) on each device. High friction. Issue #5 scope is "config" only — `chrome.storage.sync` is the KISS solution (built-in MV3, free, no infra).

## Approach

Replace the storage adapter for **settings store only** with a `chrome.storage.sync`-backed adapter that auto-migrates from `chrome.storage.local` on first read. Other stores unchanged.

## Phases

| # | Phase | Status | Effort |
|---|-------|--------|--------|
| 01 | Sync Storage Adapter | completed | 1-2h |
| 02 | Wire Settings Store | completed | 1h |
| 03 | Update Content Script Settings Manager (expanded scope) | completed | 2-3h |
| 04 | Tests And Docs | completed | 2-3h |

**Note:** Phase 3 expanded mid-execution. Beyond `settings-manager.ts`, additional files needed sync routing to prevent split-brain (settings-storage was directly accessed in 8 other files): `floating-menu.ts`, `keyboard-shortcuts.ts`, `use-sidepanel-data.ts`, `translation-settings.ts`, `notifications.ts`, `notification-helpers.ts`, `use-api-key-management.ts`. A shared helper `settings-storage-access.ts` was added (DRY) for sync-first reads + RMW patches.

**Total:** ~1 day

## Key Files

- NEW: `vocabulary-extension/src/shared/chrome-sync-storage-adapter.ts`
- MOD: `vocabulary-extension/src/shared/store.ts`
- MOD: `vocabulary-extension/src/content/modules/settings-manager.ts`
- MOD: `vocabulary-extension/src/shared/store.test.ts`
- MOD: `docs/system-architecture.md`

## Dependencies

None blocking. Manifest `"storage"` permission already declared.

## Success Criteria

- Setting changed on Device A appears on Device B (verified manually)
- Existing v1.0.5 users: zero data loss on upgrade
- Chrome sync disabled → extension still functional (local fallback in sync API)
- No regression in same-device cross-tab settings sync
- Test coverage maintained
- Vocabulary/stats unaffected

## Risks

- Quota exceeded → try/catch + fallback to local
- Conflict on simultaneous edit → last-write-wins (acceptable for prefs)

## Out of Scope

- Vocabulary sync (Phase 2 separate effort)
- User accounts / auth
- Firebase
- Encrypted-at-rest API keys with passphrase
