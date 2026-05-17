---
phase: 3
title: "Update Content Script Settings Manager"
status: pending
priority: P2
effort: "30m"
dependencies: [2]
---

# Phase 3: Update Content Script Settings Manager

## Overview

The content script reads settings directly via `chrome.storage.local.get` (bypassing Zustand). Update those reads to `chrome.storage.sync.get` with one-time `local` fallback (mirrors Phase 1 adapter behavior).

## Requirements

**Functional**
- `settings-manager.ts` reads from `chrome.storage.sync` first, falls back to `chrome.storage.local`
- `chrome.storage.onChanged` listener filters `areaName === 'sync'`

**Non-functional**
- No new files; modify existing module
- Cached settings still respect `chrome.storage.onChanged` for hot updates

## Architecture

```
Before:
settings-manager.init():
  result = await chrome.storage.local.get('settings-storage')
  cache = parse(result)
  chrome.storage.onChanged listener: any area → re-cache

After:
settings-manager.init():
  result = await chrome.storage.sync.get('settings-storage')
  if (empty) result = await chrome.storage.local.get('settings-storage')   // legacy
  cache = parse(result)
  chrome.storage.onChanged listener: areaName === 'sync' → re-cache
```

The fallback is read-only here — we don't copy local→sync from content script (avoid race with Phase 1 adapter; popup/options will trigger the migration first time they read).

## Related Code Files

- **Modify:** `vocabulary-extension/src/content/modules/settings-manager.ts`

## Implementation Steps

1. Open `src/content/modules/settings-manager.ts`
2. Locate the `chrome.storage.local.get('settings-storage')` call(s)
3. Replace with sync-first + local-fallback pattern:
   ```typescript
   let result = await chrome.storage.sync.get('settings-storage')
   if (!result['settings-storage']) {
     result = await chrome.storage.local.get('settings-storage')
   }
   ```
4. Locate `chrome.storage.onChanged.addListener(...)` and add `areaName === 'sync'` filter for settings change handling
5. Run `npx tsc --noEmit`
6. Manually test: load extension, change setting in options page, verify content script tooltip updates on already-open tab

## Todo List

- [ ] Read current `settings-manager.ts` to identify exact lines
- [ ] Update `getSettings`/initialization read to sync-first pattern
- [ ] Add `areaName === 'sync'` filter to onChanged
- [ ] TypeScript compile clean
- [ ] Manual smoke test: setting change propagates to content script

## Success Criteria

- [ ] Content script reads sync storage with local fallback
- [ ] onChanged filter scoped to `'sync'` area
- [ ] No regression: tooltip + menu still respect setting changes within same device

## Risk Assessment

| Risk | Mitigation |
|------|------------|
| Content script reads stale local before sync migration completes | Fallback only fires when sync is empty; once Phase 1 adapter migrates, sync wins |
| Multiple onChanged listeners across content/store both firing | Each module owns its own cache; filter ensures only sync triggers settings re-read |
