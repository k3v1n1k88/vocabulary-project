---
phase: 1
title: "Update Model Lists + Defensive Fallback"
status: pending
priority: P1
effort: "1h"
dependencies: []
---

# Phase 1: Update Model Lists + Defensive Fallback

## Overview

Replace deprecated model IDs in `llm-provider-config.ts` with current non-deprecated alternatives for all 3 providers. Add a defensive fallback so users with stale stored preferences are redirected to the new default rather than hitting a "model not available" error.

## Requirements

- Functional:
  - Gemini default → `gemini-2.5-flash`; remove all `gemini-2.0-*` and `gemini-1.5-*` entries
  - OpenAI: keep `gpt-4o-mini` default; remove `gpt-4-turbo` (deprecated)
  - Grok: default → `grok-3-mini`; remove `grok-beta`; add `grok-4.3`
  - `getProviderConfig(id)` falls back to `defaultModel` when stored model ID not in provider's list
- Non-functional:
  - No API endpoint URL changes
  - No `ProviderConfig` / `ModelInfo` type changes
  - Backward-compatible: extension continues to work for existing users

## Architecture

`llm-provider-config.ts` is the single source of truth for provider metadata. `translation-service.ts` calls `getProviderConfig(provider)` and reads `.defaultModel`. The stored user preference (`chrome.storage.sync`) may contain an old model ID — the fix intercepts at `getProviderConfig` by validating the stored ID against the models list and falling back if invalid.

Flow for stale preference:
```
chrome.storage.sync → old model ID
  → translation-service reads config.defaultModel (NOT stored ID directly)
```
Wait — stored ID is the provider ID (`openai`/`gemini`/`grok`), NOT the model ID. The model selection within a provider IS stored separately. Check `translation-settings.ts` to confirm where per-provider model choice is persisted and add fallback there if needed.

## Related Code Files

- Modify: `vocabulary-extension/src/shared/llm-provider-config.ts`
- Read: `vocabulary-extension/src/shared/translation-settings.ts` (confirm model storage key)
- Read: `vocabulary-extension/src/options/` (confirm UI uses model IDs from config)

## Implementation Steps

1. Open `vocabulary-extension/src/shared/llm-provider-config.ts`

2. **Gemini block** — replace models array:
   ```ts
   defaultModel: 'gemini-2.5-flash',
   models: [
     { id: 'gemini-2.5-flash', description: 'Latest & fastest - Recommended' },
     { id: 'gemini-2.5-pro',   description: 'Most capable - Complex translations' },
   ]
   ```

3. **OpenAI block** — remove `gpt-4-turbo` entry, keep others unchanged:
   ```ts
   defaultModel: 'gpt-4o-mini',
   models: [
     { id: 'gpt-4o-mini', description: 'Fast & affordable - Best for translations' },
     { id: 'gpt-4o',      description: 'High quality - Better accuracy' },
   ]
   ```

4. **Grok block** — replace models array:
   ```ts
   defaultModel: 'grok-3-mini',
   models: [
     { id: 'grok-3-mini', description: 'Fast & efficient - Good for translations' },
     { id: 'grok-4.3',    description: 'Most capable - Highest quality' },
   ]
   ```

5. **Defensive fallback** — check `translation-settings.ts` for the model storage key. If a per-provider model preference is stored, add a helper to validate and reset:
   ```ts
   // In llm-provider-config.ts or translation-settings.ts
   export function resolveModel(providerId: LLMProvider, storedModel?: string): string {
     const config = LLM_PROVIDERS.find(p => p.id === providerId)!
     if (storedModel && config.models.some(m => m.id === storedModel)) {
       return storedModel
     }
     return config.defaultModel
   }
   ```
   Wire this into the call site in `translation-service.ts` (replace `config.defaultModel` with `resolveModel(provider, storedModelId)`).

## Success Criteria

- [ ] All 3 providers use non-deprecated model IDs as defaults
- [ ] `gpt-4-turbo` and `grok-beta` no longer appear in any model list
- [ ] `resolveModel` returns `defaultModel` when given an unknown model ID
- [ ] TypeScript compiles with no errors (`npm run build` passes in submodule)

## Risk Assessment

| Risk | Mitigation |
|------|-----------|
| `gemini-2.0-flash-lite` still active but excluded | Only include confirmed-stable 2.5-series models |
| `grok-4.3` ID format may differ in API | Verify against xAI docs before committing |
| Stored model preference key unknown | Read `translation-settings.ts` before step 5 |
