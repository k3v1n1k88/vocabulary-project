---
phase: 4
title: "Tests And Docs"
status: pending
priority: P2
effort: "2-3h"
dependencies: [1, 2, 3]
---

# Phase 4: Tests And Docs

## Overview

Add unit tests covering the sync adapter migration path and onChanged area filter. Update `docs/system-architecture.md` to reflect dual storage areas. Manual cross-device verification.

## Requirements

**Functional**
- Unit tests cover: sync hit, sync miss + local hit (migration), sync miss + local miss (null), quota-exceeded fallback, onChanged area filter
- `docs/system-architecture.md` Storage Schema section split into `chrome.storage.sync` (settings) vs `chrome.storage.local` (vocabulary, stats, highlights, ui)
- Manual test plan executed on signed-in Chrome profile across 2 devices

**Non-functional**
- Test coverage for new adapter ≥85%
- No fake data, no mocks of business logic — only mock `chrome.storage` APIs (vitest-chrome already provides this)

## Architecture

Test surface:

```
chrome-sync-storage-adapter.test.ts (NEW or extend store.test.ts):
  - getItem: sync has data → returns it (no local read)
  - getItem: sync empty, local has data → returns it + writes to sync
  - getItem: both empty → returns null
  - setItem: sync.set succeeds → no local write
  - setItem: sync.set throws QUOTA → falls back to local.set
  - removeItem: removes from sync only

store.test.ts additions:
  - settings store onChanged listener: ignores areaName === 'local'
  - settings store onChanged listener: rehydrates on areaName === 'sync'
```

## Related Code Files

- **Create:** `vocabulary-extension/src/shared/chrome-sync-storage-adapter.test.ts` (or co-locate in `store.test.ts` if tiny)
- **Modify:** `vocabulary-extension/src/shared/store.test.ts` (onChanged area filter test)
- **Modify:** `docs/system-architecture.md` (Storage Schema section + Open Questions section: remove sync clarification)
- **Modify:** `docs/project-roadmap.md` (mark "settings sync" delivered; vocab sync remains Phase 2)

## Implementation Steps

1. Create `chrome-sync-storage-adapter.test.ts`
2. Set up vitest-chrome mocks for `chrome.storage.sync` + `chrome.storage.local`
3. Write `getItem` tests: sync-hit / sync-miss-local-hit / both-miss
4. Write `setItem` tests: success / quota-exceeded fallback
5. Write `removeItem` test
6. Add to `store.test.ts`: simulate onChanged for `local` area → assert settings store NOT updated
7. Add to `store.test.ts`: simulate onChanged for `sync` area → assert settings store updated
8. Run `npm run test:unit` → all pass
9. Run `npm run test:coverage` → verify ≥85% on new adapter
10. Update `docs/system-architecture.md`:
    - Storage Schema section: split into sync vs local subsections
    - High-level diagram: optional update to show two storage areas
11. Update `docs/project-roadmap.md`:
    - Phase 2 section: note settings sync delivered standalone via chrome.storage.sync; cloud-vocab sync still requires Firebase decision
12. Manual cross-device test:
    - Sign Chrome into Google account on 2 machines
    - Install extension dev build on both
    - On Device A: change target language, daily goal, LLM provider
    - On Device B: open options page → verify settings match within ~10s
    - Toggle dark mode on B → verify A picks it up
    - Disable Chrome sync on Device A → verify extension still works locally

## Todo List

- [ ] Create sync adapter test file with vitest-chrome mocks
- [ ] Test: sync getItem hit
- [ ] Test: sync getItem miss → local fallback + copy-back
- [ ] Test: getItem both empty
- [ ] Test: setItem success
- [ ] Test: setItem quota exceeded → local fallback
- [ ] Test: removeItem
- [ ] Test: onChanged areaName !== 'sync' ignored for settings
- [ ] Test: onChanged areaName === 'sync' rehydrates settings
- [ ] `npm run test:unit` passes
- [ ] Coverage ≥85% on new adapter
- [ ] Update `docs/system-architecture.md` Storage Schema
- [ ] Update `docs/project-roadmap.md` Phase 2 status note
- [ ] Manual cross-device test executed and documented

## Success Criteria

- [ ] All new tests pass
- [ ] No existing tests broken
- [ ] Coverage maintained or improved
- [ ] Docs reflect new dual-area storage model
- [ ] Manual cross-device sync verified within ~10s
- [ ] Chrome-sync-disabled fallback verified

## Risk Assessment

| Risk | Mitigation |
|------|------------|
| vitest-chrome doesn't mock `chrome.storage.sync` fully | Check existing mock surface in test setup; extend if needed |
| Cross-device manual test requires 2 machines | Alternative: 2 separate Chrome profiles signed into same Google account on one machine |
| Chrome sync interval can be slow (~1min) | Document expected delay; not a bug |

## Security Considerations

- API keys travel via Google sync infra (encrypted at rest by Google, encrypted in transit)
- No new attack surface vs existing chrome.storage.local model
- Confirm `manifest.json` does NOT log/transmit settings outside Chrome sync
