---
phase: 1
title: "Update Model Lists"
status: pending
priority: P1
effort: "30m"
dependencies: []
---

# Phase 1: Update Model Lists

<!-- Updated: Validation Session 1 - Removed resolveModel step; translation-service.ts always uses config.defaultModel and never reads settings.llmModel -->

## Overview

Replace deprecated model IDs in `llm-provider-config.ts` with current non-deprecated alternatives for all 3 providers. No service-layer changes needed — `translation-service.ts` already reads `config.defaultModel` directly and ignores any stored per-model preference.

## Requirements

- Functional:
  - Gemini default → `gemini-2.5-flash`; remove all `gemini-2.0-*` and `gemini-1.5-*` entries
  - OpenAI: keep `gpt-4o-mini` default; remove `gpt-4-turbo` (deprecated)
  - Grok: default → `grok-3-mini`; remove `grok-beta`; add `grok-4.3`
- Non-functional:
  - No API endpoint URL changes
  - No `ProviderConfig` / `ModelInfo` type changes
  - No changes to `translation-service.ts` or `translation-settings.ts`

## Architecture

`llm-provider-config.ts` is the sole change. `translation-service.ts` calls `getProviderConfig(provider)` and reads `.defaultModel` directly — it does NOT read `settings.llmModel` from storage. The options UI uses `settings.llmModel || currentProvider.defaultModel` for display; stale stored values fall back gracefully to the new default without code changes.

## Related Code Files

- Modify: `vocabulary-extension/src/shared/llm-provider-config.ts`

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

## Success Criteria

- [ ] All 3 providers use non-deprecated model IDs as defaults
- [ ] `gpt-4-turbo` and `grok-beta` no longer appear in any model list
- [ ] TypeScript compiles with no errors (`npm run build` in `vocabulary-extension/`)

## Risk Assessment

| Risk | Mitigation |
|------|-----------|
| `grok-4.3` ID format may differ in actual xAI API | Verify against xAI docs / `docs.x.ai/developers/models` before committing |
| `gemini-2.0-flash-lite` still active but excluded | Intentional — only 2.5-series confirmed stable long-term |
