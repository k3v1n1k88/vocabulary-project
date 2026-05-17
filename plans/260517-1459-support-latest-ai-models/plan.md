---
title: "Support Latest AI Models (Issue #5)"
status: pending
priority: P1
created: 2026-05-17
issue: https://github.com/k3v1n1k88/vocabulary-project/issues/5
brainstorm: ../reports/brainstorm-260517-1459-support-latest-ai-models.md
blockedBy: []
blocks: []
---

# Support Latest AI Models (Issue #5)

## Summary

Update hardcoded AI model IDs in `llm-provider-config.ts` to current non-deprecated models for all 3 providers (Gemini, OpenAI, Grok). Add a defensive fallback for stale user preferences stored in `chrome.storage.sync`.

## Why

`gemini-2.0-flash` is deprecated and unavailable to new users (shutdown Jun 1 2026). `gpt-4-turbo` and `grok-beta` are also legacy/removed. Users encounter "model not available" runtime errors when selecting these providers.

## Approach

Static model list update — KISS. No architectural changes. Single file primary target. See brainstorm for why dynamic model discovery was ruled out.

## Phases

| # | Phase | Status | Priority | Effort |
|---|-------|--------|----------|--------|
| 1 | [Update model lists](./phase-01-update-model-lists.md) | pending | P1 | 30m |
| 2 | [Verify and update tests](./phase-02-verify-tests.md) | pending | P2 | 30m |

## Key Files

- `vocabulary-extension/src/shared/llm-provider-config.ts` — primary change
- `vocabulary-extension/src/shared/translation-service.test.ts` — verify no breakage

## Success Criteria

- [ ] No "model not available" errors for any of the 3 providers
- [ ] All existing translation tests pass
- [ ] TypeScript compiles without errors

## Validation Log

### Session 1 — 2026-05-17
**Trigger:** Automated bootstrap (non-interactive)
**Questions asked:** 0 applied, 3 surfaced as open questions

#### Confirmed Decisions

1. **[Architecture]** `resolveModel` defensive fallback: NOT needed — `translation-service.ts` always reads `config.defaultModel` directly and never reads `settings.llmModel` from storage. Options UI handles stale values with `|| currentProvider.defaultModel` pattern. No service-layer changes required.

#### Verification Results
- **Tier:** Light (2 phases)
- **Claims checked:** 8
- **Verified:** 7 | **Failed:** 1 | **Unverified:** 0

#### Failures
1. [Fact Checker] Plan assumed `resolveModel` fallback required for stale stored model IDs — FAILED. `translation-service.ts` ignores `settings.llmModel`; always uses `config.defaultModel`. Fallback step removed from Phase 1.

#### Action Items
- [x] Remove `resolveModel` step from Phase 1 (applied)
- [x] Reduce Phase 1 effort estimate: 1h → 30m (applied)
- [ ] Open question Q3 — verify `grok-4.3` model ID against xAI API docs before implementation
