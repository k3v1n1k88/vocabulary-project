---
phase: 1
title: "Update Model IDs + Deprecation Detection"
status: pending
priority: P1
effort: "0.5d"
dependencies: []
---

# Phase 01: Update Model IDs + Deprecation Detection

## Overview

Fix the immediate bug: replace deprecated/stale model IDs in the provider registry and add pattern-based deprecation-error detection that surfaces an actionable user message instead of a raw API error.

## Requirements

- Functional:
  - `gemini-2.0-flash` removed; replaced with current stable/preview models
  - `gpt-4-turbo` replaced with `gpt-4.1`
  - Grok `grok-beta` replaced with a stable alias
  - When a provider returns a deprecation error, user sees: "The selected model is no longer available. Go to Settings → AI Translation and choose a different model."
- Non-functional:
  - TypeScript compiles cleanly (`npx tsc --noEmit` passes)
  - No changes to API request format or auth flow

## Architecture

Detection runs inside the existing `catch` block in `translateText` and `testConnection` in `translation-service.ts`. A small pure helper `isDeprecationError(msg: string): boolean` checks the message against a constant pattern list. No new modules needed — the helper lives in `translation-service.ts` alongside the error-handling code.

```
API error response
       ↓
translation-service.ts → parseErrorMessage() → isDeprecationError()
       ↓ yes                                         ↓ no
"Model no longer available…"               original error message re-thrown
```

## Related Code Files

- Modify: `vocabulary-extension/src/shared/llm-provider-config.ts`
- Modify: `vocabulary-extension/src/shared/translation-service.ts`

## Implementation Steps

### 1. Update `llm-provider-config.ts`

**Gemini** — replace entire `models` array and `defaultModel`:
```ts
defaultModel: 'gemini-2.5-flash-preview-05-20',
models: [
  { id: 'gemini-2.5-flash-preview-05-20', description: 'Latest preview — best quality' },
  { id: 'gemini-2.0-flash-lite',          description: 'Fast & free-tier friendly' },
  { id: 'gemini-1.5-flash',               description: 'Stable GA — reliable fallback' },
  { id: 'gemini-1.5-pro',                 description: 'Most capable — complex translations' },
]
```
Remove `gemini-2.0-flash` entirely.

**OpenAI** — replace `gpt-4-turbo` entry:
```ts
{ id: 'gpt-4.1', description: 'Balanced — good quality & speed' }
```
Keep `gpt-4o-mini` (default) and `gpt-4o`.

**Grok** — replace both entries:
```ts
defaultModel: 'grok-2-latest',
models: [
  { id: 'grok-2-latest', description: 'Latest stable — best quality' },
  { id: 'grok-2-1212',   description: 'Pinned version — predictable behavior' },
]
```
Remove `grok-2` and `grok-beta`.

### 2. Add deprecation detection to `translation-service.ts`

Add after existing imports, before `testConnection`:

```ts
const DEPRECATION_PATTERNS = [
  'no longer available',
  'deprecated',
  'model not found',
  'model_not_found',
  'does not exist',
] as const

function isDeprecationError(msg: string): boolean {
  const lower = msg.toLowerCase()
  return DEPRECATION_PATTERNS.some(p => lower.includes(p))
}
```

In both `testConnection` and `translateText`, inside the `if (!response.ok)` block, replace:
```ts
throw new Error(errorMsg || `${config.name} API error: ${response.status}`)
```
With:
```ts
const finalMsg = errorMsg || `${config.name} API error: ${response.status}`
if (isDeprecationError(finalMsg)) {
  throw new Error(
    `The selected model is no longer available. Go to Settings → AI Translation and choose a different model.`
  )
}
throw new Error(finalMsg)
```

### 3. Verify compilation

```bash
cd vocabulary-extension && npx tsc --noEmit
```

## Success Criteria

- [ ] `gemini-2.0-flash` does not appear anywhere in `llm-provider-config.ts`
- [ ] `gpt-4-turbo` and `grok-beta` removed from config
- [ ] `isDeprecationError` returns `true` for "no longer available", "model not found", "deprecated"
- [ ] `isDeprecationError` returns `false` for "rate limit exceeded", "invalid api key"
- [ ] `npx tsc --noEmit` exits 0

## Risk Assessment

- **Gemini preview model deprecation risk:** `gemini-2.5-flash-preview-05-20` is a preview — could itself be deprecated. Mitigation: include `gemini-1.5-flash` as a stable fallback option; document in code comment.
- **Pattern false-positives:** "model not found" could theoretically match a non-deprecation error. Mitigation: keep patterns conservative; the fallback behavior (user goes to Settings) is always safe.
