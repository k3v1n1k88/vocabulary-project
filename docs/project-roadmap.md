# Project Roadmap & Changelog

**Vocabulary Builder Chrome Extension**

---

## Phase Overview

### Phase 1: Foundation (v1.0.5 - SHIPPED)

**Status:** Complete & stable

Core extension with all essential features for vocabulary learning:

- Word lookup via Free Dictionary API + right-click context menu
- Multi-language translation (12 languages: VI, ZH, JA, KO, ES, FR, DE, PT, RU, TH, ID, AR)
- Optional AI-powered translation (6 LLM providers: OpenAI, Gemini, Grok, OpenRouter, Groq, Mistral)
- Spaced repetition (SM-2 algorithm) with flashcard study mode
- Gamification (XP, streaks, levels, daily goals)
- Offline-first storage (chrome.storage.local)
- Tooltip + floating menu + highlight + keyboard shortcuts
- PDF side panel lookup (Chrome 114+)
- Audio pronunciation (Google TTS)
- Daily study reminders (chrome.alarms)
- Dark mode support
- Data export/import (JSON)

**Test coverage:** >80% on critical paths (translation, SM-2, store, notification helpers)

**Acceptance criteria:** All met. v1.0.5 released to Chrome Web Store.

---

## Phase 2: Cloud Sync & Collaboration (Planned - Post v1.0.5)

**Status:** Backlog / Design phase

### Settings Sync (DELIVERED — issue #5)

