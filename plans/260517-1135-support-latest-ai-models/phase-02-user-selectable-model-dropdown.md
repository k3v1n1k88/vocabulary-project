---
phase: 2
title: "User-Selectable Model Dropdown"
status: pending
priority: P2
effort: "1d"
dependencies: [1]
---

# Phase 02: User-Selectable Model Dropdown

## Overview

Wire the already-typed `llmModel` field in `SettingsData` into the Options UI so users can pick a model per-provider without editing code. `translation-service.ts` reads the stored model, falling back to `defaultModel` when unset.

## Requirements

- Functional:
  - Options → AI Translation shows a model dropdown when a provider is selected
  - Dropdown options are sourced from `LLM_PROVIDERS[provider].models`
  - Selected model saved to `chrome.storage.sync` via existing settings infrastructure
  - `translateText` and `testConnection` use stored `llmModel` (fallback: `defaultModel`)
  - Switching provider resets model selection to that provider's `defaultModel`
- Non-functional:
  - No new storage keys; reuse `llmModel` already in `SettingsData` (`types/index.ts:87`)
  - Dropdown accessible (label + keyboard navigation via native `<select>`)

## Architecture

```
Options UI (translation-settings.tsx)
  └── model <select> ← models from getProviderConfig(selectedProvider).models
         ↓ onChange
  saveSettings({ llmModel: value })   ← chrome.storage.sync

translation-service.ts
  getSelectedModel() → chrome.storage.sync.get('llmModel') || config.defaultModel
  buildProviderRequest(provider, endpoint, selectedModel, ...)
```

No new storage helpers needed — `llmModel` is already part of `SettingsData`.

## Related Code Files

- Modify: `vocabulary-extension/src/shared/translation-settings.ts` — add `getSelectedModel()`
- Modify: `vocabulary-extension/src/shared/translation-service.ts` — use `getSelectedModel()` instead of `config.defaultModel`
- Modify: `vocabulary-extension/src/options/components/translation-settings.tsx` — add model dropdown

## Implementation Steps

### 1. Add `getSelectedModel` to `translation-settings.ts`

```ts
export async function getSelectedModel(provider: LLMProvider): Promise<string> {
  const config = getProviderConfig(provider)
  const result = await chrome.storage.sync.get('llmModel')
  return (result.llmModel as string) || config.defaultModel
}
```

### 2. Update `translation-service.ts`

In `testConnection`, replace:
```ts
config.defaultModel
```
With:
```ts
await getSelectedModel(provider)
```

In `translateText`, replace:
```ts
config.defaultModel
```
With:
```ts
const selectedModel = await getSelectedModel(provider)
```
Then pass `selectedModel` to `buildProviderRequest`.

Import `getSelectedModel` from `./translation-settings`.

### 3. Update Options UI — `translation-settings.tsx`

After the provider `<select>`, add a model `<select>`:

```tsx
const providerConfig = getProviderConfig(settings.llmProvider)

<div>
  <label htmlFor="model-select">Model</label>
  <select
    id="model-select"
    value={settings.llmModel || providerConfig.defaultModel}
    onChange={e => updateSetting('llmModel', e.target.value)}
  >
    {providerConfig.models.map(m => (
      <option key={m.id} value={m.id}>{m.id} — {m.description}</option>
    ))}
  </select>
</div>
```

When provider changes, also reset model:
```ts
onChange={e => {
  const newProvider = e.target.value as LLMProvider
  const newDefault = getProviderConfig(newProvider).defaultModel
  updateSetting('llmProvider', newProvider)
  updateSetting('llmModel', newDefault)
}}
```

### 4. Verify compilation

```bash
cd vocabulary-extension && npx tsc --noEmit
```

## Success Criteria

- [ ] `getSelectedModel()` returns stored model when set, `defaultModel` when not
- [ ] Model dropdown renders with correct options for the active provider
- [ ] Changing provider resets model dropdown to that provider's default
- [ ] `translateText` uses stored model (verifiable via network tab in DevTools)
- [ ] `npx tsc --noEmit` exits 0

## Risk Assessment

- **`llmModel` persists across provider switch:** If user had `gpt-4o` stored and switches to Gemini, the old value could be passed to Gemini. Mitigation: reset `llmModel` to `defaultModel` on provider change (step 3 above).
- **Storage key collision:** `llmModel` is already in `SettingsData` type but may not be saved by existing settings save path. Verify `saveSettings` serializes it — if not, add it explicitly.
