---
phase: 2
title: "Wire Settings Store"
status: pending
priority: P2
effort: "1h"
dependencies: [1]
---

# Phase 2: Wire Settings Store

## Overview

Switch `useSettingsStore` to use the new sync adapter. Fix the `chrome.storage.onChanged` listener to filter by `areaName` so cross-device settings updates rehydrate the store correctly.

## Requirements

**Functional**
- `useSettingsStore` persists via `chromeSyncStorage` (was `chromeStorage`)
- `onChanged` listener for `'settings-storage'` key triggers only on `areaName === 'sync'` (avoid duplicate fires)
- Other stores (`useVocabularyStore`, `useStatsStore`) untouched

**Non-functional**
- No behavior change for vocabulary/stats stores
- Backward-compatible: existing local data auto-migrates via Phase 1 adapter

## Architecture

```
Before:
useSettingsStore → chromeStorage → chrome.storage.local
onChanged listener: fires for ANY area on 'settings-storage' key

After:
useSettingsStore → chromeSyncStorage → chrome.storage.sync (+ legacy local fallback)
onChanged listener: filtered to areaName === 'sync'
```

The existing listener (`store.ts:238-251`) already correctly parses Zustand's persisted JSON shape. Only change: filter `areaName`.

## Related Code Files

- **Modify:** `vocabulary-extension/src/shared/store.ts`
  - Line ~4: add import `chromeSyncStorage` from `./chrome-sync-storage-adapter`
  - Line ~232: settings store storage adapter swap
  - Lines ~238-251: add area filter to onChanged listener

## Implementation Steps

1. Open `src/shared/store.ts`
2. Add import: `import { chromeSyncStorage } from './chrome-sync-storage-adapter'`
3. In `useSettingsStore` persist config (around line 232): change `() => chromeStorage` → `() => chromeSyncStorage`
4. In the onChanged listener (line ~239), update signature to capture `areaName` and early-return if not `'sync'`:
   ```typescript
   chrome.storage.onChanged.addListener((changes, areaName) => {
     if (areaName !== 'sync') return
     if (changes['settings-storage']?.newValue) { ... }
   })
   ```
5. Verify `useVocabularyStore` and `useStatsStore` still use `chromeStorage` (no change)
6. Run `npx tsc --noEmit`
7. Run unit tests: `npm run test:unit -- store.test`

## Todo List

- [ ] Import `chromeSyncStorage` in `store.ts`
- [ ] Swap settings store adapter to `chromeSyncStorage`
- [ ] Add `areaName === 'sync'` filter to settings onChanged listener
- [ ] Verify vocabulary/stats stores unchanged
- [ ] TypeScript compile clean
- [ ] Existing unit tests still pass

## Success Criteria

- [ ] `useSettingsStore` writes to `chrome.storage.sync`
- [ ] onChanged listener filters by `areaName === 'sync'`
- [ ] `useVocabularyStore` + `useStatsStore` still use `chrome.storage.local`
- [ ] TypeScript compile clean
- [ ] Existing tests pass

## Risk Assessment

| Risk | Mitigation |
|------|------------|
| Forgetting area filter → duplicate fires when local-only stores change | Phase 4 test covers this; manual code review |
| Wrong adapter applied to vocabulary store accidentally | Diff review; vocab store keeps `chromeStorage` import |
