---
phase: 1
title: "Sync Storage Adapter"
status: pending
priority: P2
effort: "1-2h"
dependencies: []
---

# Phase 1: Sync Storage Adapter

## Overview

Create a new storage adapter that uses `chrome.storage.sync` with a one-time fallback read from `chrome.storage.local` to seamlessly migrate existing users.

## Requirements

**Functional**
- Implement `StorageAdapter` interface (already defined in `chrome-storage-adapter.ts:6-10`)
- `getItem`: prefer sync; fallback to local once; copy local→sync on hit
- `setItem`: write to sync only
- `removeItem`: remove from sync only
- Quota-exceeded fallback: catch sync write errors, fallback to local with console.warn

**Non-functional**
- File <100 LOC (DRY: import `StorageAdapter` interface from existing adapter)
- All chrome API calls wrapped in try/catch
- No new permissions required

## Architecture

```
chromeSyncStorage.getItem(name):
  try:
    syncResult = await chrome.storage.sync.get(name)
    if syncResult[name] !== undefined: return syncResult[name]
    // Migration: legacy local read
    localResult = await chrome.storage.local.get(name)
    if localResult[name] !== undefined:
      await chrome.storage.sync.set({ [name]: localResult[name] })   // best-effort
      return localResult[name]
    return null
  catch: return null

chromeSyncStorage.setItem(name, value):
  try:
    await chrome.storage.sync.set({ [name]: value })
  catch (QUOTA error):
    console.warn('sync quota exceeded, falling back to local')
    await chrome.storage.local.set({ [name]: value })

chromeSyncStorage.removeItem(name):
  try: await chrome.storage.sync.remove(name)
  catch: log
```

**Note:** Migration copy is best-effort. If sync.set throws (e.g., user offline + quota race), we still return the value from local — user sees no breakage.

## Related Code Files

- **Create:** `vocabulary-extension/src/shared/chrome-sync-storage-adapter.ts`
- **Read for reference:** `vocabulary-extension/src/shared/chrome-storage-adapter.ts` (existing pattern)

## Implementation Steps

1. Create `vocabulary-extension/src/shared/chrome-sync-storage-adapter.ts`
2. Import `StorageAdapter` interface from `./chrome-storage-adapter` (DRY — don't redefine)
3. Implement `chromeSyncStorage` const matching the interface
4. `getItem`: sync first, then legacy local fallback with copy-on-hit
5. `setItem`: sync write with quota-exceeded catch → local fallback
6. `removeItem`: sync remove with try/catch
7. Add JSDoc explaining migration semantics
8. Run `npx tsc --noEmit` to verify no compile errors

## Todo List

- [ ] Create file `chrome-sync-storage-adapter.ts`
- [ ] Import `StorageAdapter` interface
- [ ] Implement `getItem` with sync→local fallback + auto-copy
- [ ] Implement `setItem` with quota-exceeded local fallback
- [ ] Implement `removeItem`
- [ ] Add JSDoc documenting migration behavior
- [ ] `npx tsc --noEmit` passes

## Success Criteria

- [ ] File exists, exports `chromeSyncStorage` const
- [ ] Interface matches `StorageAdapter`
- [ ] TypeScript compile clean
- [ ] No new dependencies added
- [ ] File ≤100 LOC

## Risk Assessment

| Risk | Mitigation |
|------|------------|
| Race: setItem during getItem migration | Both calls are awaited; chrome.storage serializes per-area writes |
| Sync API unavailable in old Chrome | Manifest already requires modern Chrome; sync is standard since Chrome 20 |
| Migration loop (sync set fails repeatedly) | Best-effort copy; if it fails, next read retries. No infinite loop |
