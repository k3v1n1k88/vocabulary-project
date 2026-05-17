---
phase: 2
title: "Verify and Update Tests"
status: pending
priority: P2
effort: "30m"
dependencies: [1]
---

# Phase 2: Verify and Update Tests

## Overview

Run existing tests to confirm no regressions after model list changes. Update any test fixtures that reference deprecated model IDs.

## Requirements

- Functional:
  - All existing translation tests pass
  - Any hardcoded model IDs in test fixtures updated to match new defaults
- Non-functional:
  - No new test infrastructure required

## Related Code Files

- Read/Modify: `vocabulary-extension/src/shared/translation-service.test.ts`
- Read: `vocabulary-extension/src/shared/llm-provider-config.ts` (confirm final model IDs)

## Implementation Steps

1. Run the test suite from the submodule:
   ```bash
   cd vocabulary-extension && npm test -- --run 2>&1
   ```

2. Search for deprecated model IDs in test files:
   ```bash
   grep -r "gemini-2.0-flash\|gemini-1.5\|gpt-4-turbo\|grok-beta" vocabulary-extension/src --include="*.test.ts"
   ```

3. For each match, replace the deprecated ID with the new provider default from Phase 1.

4. If `resolveModel` was added in Phase 1, add a unit test for it:
   - Known model ID → returns that ID
   - Unknown model ID → returns provider `defaultModel`
   - Empty/undefined → returns provider `defaultModel`

5. Re-run tests to confirm all pass:
   ```bash
   cd vocabulary-extension && npm test -- --run 2>&1
   ```

## Success Criteria

- [ ] `npm test -- --run` exits with code 0
- [ ] No test references deprecated model IDs (`gemini-2.0-flash`, `gpt-4-turbo`, `grok-beta`)
- [ ] `resolveModel` fallback logic covered by at least 3 test cases (if added)

## Risk Assessment

| Risk | Mitigation |
|------|-----------|
| Test mocks hardcode specific model IDs | grep scan in step 2 catches all occurrences |
| `resolveModel` not needed (model stored at provider level only) | Skip step 4 if Phase 1 confirms no per-model storage |
