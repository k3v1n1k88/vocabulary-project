---
phase: 1
title: "Wire GitHub Releases as the changelog"
status: pending
priority: P3
effort: "~1h"
dependencies: []
---

# Phase 1: Wire GitHub Releases as the Changelog

## Overview

Replace the never-existed in-extension changelog with a one-click jump to the GitHub Releases page. Backfill historical tags so the page has real content from day one. Clean dead changelog code out of `release.yml`.

## Key Insights

- Existing tags v1.0.1 → v1.0.5 have **no associated GitHub Release** — they're plain git tags. Without a Release object, the GitHub Releases page is empty, which makes a "View release notes →" button pointless on day one. Backfill is the prerequisite for everything else.
- `release.yml` already creates GitHub Releases with ZIP attachments on dispatch. The pipeline forward is solid; only the legacy `docs/CHANGELOG.md` write/commit steps are dead code.
- `store-listing.md` line 88 promises "Changelog: in-extension Options → About". The link button satisfies that promise without parsing anything.
- Hardcoded `v1.0.0` on `about-section.tsx` line 71 is stale — current version is 1.1.0. Worth fixing in the same touch.

## Requirements

**Functional**
- GitHub Releases exist for tags v1.0.1, v1.0.2, v1.0.3, v1.0.4, v1.0.5 with reasonable user-facing notes
- Options → About has a button labeled "View release notes" that opens `https://github.com/k3v1n1k88/vocabulary-extension/releases` in a new tab
- About panel shows live version from `package.json`, not the hardcoded `v1.0.0`
- `store-listing.md` text matches reality (link goes to GitHub, Pre-Submit Checklist mentions copying release-note bullets to Web Store "What's New")
- `release.yml` no longer references `docs/CHANGELOG.md`

**Non-functional**
- Zero new runtime dependencies
- Zero markdown parsing at runtime
- Bundle size delta ≈ 0 (one anchor + one import of `package.json` for version)

## Architecture

```
GitHub Releases (per tag)
    │
    ├──→ Repo visitors                       (read directly on github.com)
    ├──→ Web Store "What's New"              (manual copy on release)
    └──→ Options → About → "View release notes →"  (anchor tag, target=_blank)
```

No file imports. No parser. No build-time data flow. Just an anchor and a static URL.

## Related Code Files

- **Modify:**
  - `vocabulary-extension/src/options/components/about-section.tsx` — add "View release notes" button, replace hardcoded version with `package.json` version
  - `vocabulary-extension/docs/store-listing.md` — line 88 wording, Pre-Submit Checklist additions
  - `vocabulary-extension/.github/workflows/release.yml` — drop the "Generate changelog" / "Update CHANGELOG.md" / "Commit changelog" steps
- **Create:** none
- **Delete:** none in repo. (GitHub Releases will be created via UI / `gh` CLI; no git artifacts.)

## Implementation Steps

### Step 1 — Backfill GitHub Releases (~25m)

For each existing tag v1.0.1 → v1.0.5:

1. Get tag commit + date:
   ```bash
   git -C vocabulary-extension show -s --format="%cI %s" v1.0.5
   ```
2. Use `gh` CLI from the submodule directory to create the Release with auto-generated notes:
   ```bash
   gh release create v1.0.5 \
     --repo k3v1n1k88/vocabulary-extension \
     --title "v1.0.5" \
     --generate-notes \
     --verify-tag
   ```
   `--generate-notes` produces bullets from PRs merged between v1.0.4 and v1.0.5. If the output is too commit-message-shaped, hand-edit on GitHub UI after creation.
3. Repeat for v1.0.4, v1.0.3, v1.0.2, v1.0.1 in that order (newest first; `--generate-notes` walks back to the previous tag).
4. Spot-check on github.com: each release page should have ≥1 user-readable bullet, correct date, no `chore:` / `docs:` prefixes leaking through. Edit on the web UI where needed.

**Skip** v1.0.6 and v1.1.0 — those have no tag yet. The next normal release run via `release.yml` will create them.

### Step 2 — Add the link to About (~10m)

In `vocabulary-extension/src/options/components/about-section.tsx`:

1. Import `package.json` version at the top:
   ```tsx
   import pkg from '../../../package.json'
   ```
2. Replace the `v1.0.0` literal on line 71 with `v{pkg.version}`.
3. In the buttons row (the same flex container that holds "Rate Extension" and "Report Issue"), add a third button **before** "Report Issue":
   ```tsx
   <a
     href="https://github.com/k3v1n1k88/vocabulary-extension/releases"
     target="_blank"
     rel="noopener noreferrer"
     className="inline-flex items-center gap-2 px-4 py-2 bg-gray-100 text-gray-800 rounded-lg text-sm font-medium hover:bg-gray-200 transition-colors shadow-sm"
   >
     <svg className="w-5 h-5" viewBox="0 0 24 24" fill="currentColor">
       {/* tag icon — pick from existing icon style or use a simple SVG */}
       <path d="..." />
     </svg>
     View release notes
   </a>
   ```
   Keep styling consistent with the existing button row.
4. Build + lint:
   ```bash
   cd vocabulary-extension && npm run build && npm run lint
   ```
