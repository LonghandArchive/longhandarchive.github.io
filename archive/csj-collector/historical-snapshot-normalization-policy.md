---
title: "Historical Snapshot Normalization Policy"
permalink: /archive/csj-collector/historical-snapshot-normalization-policy/
---

> **Superseded by** [ADR 0007 — Mutable projections and immutable evidence](../../../docs/decisions/0007-mutable-projections-and-immutable-evidence.md).
> Retained as historical record of the detailed normalisation policy and the metadata categories considered.

---

# CSJ historical snapshot normalization policy

Last reviewed: 2026-04-20

This note defines a conservative policy for whether historical snapshots should ever be rewritten purely to add newer metadata fields.

Short answer:
- by default, no
- historical snapshots should be treated as immutable archival evidence
- metadata-only normalization should prefer companion artifacts, forward-fix conventions, or explicitly versioned migration outputs rather than silent in-place rewrites

## Why this matters

The collector now has newer metadata concepts such as:
- `archive_tier`
- `freeze_status`
- `storage_backend`
- `storage_locator`
- `captured_by_run_id`
- run manifests

Older historical snapshots were created before some of these concepts existed.
That naturally raises the question:
- should we rewrite those old snapshots so the archive looks uniform?

For a normal application database, the answer is often yes.
For an archive, the answer should be much more conservative.

## Core policy

### Current job records and current asset manifests
Safe to rewrite for metadata normalization.

Reason:
- these are current operational/materialized views
- they are meant to reflect the latest understanding
- metadata normalization improves present-day consistency without pretending to preserve an older original state

### Historical job snapshots (`csj_history/`)
Do not rewrite in place for metadata-only normalization by default.

Reason:
- they are immutable historical versions
- the absence of a field is itself historically meaningful
- silently injecting newer metadata into an older snapshot can blur the distinction between:
  - what was preserved at that time
  - what was inferred later

### Historical asset snapshots (`csj_asset_history/`)
Same rule: do not rewrite in place by default.

### Event logs (`*.jsonl`)
Never rewrite in place except for severe repair/recovery cases with explicit archival justification.

## Default stance

Use this rule:

- normalize current projections aggressively
- preserve historical evidence conservatively

That gives you:
- operational consistency now
- archival honesty over time

## What to do instead of rewriting old snapshots

Prefer one of these approaches.

### Option A — leave older snapshots as historically authentic
This is the default.

Interpretation rule:
- if a field is absent in an older snapshot, treat that as "this metadata was not recorded in that snapshot format at the time"
- do not treat it as proof that the value was false

This is often the cleanest archival answer.

### Option B — add companion derived indexes/manifests
If you need uniform querying, create separate derived artifacts such as:
- normalized history index files
- per-run reconstruction indexes
- metadata overlays
- reporting tables/materialized views

These should be clearly marked as:
- derived later
- not original historical snapshots

This is usually the best option when you want both:
- archival integrity
- easier analysis

### Option C — versioned migration outputs
If you ever truly need normalized historical snapshots, do not silently rewrite originals.
Instead:
- create a versioned migration target
- write new derived snapshots into a separate namespace
- preserve the originals

For example:
- original immutable history remains in `csj_history/`
- derived normalized history could live in a separate analysis-oriented area

That preserves provenance.

## When rewriting an old historical snapshot may be acceptable

Only consider in-place rewrite if all of the following are true:

1. the change corrects a demonstrable storage corruption or broken serialization problem
2. the original content cannot be meaningfully interpreted without repair
3. the repair is explicitly documented
4. the repair event itself is recorded as archive maintenance provenance
5. the rewrite does not silently change the substantive historical meaning

Even then, prefer writing a repair note or companion record over mutating the original if possible.

## Metadata categories and rewrite policy

### Safe for current-layer backfill only
- `archive_tier`
- `freeze_status`
- `freeze_candidate_at`
- `frozen_at`
- `storage_backend`
- `storage_locator`

These belong to archive-management semantics, not historical source truth.
They are fine to add to current records/manifests.
They should not be silently injected into older immutable evidence by default.

### Provenance fields requiring extra caution
- `captured_by_run_id`
- any future `captured_from_manifest_id`
- any future raw-capture metadata

These should only be added to older historical snapshots if you actually know them, not because they are useful now.
If unknown, leave them absent or null in current derived overlays, not fabricated in immutable originals.

### Never fabricate
Do not invent later values for older snapshots such as:
- run linkage you cannot prove
- archive completeness status you did not actually compute at the time
- source evidence metadata you did not preserve

## Recommended practical policy for CSJ

### Policy 1
Keep the new `--backfill-archive-metadata` mode limited to:
- `csj_jobs/`
- `csj_asset_manifests/`

### Policy 2
Do not extend that mode to:
- `csj_history/`
- `csj_asset_history/`
- JSONL event logs
without a separate explicit migration design.

### Policy 3
If analysts need uniform historical metadata, add a separate derived reporting/index layer rather than mutating immutable evidence.

### Policy 4
If a future one-time migration of history is ever considered, require:
- a written migration note
- a dry-run report
- clear distinction between original and derived outputs
- explicit archival justification

## Maintainer rule of thumb

Ask this before rewriting an old historical snapshot:

"Am I repairing evidence, or am I making old evidence look more like current expectations?"

If the answer is the second one, do not rewrite it in place.

## Current recommendation

Given the current CSJ state, the best policy is:
- yes: normalize current job records and current asset manifests
- no: do not rewrite immutable history snapshots for metadata-only uniformity
- instead: build derived analysis/reconstruction layers when needed

That preserves archive integrity while still letting the active system evolve.
