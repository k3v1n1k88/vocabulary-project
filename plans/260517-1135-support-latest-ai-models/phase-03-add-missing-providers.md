---
phase: 3
title: "Add OpenRouter, Groq, Mistral Providers"
status: pending
priority: P2
effort: "1.5d"
dependencies: [1]
---

# Phase 03: Add OpenRouter, Groq, Mistral Providers

## Overview

Add the 3 providers mentioned in the README but not yet implemented. All three use the OpenAI-compatible `/v1/chat/completions` format, so they reuse `buildOpenAIRequest`. The main work is extending the `LLMProvider` type union, adding configs, and updating `buildProviderRequest`'s type signature.

## Requirements

- Functional:
  - OpenRouter, Groq, Mistral selectable in Options → AI Translation
  - Each provider has correct endpoint, default model, model list, and API key storage key
  - `testConnection` works for all three providers
  - `translateText` works for all three providers
  - Each provider's API key stored under a distinct `chrome.storage.sync` key
- Non-functional:
  - No new request builder functions — all three reuse `buildOpenAIRequest`
  - `buildProviderRequest` type signature updated to accept full `LLMProvider` union
  - `npx tsc --noEmit` passes

## Architecture

All three providers use OpenAI-compatible chat completions format:

```
LLM_PROVIDERS registry (llm-provider-config.ts)
  openrouter → endpoint: https://openrouter.ai/api/v1/chat/completions
  groq       → endpoint: https://api.groq.com/openai/v1/chat/completions
  mistral    → endpoint: https://api.mistral.ai/v1/chat/completions

buildProviderRequest(provider, ...)
  provider === 'gemini' → buildGeminiRequest(...)
  else                  → buildOpenAIRequest(...)   ← catches all 5 OpenAI-compat providers
```

The only structural change beyond config data is the `LLMProvider` type and `buildProviderRequest` signature.

## Related Code Files

- Modify: `vocabulary-extension/src/types/index.ts` — extend `LLMProvider` union
- Modify: `vocabulary-extension/src/shared/llm-provider-config.ts` — add 3 provider entries
- Modify: `vocabulary-extension/src/shared/llm-request-builders.ts` — widen type signature
- Modify: `vocabulary-extension/src/shared/translation-settings.ts` — add 3 storage key mappings

## Implementation Steps

### 1. Extend `LLMProvider` type — `types/index.ts:55`

```ts
export type LLMProvider = 'openai' | 'gemini' | 'grok' | 'openrouter' | 'groq' | 'mistral'
```

### 2. Add provider entries to `llm-provider-config.ts`

Append after the Grok entry:

```ts
{
  id: 'openrouter',
  name: 'OpenRouter',
  endpoint: 'https://openrouter.ai/api/v1/chat/completions',
  defaultModel: 'meta-llama/llama-3.3-70b-instruct',
  models: [
    { id: 'meta-llama/llama-3.3-70b-instruct', description: 'Llama 3.3 70B — fast & capable' },
    { id: 'google/gemini-2.5-flash-preview',    description: 'Gemini via OpenRouter' },
    { id: 'openai/gpt-4o-mini',                 description: 'GPT-4o Mini via OpenRouter' },
  ],
  authType: 'bearer',
  apiKeyStorageKey: 'openrouterApiKey',
  registerUrl: 'https://openrouter.ai/keys'
},
{
  id: 'groq',
  name: 'Groq',
  endpoint: 'https://api.groq.com/openai/v1/chat/completions',
  defaultModel: 'llama-3.3-70b-versatile',
  models: [
    { id: 'llama-3.3-70b-versatile', description: 'Llama 3.3 70B — recommended' },
    { id: 'llama3-8b-8192',          description: 'Llama 3 8B — fastest' },
    { id: 'mixtral-8x7b-32768',      description: 'Mixtral 8x7B — multilingual' },
  ],
  authType: 'bearer',
  apiKeyStorageKey: 'groqApiKey',
  registerUrl: 'https://console.groq.com/keys'
},
{
  id: 'mistral',
  name: 'Mistral',
  endpoint: 'https://api.mistral.ai/v1/chat/completions',
  defaultModel: 'mistral-small-latest',
  models: [
    { id: 'mistral-small-latest',  description: 'Small — fast & cheap' },
    { id: 'mistral-medium-latest', description: 'Medium — balanced' },
    { id: 'mistral-large-latest',  description: 'Large — best quality' },
  ],
  authType: 'bearer',
  apiKeyStorageKey: 'mistralApiKey',
  registerUrl: 'https://console.mistral.ai/api-keys'
},
```

### 3. Widen `buildProviderRequest` signature — `llm-request-builders.ts:101`

```ts
import type { LLMProvider } from '../types'

export function buildProviderRequest(
  provider: LLMProvider,   // was: 'openai' | 'grok' | 'gemini'
  endpoint: string,
  model: string,
  apiKey: string,
  system: string,
  user: string
): { url: string; options: RequestInit } {
  return provider === 'gemini'
    ? buildGeminiRequest(endpoint, model, apiKey, system, user)
    : buildOpenAIRequest(endpoint, model, apiKey, system, user)
}
```

No other changes needed — the `else` branch already handles all OpenAI-compatible providers.

### 4. Add storage key mappings — `translation-settings.ts`

Verify `getApiKey` uses `config.apiKeyStorageKey` dynamically (likely already does via `getProviderConfig`). If it's a hard-coded switch, extend it to cover the 3 new keys. Check current implementation before editing.

### 5. Verify compilation

```bash
cd vocabulary-extension && npx tsc --noEmit
```

### 6. Manual smoke test

With a valid API key for each new provider, open Options → AI Translation, select the provider, enter the key, click Test Connection. Verify it succeeds.

## Success Criteria

- [ ] `LLMProvider` type includes `'openrouter' | 'groq' | 'mistral'`
- [ ] All 3 new providers appear in the Options provider dropdown
- [ ] `testConnection` succeeds for OpenRouter with a valid key
- [ ] `testConnection` succeeds for Groq with a valid key
- [ ] `testConnection` succeeds for Mistral with a valid key
- [ ] `buildProviderRequest` accepts any `LLMProvider` value without TypeScript error
- [ ] `npx tsc --noEmit` exits 0

## Risk Assessment

- **OpenRouter model IDs drift frequently:** OpenRouter aggregates models from many providers; specific model IDs change. Mitigation: use well-known stable IDs (Llama 3.3); the user-selectable model dropdown (Phase 2) lets users override without a code change.
- **Groq `mixtral-8x7b-32768` may be deprecated:** Groq's model availability changes. Mitigation: `llama-3.3-70b-versatile` is the primary default; others are options.
- **API key storage key naming:** new keys (`openrouterApiKey`, etc.) must not conflict with existing keys. Verify against `SettingsData` interface before shipping.