Cross-device settings + LLM API key sync ships via `chrome.storage.sync`
(no backend, no auth). Migrate-on-read upgrades v1.0.5 users transparently.
See [system-architecture.md](./system-architecture.md#chromestoragesync-cross-device--issue-5).

### Vocabulary Cloud Sync (Still Open)

Vocabulary words, flashcards, stats remain device-local. Sizes exceed
`chrome.storage.sync` quotas (~10MB) so cloud backend still needed if pursued.

### Firebase Integration Audit (Open)

Firebase listed in `package.json` (v10.14.1) but no imports found in codebase.
Issue #5 resolved settings sync without Firebase, so the decision is now:
- Remove dead dependency, **or**
- Keep for future vocabulary cloud sync.

**Acceptance criteria (vocabulary cloud sync, if pursued):**
- [ ] Design doc for cloud sync architecture
- [ ] User auth flow (Google/Email sign-in)
- [ ] Data encryption at rest (client-side)
- [ ] Conflict resolution for offline edits
- [ ] Define if Firebase or alternative

### High-Level Goals

If vocabulary cloud sync approved, v2.0.0 will enable:
- Cross-device vocabulary synchronization
- User accounts with sign-in
- Backup & restore
- Shared vocabulary lists (collaborative learning)
- Cloud-backed progress tracking

---

## Phase 3: Advanced Features (Planned - Q3 2026+)

### Multi-Language UI

Localize popup, options, sidepanel to top 5 languages (ZH, VI, ES, FR, DE).

**Effort:** ~2–3 weeks (i18n setup + translation review)

### Voice Input

Add microphone icon to lookup; transcribe speech → dictionary lookup.

**Effort:** ~1 week (Web Speech API integration)

### Extended Language Support

Add 10+ more languages (auto-expansion via community translation).

**Effort:** ~1 week (language pack system)

### Context-Aware Definitions

Pass sentence context to LLM for more accurate, contextual translations.

**Effort:** ~3 days (UX iteration on context capture)

### Mobile App (iOS/Android)

Native vocabulary app with sync to Chrome extension.

**Effort:** ~8–12 weeks (React Native or Flutter)

**Dependencies:** Requires Phase 2 cloud sync.

---

## Phase 4: Monetization & Scale (Planned - Q4 2026+)

### Premium Tier

Optional subscription for advanced features:
- Unlimited LLM translation requests (vs. free tier limits)
- Advanced analytics (learning pace, vocabulary gaps)
- Priority API support
- Ad-free experience

**Effort:** ~3 weeks (pricing logic, backend, billing integration)

### Analytics Dashboard

User-facing learning analytics:
- Words learned per week
- Average retention rate
- Predicted proficiency level
- Recommendations (focus on weaker areas)

**Effort:** ~2 weeks (metrics collection, visualization)

### Community Features

- Public vocabulary lists (shared learning resources)
- User profiles & leaderboards
- Discussion forum (Q&A on word meanings)

**Effort:** ~4–6 weeks (social features, moderation)

---

## Near-Term Backlog (v1.0.5 Post-Release)

### High Priority (Next sprint)

| Task | Scope | Owner | Status |
|------|-------|-------|--------|
| **Firebase dependency audit** | Clarify if dead code or planned; document decision | TBD | Blocked |
| **release-manifest.json deduplication** | Consolidate root & submodule versions; clarify release workflow | TBD | Todo |
| **design-tokens.css documentation** | Map token contract, verify Tailwind integration | TBD | Todo |
| **vitest-chrome mocking surface** | Document chrome API mock coverage; identify gaps | TBD | Todo |
| **A11y audit & fixes** | WCAG 2.1 AA compliance verification + fixes | TBD | Todo |
| **Performance benchmark** | Establish baseline metrics (tooltip latency, highlight restore, etc.) | TBD | Todo |

### Medium Priority (2–4 weeks out)

| Task | Scope | Owner | Status |
|------|-------|-------|--------|
| **Keyboard shortcut UX polish** | Improve key-combo recorder UI & validation feedback | TBD | Todo |
| **Translation response caching** | Cache LLM responses per word+language to reduce API calls | TBD | Todo |
| **Highlight color persistence UI** | Improve color picker UX, remember recent colors | TBD | Todo |
| **Error toast notifications** | Standardize toast UI & messaging (currently inconsistent) | TBD | Todo |
| **PDF detection reliability** | Strengthen `isPdfUrl` heuristic; handle more edge cases | TBD | Todo |

### Low Priority (Nice-to-have)

| Task | Scope | Owner | Status |
|------|-------|-------|--------|
| **Icon refresh** | Modernize extension icon design | TBD | Todo |
| **Welcome onboarding** | Tutorial for first-time users | TBD | Todo |
| **Newsletter integration** | Weekly vocabulary email digest (opt-in) | TBD | Todo |

---

## Success Metrics

### Phase 1 (Shipped)

- [x] 0 critical bugs in v1.0.5
- [x] >80% test coverage on critical paths
- [x] Tooltip latency <200ms (verified via performance tests)
- [x] Highlight restoration <500ms (verified)
- [x] Cross-tab settings sync <100ms (verified)
- [x] Chrome Web Store approval (live)

### Phase 2 (Cloud Sync)

- [ ] 500+ users with accounts
- [ ] 99.9% uptime on auth + sync endpoints
- [ ] Cross-device sync latency <2 seconds
- [ ] Zero unencrypted data in cloud storage

### Phase 3 (Advanced Features)

- [ ] 1,000+ active users
- [ ] >50% adoption of voice input
- [ ] Average 100+ words learned per user per month

### Phase 4 (Monetization)

- [ ] 10%+ conversion to premium tier
- [ ] $5k+ monthly recurring revenue
- [ ] Net Promoter Score (NPS) >50

---

## Dependencies & Blockers

### Blocker: Firebase Dependency Clarification

**Issue:** Firebase listed in `package.json` but no imports found.

**Impact:** Cannot proceed with cloud sync planning without clarification.

**Resolution needed:** Confirm with product team whether Firebase is planned feature or dead code.

### Blocker: release-manifest.json Duplication

**Issue:** Files exist at `vocabulary-extension/` and project root; unclear which drives CI/CD.

**Impact:** Release process may be ambiguous; version bumping could fail.

**Resolution needed:** Audit and consolidate; document single source of truth.

### Risk: LLM Provider API Costs

**Issue:** If premium unlimited LLM translation offered, costs could scale unpredictably.

**Mitigation:** Rate limit per user, use model auto-routing (cheaper models first), or require user API keys.

### Risk: Data Privacy Compliance

**Issue:** Cloud sync requires GDPR/privacy compliance (if EU users).

**Mitigation:** Privacy policy, data retention policy, opt-in analytics consent.

---

## Deprecation & Breaking Changes

None in v1.0.5. Future phases will maintain backward compatibility:

- Storage schema migrations handled via version checks
- API contract versioning (message enums support new actions without breaking old)
- UI theme changes opt-in (dark mode toggle respected)

---

## Timeline Estimate

| Phase | Scope | Team Size | Duration | Est. Start |
|-------|-------|-----------|----------|-----------|
| Phase 1 | Core features | 1–2 | 3 months | Oct 2025 |
| Phase 2 | Cloud sync | 2–3 | 6–8 weeks | Jun 2026 |
| Phase 3 | Advanced features | 2 | 4–6 weeks | Aug 2026 |
| Phase 4 | Monetization | 3 | 8–10 weeks | Oct 2026 |

---

## Version History

### v1.0.5 (2026-05-08)

**Stable release**

- Word lookup via Free Dictionary API
- 12-language translation support (free + 6 LLM providers)
- SM-2 spaced repetition with flashcards
- Gamification (XP, streaks, levels)
- Tooltip + floating menu + highlight + shortcuts
- PDF side panel (Chrome 114+)
- Dark mode
- Data export/import
- Audio pronunciation
- Daily reminders
- 95 source files, ~11.7k LOC
- Test coverage >80%

### v1.0.0 - v1.0.4

(Earlier versions; see GitHub releases for detailed changelog)

---

## Stakeholder Input

### Product Team

- Confirm Firebase plan
- Prioritize Phase 2 vs Phase 3 features
- Set user growth targets (Phase 4 monetization trigger)

### Engineering Team

- Estimate capacity for Phase 2 cloud sync architecture
- Review tech debt (design-tokens.css, vitest-chrome mocking)
- Plan A11y audit scope

### Users

- Collect feature requests (survey, Discord, GitHub issues)
- Track churn / NPS
- Identify pain points in current workflows

---

## Review Cadence

- **Monthly:** Update progress, re-rank backlog
- **Quarterly:** Assess phase readiness, reset timelines
- **Semi-annually:** Major roadmap refresh

Next review: June 8, 2026.

---

## Open Questions

1. **Firebase clarification** — Is cloud sync in Phase 2, or remove dependency?
2. **Monetization strategy** — Premium tier, ads, or donations only?
3. **Mobile app priority** — Is native iOS/Android in scope?
4. **Community features** — Will project support user-generated content (lists, leaderboards)?
5. **Localization priority** — Which 5 languages to localize first?
