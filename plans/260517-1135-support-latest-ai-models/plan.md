---
title: "Support Latest AI Models (Issue #3)"
description: "Update deprecated model IDs, add deprecation-error detection, wire user-selectable models, add 3 missing providers"
status: pending
priority: P2
effort: 4d
branch: feature/3-support-latest-ai-models
issue: https://github.com/k3v1n1k88/vocabulary-project/issues/3
brainstorm: ../reports/brainstorm-260517-1135-support-latest-ai-models.md
tags: [ai, llm, providers, gemini, openai, grok, openrouter, groq, mistral]
created: 2026-05-17
blockedBy: []
blocks: []
---

# Plan — Support Latest AI Models (Issue #3)

## Context

- GitHub issue: https://github.com/k3v1n1k88/vocabulary-project/issues/3
- Brainstorm: [../reports/brainstorm-260517-1135-support-latest-ai-models.md](../reports/brainstorm-260517-1135-support-latest-ai-models.md)
- Submodule (code root): `vocabulary-extension/`
- Trigger: `gemini-2.0-flash` deprecated for new users → raw API error surfaced with no guidance

## Problem Summary

1. `gemini-2.0-flash` hardcoded as default — now deprecated for new users
2. Other providers (Grok `grok-beta`, OpenAI `gpt-4-turbo`) have stale model IDs
3. No deprecation-error detection — raw provider error shown to user
4. 3 providers (OpenRouter, Groq, Mistral) listed in README but not implemented

## Solution Overview

Three-phase delivery:

| Phase | Title | Effort | Status |
|-------|-------|--------|--------|
| [01](./phase-01-update-model-ids-and-deprecation-detection.md) | Update Model IDs + Deprecation Detection | 0.5d | pending |
| [02](./phase-02-user-selectable-model-dropdown.md) | User-Selectable Model Dropdown | 1d | pending |
| [03](./phase-03-add-missing-providers.md) | Add OpenRouter, Groq, Mistral | 1.5d | pending |
| [04](./phase-04-tests.md) | Tests | 1d | pending |

## Key Files

```
vocabulary-extension/src/
├── types/index.ts                              # LLMProvider union, ProviderConfig, SettingsData
├── shared/
│   ├── llm-provider-config.ts                 # Provider + model registry
│   ├── llm-request-builders.ts                # Provider request factories
│   ├── translation-service.ts                 # Orchestrator + error handling
│   ├── translation-settings.ts                # Storage helpers
│   └── translation-service.test.ts            # Unit tests
└── options/components/
    └── translation-settings.tsx               # Settings UI
```

## Dependencies Between Phases

Phase 01 → Phase 04 (tests verify fixes)  
Phase 02 depends on Phase 01 (model IDs must be correct before wiring UI)  
Phase 03 depends on Phase 01 (type changes in Phase 01 unblock Phase 03 additions)  
Phase 04 covers all phases

## Open Questions

1. Should `gemini-2.5-flash-preview-05-20` be the new Gemini default or use `gemini-1.5-flash` (stable GA) as safe default? Preview models deprecate faster.
2. Is adding OpenRouter/Groq/Mistral in scope for this issue or a follow-up? Brainstorm recommends including them to align with README claims.
3. For Grok, confirm correct stable alias: `grok-2-1212` vs `grok-2-latest`?
