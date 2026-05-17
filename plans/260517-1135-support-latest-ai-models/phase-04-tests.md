---
phase: 4
title: "Tests"
status: pending
priority: P2
effort: "1d"
dependencies: [1, 2, 3]
---

# Phase 04: Tests

## Overview

Add unit tests covering deprecation-error detection, model selection fallback logic, and the 3 new provider configs. Extend the existing `translation-service.test.ts` — no new test file needed.

## Requirements

- Functional:
  - `isDeprecationError` (Phase 01) tested with true/false cases
  - `getSelectedModel` (Phase 02) tested: stored value returned, fallback to defaultModel
  - New provider configs (Phase 03) validated: correct endpoint, authType, model count
  - Existing tests continue to pass
- Non-functional:
  - Tests run via `npm run test:unit`
  - No real API calls — mock `chrome.storage.sync.get`

## Related Code Files

- Modify: `vocabulary-extension/src/shared/translation-service.test.ts`

## Implementation Steps

### 1. Deprecation detection tests

```ts
describe('isDeprecationError', () => {
  // Note: isDeprecationError is not exported — test via translateText mock
  // or export it for direct unit testing
  it('returns true for "no longer available"', ...)
  it('returns true for "model not found"', ...)
  it('returns true for "deprecated"', ...)
  it('returns false for "rate limit exceeded"', ...)
  it('returns false for "invalid api key"', ...)
})
```

If `isDeprecationError` is not exported, either:
a. Export it (preferred — pure function, no side effects), or
b. Test via `translateText` with a mocked fetch that returns the error message

### 2. `getSelectedModel` fallback tests

```ts
describe('getSelectedModel', () => {
  it('returns stored llmModel when set', async () => {
    mockChromeStorage({ llmModel: 'gpt-4o' })
    expect(await getSelectedModel('openai')).toBe('gpt-4o')
  })

  it('falls back to defaultModel when llmModel not set', async () => {
    mockChromeStorage({})
    expect(await getSelectedModel('openai')).toBe('gpt-4o-mini')
  })

  it('falls back to provider defaultModel regardless of stored value type', async () => {
    mockChromeStorage({ llmModel: undefined })
    expect(await getSelectedModel('gemini')).toBe('gemini-2.5-flash-preview-05-20')
  })
})
```

### 2b. Chrome mock availability

`settings-storage-access.test.ts` calls `chrome.storage.sync.set/get` directly and passes — confirming vitest-chrome is globally configured. `getSelectedModel` tests can call `patchSettings({ llmModel: 'gpt-4o' })` to seed state, then call `getSelectedModel('openai')` and assert the result. No additional mock setup needed.

### 3. Provider config validation tests

```ts
describe('LLM_PROVIDERS config', () => {
  const providerIds = ['openai', 'gemini', 'grok', 'openrouter', 'groq', 'mistral']

  it('has an entry for all 6 providers', () => {
    expect(LLM_PROVIDERS.map(p => p.id)).toEqual(expect.arrayContaining(providerIds))
  })

  it.each(providerIds)('%s has a non-empty models list', (id) => {
    const config = getProviderConfig(id as LLMProvider)
    expect(config.models.length).toBeGreaterThan(0)
  })

  it.each(providerIds)('%s defaultModel exists in its models list', (id) => {
    const config = getProviderConfig(id as LLMProvider)
    expect(config.models.map(m => m.id)).toContain(config.defaultModel)
  })

  it('gemini defaultModel is not gemini-2.0-flash', () => {
    const gemini = getProviderConfig('gemini')
    expect(gemini.defaultModel).not.toBe('gemini-2.0-flash')
    expect(gemini.models.map(m => m.id)).not.toContain('gemini-2.0-flash')
  })
})
```

### 4. Run tests

```bash
cd vocabulary-extension && npm run test:unit
```

Fix any failures before marking phase complete.

## Success Criteria

- [ ] Deprecation detection: ≥5 test cases (3 true, 2 false)
- [ ] `getSelectedModel`: 3 test cases (stored, missing, undefined)
- [ ] Provider config: all 6 providers validated for models list + defaultModel presence
- [ ] `gemini-2.0-flash` absence explicitly asserted
- [ ] `npm run test:unit` exits 0 with no skipped tests

## Risk Assessment

- **`isDeprecationError` not exported:** If it's private, exporting it is low-risk (pure function). Alternatively, integration-test it through a mocked `translateText` call.
- **Chrome storage mock scope:** Existing test setup in `translation-service.test.ts` likely already mocks `chrome.storage` via `vitest-chrome`. Reuse the existing mock pattern.
