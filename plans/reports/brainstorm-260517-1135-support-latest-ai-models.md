# Brainstorm: Support Latest AI Models (Issue #3)

**Date:** 2026-05-17  
**Issue:** [#3 — Support latest AI models](https://github.com/k3v1n1k88/vocabulary-project/issues/3)  
**Status:** Complete

---

## Problem Statement

`gemini-2.0-flash` is deprecated for new users; the error surfaces directly to users during translation. The extension has no mechanism to detect deprecated model IDs or guide users to alternatives. Additionally, the 3 implemented providers (OpenAI, Gemini, Grok) haven't been audited for stale model IDs, and 3 more providers mentioned in docs (OpenRouter, Groq, Mistral) aren't implemented yet.

---

## Codebase Findings

| File | Finding |
|------|---------|
| `src/types/index.ts:55` | `LLMProvider = 'openai' \| 'gemini' \| 'grok'` — only 3 providers |
| `src/shared/llm-provider-config.ts` | Gemini `defaultModel: 'gemini-2.0-flash'` — **deprecated** |
| `src/shared/llm-provider-config.ts` | Grok: `grok-2`, `grok-beta` — `grok-beta` alias ambiguous |
| `src/shared/llm-request-builders.ts:101` | `buildProviderRequest` type-locked to 3 providers; OpenRouter/Groq/Mistral would break |
| `src/shared/translation-service.ts:139` | Surfaces raw API error message to user — no model-deprecation detection |

---

## Evaluated Approaches

### Option A — Minimal: Update model IDs only *(RECOMMENDED)*

**What:** Update stale model IDs in `llm-provider-config.ts` only. No new providers.

- Gemini `defaultModel` → `gemini-2.0-flash-lite` (GA, free tier) or `gemini-2.5-flash-preview-05-20` (latest preview)
- Remove deprecated `gemini-2.0-flash` from models list, replace with current stable IDs
- Grok: replace `grok-beta` with `grok-2-1212` (stable alias)
- OpenAI: drop `gpt-4-turbo` (legacy); add `gpt-4o-mini` as default (already correct), optionally add `gpt-4.1-mini`
- Add a dedicated `MODEL_DEPRECATION_ERRORS` pattern array in `translation-service.ts` to detect deprecation messages and surface a cleaner "Please update your model in Settings" prompt

**Pros:** Tiny diff, no architecture change, ships fast, directly fixes the reported issue  
**Cons:** Doesn't add missing providers; models will drift again eventually

---

### Option B — Medium: Add user-selectable model field + 3 missing providers

**What:** Option A + surface model selection per-provider in the Options UI + add OpenRouter/Groq/Mistral.

- Add `llmModel` to `SettingsData` (already typed at `index.ts:87` as optional — just wire it up)
- Options UI gets a model dropdown per provider
- `translation-service.ts` reads `llmModel` from storage, falls back to `defaultModel`
- Add OpenRouter, Groq, Mistral to `LLM_PROVIDERS` config and extend `LLMProvider` type
- `buildProviderRequest` already handles OpenAI-compatible APIs; OpenRouter/Groq/Mistral all use the same `/v1/chat/completions` format → reuse `buildOpenAIRequest`

**Pros:** Users self-serve model updates; adds providers promised by README  
**Cons:** More scope, UI work needed, 3x test surface

---

### Option C — Full: Dynamic model list fetching

**What:** Fetch available models from each provider's API at settings-open time and populate dropdowns dynamically.

- Hit `/v1/models` (OpenAI-compatible) or Gemini models endpoint, cache results in `chrome.storage.session`
- Always shows current models without code changes

**Pros:** Never stale  
**Cons:** Requires API key before listing models; CORS/fetch permissions need auditing; significant complexity for marginal gain; all providers have different model list endpoints/formats

**Verdict:** Over-engineered for the immediate problem. Revisit in a later phase.

---

## Recommended Solution

**Option B** — fixes the immediate deprecation bug (Option A core) while also:
1. Wiring up the already-typed `llmModel` field so users can self-serve future model updates
2. Adding the 3 missing providers to align code with README claims

**Phasing:**
- **Phase 1 (critical):** Update stale model IDs in `llm-provider-config.ts`; add deprecation-error detection in `translation-service.ts` → 1 day
- **Phase 2 (UX):** Wire `llmModel` setting into Options UI model dropdown → 1 day
- **Phase 3 (providers):** Add OpenRouter, Groq, Mistral configs + extend `LLMProvider` type + tests → 2 days

---

## Model ID Audit (Current State)

| Provider | Current Default | Status | Recommended Default |
|----------|----------------|--------|---------------------|
| OpenAI | `gpt-4o-mini` | ✅ Active | Keep |
| OpenAI | `gpt-4o` | ✅ Active | Keep |
| OpenAI | `gpt-4-turbo` | ⚠️ Legacy | Replace with `gpt-4.1` |
| Gemini | `gemini-2.0-flash` | ❌ Deprecated (new users) | `gemini-2.5-flash-preview-05-20` |
| Gemini | `gemini-1.5-flash` | ✅ Active | Keep |
| Gemini | `gemini-1.5-pro` | ✅ Active | Keep |
| Grok | `grok-2` | ⚠️ Old alias | `grok-2-1212` |
| Grok | `grok-beta` | ⚠️ Deprecated alias | `grok-2-latest` |

---

## Deprecation Error Detection

Add to `translation-service.ts` before throwing:

```ts
const DEPRECATION_PATTERNS = [
  'no longer available',
  'deprecated',
  'model not found',
  'model_not_found',
]

function isDeprecationError(msg: string): boolean {
  return DEPRECATION_PATTERNS.some(p => msg.toLowerCase().includes(p))
}
```

If `isDeprecationError(errorMsg)` → throw with: `"The selected model is no longer available. Go to Settings → AI Translation and choose a different model."`

---

## Implementation Considerations

- `buildProviderRequest` signature currently typed as `provider: 'openai' | 'grok' | 'gemini'` — must update to accept full `LLMProvider` union when adding providers
- OpenRouter/Groq/Mistral all use OpenAI-compatible chat completion format → route through existing `buildOpenAIRequest` with their respective endpoints
- Gemini uses a different request format (`generateContent`) — keep `buildGeminiRequest` as-is
- `testConnection` in `translation-service.ts` uses `config.defaultModel` — ensure new defaults are verified against provider sandbox keys before shipping
- Existing unit test in `translation-service.test.ts` should cover the new deprecation detection logic

---

## Success Criteria

- [ ] `gemini-2.0-flash` removed from model list; valid Gemini 2.x model works end-to-end
- [ ] All 6 providers configured and testable
- [ ] Deprecation error surfaces a user-actionable message (not raw API error)
- [ ] User can select model per-provider in Settings UI
- [ ] All existing tests pass; new deprecation-detection tests added
- [ ] `LLMProvider` type union updated; TypeScript compiles cleanly

---

## Risks

| Risk | Mitigation |
|------|-----------|
| Gemini preview model may itself change | Use `gemini-2.5-flash-preview-05-20` pinned ID; also offer `gemini-1.5-flash` as stable fallback |
| New providers require new API keys in user settings | Each new provider is opt-in; no key = fallback to free API |
| Model dropdown adds UI complexity | Single dropdown component, reusable across all providers |

---

## Open Questions

1. Should `gemini-2.5-flash-preview-05-20` be the new default (latest, best) or should `gemini-1.5-flash` be the safe default (stable GA)? Preview models could also be deprecated faster.
2. Is the README's mention of 6 providers intentional scope (Phase 3 target) or documentation drift? Confirm before adding all 3 missing providers in this issue's scope.
3. For Grok, the correct stable alias — is it `grok-2-1212` or `grok-2-latest`? Need to verify against xAI API docs.
