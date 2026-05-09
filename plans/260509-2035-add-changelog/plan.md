---
title: Release Notes via GitHub Releases (no in-repo CHANGELOG)
status: pending
priority: P3
created: 2026-05-09
revised: 2026-05-09
slug: add-changelog
phases: 1
blockedBy: []
blocks: []
---

# Release Notes via GitHub Releases

Single source of truth = GitHub Releases. No `CHANGELOG.md` file, no in-extension parser, no Vite raw import. Options → About gets a one-line "View release notes →" link that opens the GitHub Releases page. Web Store "What's New" is hand-copied from the latest release on each ship.

## Why This Shape

KISS dividend over the prior 3-phase plan ≈ 60% (~3h → ~1h). GitHub already does the hard parts (versioned, dated, immutable, RSS, API, hosted). Adding a markdown file + parser + component just to mirror that data inside the extension would have been pure duplication.

## Scope

- **In:** Backfill historical tags into GitHub Releases (v1.0.1 → v1.0.5), wire a "View release notes" link into Options → About, update `store-listing.md`, drop dead changelog code from `release.yml`.
- **Out:** Anything that creates or imports a `CHANGELOG.md` file, anything that parses markdown at runtime, in-extension auto-popup "What's New" modal on update, parent meta-repo changelog.

## Phases

| # | Phase | Status | Effort |
|---|---|---|---|
| 1 | [Wire GitHub Releases as the changelog](./phase-01-wire-github-releases.md) | pending | ~1h |

## Decisions Locked

| Decision | Choice |
|---|---|
| Source of truth | GitHub Releases page |
| In-extension surface | One link button in Options → About (no parsing, no markdown) |
| Web Store "What's New" sync | Manual copy from latest GitHub Release on each ship |
| Backfill | Create GitHub Releases for v1.0.1 → v1.0.5 from existing tags |
| Format | GitHub's auto-generated release notes (clickable PRs/commits) |
| `release.yml` cleanup | Remove `docs/CHANGELOG.md` write + commit steps (target path doesn't exist) |
