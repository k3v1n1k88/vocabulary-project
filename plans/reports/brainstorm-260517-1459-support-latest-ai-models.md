# Brainstorm: Support Latest AI Models (Issue #5)

**Date:** 2026-05-17
**Issue:** [#5 — Support latest AI models](https://github.com/k3v1n1k88/vocabulary-project/issues/5)

---

## Problem Statement

The extension hardcodes AI model IDs in `vocabulary-extension/src/shared/llm-provider-config.ts`. At least one model (`gemini-2.0-flash`) is deprecated/unavailable to new users, causing runtime errors. The same risk exists for other providers.

**Root cause:** Static model lists with no deprecation safety net.

---

## Current State Audit

| Provider | Current Default | Status | Current List |
|----------|----------------|--------|--------------|
| Gemini   | `gemini-2.0-flash` | ❌ Deprecated — shutdown Jun 1 2026 | gemini-2.0-flash, gemini-1.5-flash, gemini-1.5-pro |
| OpenAI   | `gpt-4o-mini` | ✅ Still available in API | gpt-4o-mini, gpt-4o, gpt-4-turbo ⚠️ |
| Grok     | `grok-2` | ⚠️ Legacy — grok-2-1212 still listed; grok-beta removed | grok-2, grok-beta ❌ |

**Verified latest model IDs (May 2026):**
- Gemini: `gemini-2.5-flash` (recommended, fast), `gemini-2.5-pro` (capable)
- OpenAI: `gpt-4o-mini` (stable default), `gpt-4o` (API still active), remove `gpt-4-turbo` (deprecated)
- Grok: `grok-3-mini` (available), `grok-4.3` (latest flagship, released May 6 2026)

---

## Evaluated Approaches

### Approach A — Simple Model List Update (KISS) ✅ Recommended

Update `llm-provider-config.ts` with current, non-deprecated model IDs and descriptions.

**Pros:**
- Minimal diff, zero risk of regression
- No new dependencies or architectural changes
- Fixes the immediate error completely
- Aligns with KISS/YAGNI

**Cons:**
- Models will need manual updates again when providers deprecate further
- No automated deprecation detection

### Approach B — Dynamic Model Discovery

Fetch available models from each provider's API at extension startup (e.g., `GET /v1/models` for OpenAI/Grok, `GET /v1beta/models` for Gemini). Cache results in `chrome.storage.local`.

**Pros:**
- Self-healing — never shows a deprecated model
- Future-proof

**Cons:**
- Requires an API key to list models (chicken-and-egg for first-time users)
- Adds network dependency at startup; failure modes complex
- Heavy for a translation widget (YAGNI — models change rarely)
- Each provider has different list-models API shape → more surface area

### Approach C — Versioned Pinning + Deprecation Warnings

Pin specific dated model snapshots (e.g., `gemini-2.5-flash-001`) and add a `deprecationDate` field to `ModelInfo`. Show an in-UI warning when a model nears deprecation.

**Pros:**
- More stable than floating model aliases
- Deprecation visible to users

**Cons:**
- Significantly more UI and logic work
- Providers don't reliably publish deprecation dates ahead of time
- YAGNI — over-engineered for current scale

---

## Recommended Solution: Approach A

**Rationale:** This is a data correction, not an architectural problem. The extension has 3 providers with ~3 models each — a simple map update resolves the issue immediately. Approach B adds meaningful complexity for a problem that occurs infrequently (model lifecycle is months, not days). Approach C is premature optimization.

### Concrete Changes to `llm-provider-config.ts`

```
Gemini:
  defaultModel: 'gemini-2.5-flash'
  models:
    - { id: 'gemini-2.5-flash', description: 'Latest & fastest - Recommended' }
    - { id: 'gemini-2.5-pro',   description: 'Most capable - Complex translations' }
    - { id: 'gemini-2.0-flash-lite', description: 'Lightweight - Low cost' }  // still active

OpenAI:
  defaultModel: 'gpt-4o-mini'  (unchanged — still current)
  models:
    - { id: 'gpt-4o-mini', description: 'Fast & affordable - Best for translations' }
    - { id: 'gpt-4o',      description: 'High quality - Better accuracy' }
    - REMOVE gpt-4-turbo (deprecated)

Grok:
  defaultModel: 'grok-3-mini'
  models:
    - { id: 'grok-3-mini', description: 'Fast & efficient - Good for translations' }
    - { id: 'grok-4.3',    description: 'Most capable - Highest quality' }
    - REMOVE grok-beta (experimental / retired)
```

### Implementation Scope

- **File:** `vocabulary-extension/src/shared/llm-provider-config.ts` (sole change)
- **Tests:** `translation-service.test.ts` — verify model IDs in mock configs pass through correctly
- **No endpoint changes** — API URLs remain the same for all providers
- **No type changes** — `ProviderConfig`/`ModelInfo` shapes unchanged

---

## Implementation Considerations

- Gemini API endpoint (`/v1beta/models`) works with any model string — the URL is `{endpoint}/{model}:generateContent`; only the model segment changes
- OpenAI and Grok use the same request builder (`buildOpenAIRequest`) — model is passed as request body field; no builder changes needed
- Users who already saved a deprecated model as their preference in `chrome.storage.sync` will get the stored (deprecated) ID on next use → consider defaulting to new model if stored ID is no longer in the list (defensive fallback in `getProviderConfig`)

---

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| `gemini-2.0-flash-lite` also deprecated by Jun 2026 | Use `gemini-2.5-flash` as sole safe default; drop 2.0-series entirely |
| grok-4.3 ID format changes | Confirm against xAI docs before commit |
| Existing user stored preferences reference old IDs | Add fallback in `getProviderConfig` to return `defaultModel` if stored ID not in list |

---

## Success Criteria

1. No "model not available" errors for new or existing users
2. All 3 providers successfully complete test connection with updated defaults
3. Extension options UI shows valid, current model names
4. No regressions in `translation-service.test.ts`

---

## Next Steps

1. Update `llm-provider-config.ts` model lists per the concrete changes above
2. Add defensive fallback in `getProviderConfig` for stale user preferences
3. Run existing translation tests to confirm no breakage
4. Manual QA: test-connect each provider with new default model
