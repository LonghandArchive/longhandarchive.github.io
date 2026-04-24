---
title: "CSJ Collector Status Handoff"
permalink: /archive/csj-collector/status-handoff/
---

# CSJ Collector Status Handoff

Last updated: 2026-04-19

## Executive summary

The **CSJ Collector** is now in a good modular state and remains fully functional.

It has been reframed from “collector” to **collector** in architecture/documentation terms, while preserving compatibility-sensitive names/paths where needed.

The current working approach is:
- keep architecture/planning/implementation work **inside this skill**
- continue evolving the CSJ Collector in the context of **The Longhand Archive**
- only extract broader shared abstractions when a second real collector exists and shared patterns are proven

---

## What has been accomplished

### 1. Browser path removed
Browser-use/browser fallback support was removed intentionally.
The collector is now native-only.

Result:
- less branching complexity
- simpler orchestration
- cleaner module boundaries

### 2. Core modular refactor completed
The old monolithic implementation has been split into focused modules.

Current code layout:
- `scripts/collector.py` — stable public entrypoint wrapper
- `scripts/collector_impl.py` — thin integration layer / top-level scrape flow
- `csj/run.py` — collector orchestration pipeline, including the top-level scrape flow
- `csj/native.py` — native requests + ALTCHA collection/parsing
- `csj/assets.py` — attachment/transcript capture, manifests, asset history/event handling
- `csj/lifecycle.py` — lifecycle classification, repair mode, withdrawn/missing verification
- `csj/history.py` — job history versions and event logging
- `csj/hashing.py` — comparable-record hashing, diffing, asset/source normalization
- `csj/normalize.py` — normalization helpers
- `csj/records.py` — job-record normalization, finalization, and persistence helpers
- `csj/state.py` — state/locking
- `csj/config.py` — constants and paths
- `csj/cli.py` — CLI entrypoint, parser, and runtime context assembly

### 3. Cleanup pass completed
- leftover duplicate local normalization helpers removed from `scripts/collector_impl.py`
- unused imports removed
- compatibility surface preserved where tests/callers expect it (example: `evaluate_repair_action` still re-exported from collector module surface)

### 4. Terminology pass started and applied safely
Safe wording has been moved toward **Collector** in:
- skill docs
- architecture notes
- docstrings
- CLI help/banner text
- comments where safe

Compatibility-sensitive names were intentionally left unchanged, including:
- `scripts/collector.py`
- `scripts/collector_impl.py`
- skill slug `civil-service-jobs-collector`
- persisted schema field `scraped_at`
- operational examples like `/tmp/csj_collector.log`

### 5. Documentation/architecture notes added inside the skill
The following references now exist and should be maintained as we go:

- `references/csj-collector-architecture.md`
  - current collector architecture
  - source-specific vs collector/core logic
  - working rules for future changes

- `references/csj-archive-envelope.md`
  - CSJ-first explanation of source / content type / archive envelope concepts
  - frames CSJ as:
    - `source = "csj"`
    - `content_type = "job_posting"`

- `references/csj-field-mapping.md`
  - maps current live CSJ schema to a future conceptual archive-envelope model
  - conceptual only, non-breaking

- `references/refresh-lifecycle-edge-cases.md`
  - lifecycle/refresh pitfalls and known behavior

### 6. Workspace plan/status doc updated
`/root/.hermes/workspace/csj/CSJ-MODULAR-REFACTOR-PLAN.md` has been updated so it reflects the current post-refactor state rather than the older future-state plan.

---

## Current operational status

### Tests
Most recent verified state:
- `uvx pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests`
- **71 passed**

### CLI smoke
Verified:
- `python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --help`
- `python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --repair-lifecycle --dry-run`

Help output identifies the tool as:
- `Civil Service Jobs Collector`

### Lifecycle repair / reactivation bug note
A real lifecycle bug was found and fixed after hardening work:
- root cause: `refresh_existing_listing_state()` in `csj/run.py` could silently reactivate already-closed records by resetting skipped/search-present records back to `status=active` / `lifecycle_status=active`
- symptom: repeated `--repair-lifecycle` runs produced duplicate `repair_closed` history snapshots/events for a subset of refs
- quantified impact before fix: **26 of 131** repair-closed refs (~20%) had duplicate repair-closure history/events
- fix applied: closed records are now left closed in that skipped-listing refresh path; genuinely missing-state records still reopen correctly
- verification after fix:
  - targeted lifecycle/run tests passed
  - full suite passed
  - `collector.py --repair-lifecycle --dry-run` returned `closed: 0` on the repaired dataset

Historical duplicate repair-closure entries from earlier runs still exist in the archive, but the current-state reactivation risk in that path appears resolved.

---

## Important compatibility decisions

These were made deliberately and should not be changed casually:

### Keep stable for now
- file paths `scripts/collector.py` and `scripts/collector_impl.py`
- skill slug `civil-service-jobs-collector`
- persisted field names like `scraped_at`
- current CLI contract
- stable wrapper/module surface that tests or operators may import from indirectly

### Safe to continue changing now
- narrative docs/comments
- architecture note wording
- user-facing terminology in help/banners/docstrings
- internal structure where behavior remains unchanged and tests stay green

### Defer until there is an explicit migration plan
- module/file renames from `collector*` to `collector*`
- schema field renames like `scraped_at -> collected_at`
- skill slug rename
- broader shared platform extraction

---

## Recommended next steps

These are the best next actions **inside this skill**.

### Option A — keep documenting while continuing collector work
When changing the CSJ Collector, continue updating:
- `references/csj-collector-architecture.md`
- `references/csj-archive-envelope.md`
- `references/csj-field-mapping.md`

This is the default low-risk path.

### Option B — add non-breaking metadata hints in summaries/reports
Potentially start emitting or documenting conceptual metadata such as:
- `source = csj`
- `content_type = job_posting`

Important: do this first in docs/summaries if needed, not as a breaking schema rewrite.

### Option C — identify likely future shared archive-core concepts
A useful next design task would be to explicitly mark which current modules/functions are most likely to become shared later across The Longhand Archive.

Strong candidates include:
- lifecycle envelope concepts (`first_seen`, `last_seen`, `status`, `last_changed_at`)
- hashing/content-diff logic
- asset manifest/history/event patterns
- orchestration/state patterns

But this should stay conceptual until a second collector exists.

### Option D — continue feature work on CSJ itself
If there is a collector improvement/bugfix/new archival behavior to implement, the architecture is now clean enough to do that work with less risk.

---

## Working doctrine going forward

Use this principle for future work:

> Build The Longhand Archive iteratively by evolving the CSJ Collector first.
> Generalize only from working, tested collector behavior.
> Keep planning and architecture inside this skill until a second collector justifies extraction.

---

## If resuming later

Start by reading:
1. this handoff note
2. `references/csj-collector-architecture.md`
3. `references/csj-archive-envelope.md`
4. `references/csj-field-mapping.md`
5. `references/refresh-lifecycle-edge-cases.md`

Then verify current operational health with:

```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --help
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --repair-lifecycle --dry-run
```

If working on naming/terminology again, remember:
- prefer **Collector** in docs/comments
- preserve compatibility-sensitive `collector` names until there is a deliberate migration plan