5. Load unpacked from `dist/`, open Options, verify version label shows `v1.1.0` and the new button opens the GitHub Releases page in a new tab.

### Step 3 — Update `store-listing.md` (~10m)

In `vocabulary-extension/docs/store-listing.md`:

1. Line 88, change:
   ```
   - Changelog: in-extension Options → About
   ```
   to:
   ```
   - Changelog: Options → About → "View release notes" (opens GitHub Releases)
   ```
2. In section 11 "Pre-Submit Checklist", add:
   ```
   - [ ] Copy top 3 bullets from latest GitHub Release into the Web Store "What's New in this version" field (≤500 chars)
   - [ ] Confirm `package.json` version, `manifest.json` version, and the latest GitHub Release tag all match
   ```
3. Add new section 12 (short — this is a one-line reference, not the full template the prior plan revision had):
   ```markdown
   ## 12. What's New (Web Store dashboard field)

   Source: latest entry on https://github.com/k3v1n1k88/vocabulary-extension/releases
   Limit: ≤500 chars. Trim PR numbers, keep user-language bullets.
   ```

### Step 4 — Clean dead code from `release.yml` (~15m)

In `vocabulary-extension/.github/workflows/release.yml`:

1. **Delete** the "Generate changelog" step (lines ~55–99) — its output is only used by other steps that are being removed, plus the GitHub Release body which can use `--generate-notes` instead.
2. **Delete** the "Update CHANGELOG.md" step (lines ~102–121) — writes to a path that doesn't exist.
3. **Delete** the "Commit changelog" step (lines ~129–135) — has nothing to commit after step 2 is gone.
4. **Update** the "Create GitHub Release" step (lines ~145–161): replace the `body:` block that referenced `${{ steps.changelog.outputs.changelog }}` with `generate_release_notes: true` (an `action-gh-release` flag that produces the same auto-bullets GitHub gives you in the UI).
5. **Decide** about the "Update Chrome Web Store listing with changelog" step (lines ~185–197): it depends on `/tmp/changelog.txt` from the deleted step. Either:
   - **Drop it** — Web Store "What's New" becomes pure manual copy-paste (matches Step 3 above) ✅ recommended
   - **Keep it** — re-derive from `gh release view --json body $TAG` and feed into the curl payload. More code; only worth it if you genuinely never want to touch the dashboard.
   Pick "drop it" unless the cadence is so high that manual copy is annoying. (At weekly, it isn't.)
6. Run a no-op dispatch with `publish_to_store=false` once Phase 1 lands to confirm CI still passes.

## Todo List

- [ ] Backfill GitHub Releases for v1.0.1 → v1.0.5 (5 releases)
- [ ] Spot-edit any release where `--generate-notes` output is unreadable
- [ ] Add `View release notes` button to `about-section.tsx`
- [ ] Replace hardcoded `v1.0.0` with `v{pkg.version}` in `about-section.tsx`
- [ ] `npm run build && npm run lint` clean
- [ ] Manual smoke test: load unpacked, click button, lands on GitHub Releases
- [ ] Update `store-listing.md` line 88 + Pre-Submit Checklist + new section 12
- [ ] Delete dead changelog steps from `release.yml`
- [ ] Wire `generate_release_notes: true` into the `action-gh-release` step
- [ ] Drop the "Update Chrome Web Store listing with changelog" step (or refactor — recommended drop)
- [ ] Test `release.yml` with `publish_to_store=false` dispatch

## Success Criteria

- [ ] GitHub Releases page shows v1.0.1 → v1.0.5 with real bullets, correct dates, no commit-prefix leakage
- [ ] Options → About shows current version and a working "View release notes" button
- [ ] `store-listing.md` text matches reality (no broken promises about in-extension changelog)
- [ ] `release.yml` has no references to `docs/CHANGELOG.md`, runs green on dispatch with `publish_to_store=false`
- [ ] No `CHANGELOG.md` file exists in the submodule (we're explicitly NOT creating one)
- [ ] Build size delta within ±1KB of pre-change baseline

## Risk Assessment

| Risk | Mitigation |
|---|---|
| `gh release create --generate-notes` produces ugly bullets for old tags | Hand-edit on GitHub UI after creation; you'll only do this 5 times ever |
| Users see "GitHub" link and bounce | Web Store "What's New" remains the primary user-facing surface; GitHub link is for power users + honoring the store-listing promise |
| `release.yml` breaks after refactor | Test with `publish_to_store=false` before next real ship; can rollback the YAML if needed |
| Future need for offline / structured changelog reappears | Easy to add later; `CHANGELOG.md` + parser is a 2-3 hour add when actually justified by user feedback |
| `package.json` import in TS triggers `resolveJsonModule` issue | Most modern `tsconfig.json` already has it; if not, single-line config add |

## Security Considerations

None — public-facing release notes, no secrets. The Web Store API publish path stays gated behind the existing CI secrets.

## Next Steps

After this phase the system is steady-state:
- Each ship: write GitHub Release notes (manual or `--generate-notes`), copy top 3 bullets into Web Store dashboard, done.
- No file maintenance, no parser maintenance.
- Revisit only if: users complain about the GitHub redirect, OR release cadence becomes daily-enough that the manual Web Store copy is annoying. Neither is currently true.
