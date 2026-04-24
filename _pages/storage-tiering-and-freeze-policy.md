---
title: "Storage Tiering and Freeze Policy"
---

# CSJ Collector storage tiering and record-freeze note

Last reviewed: 2026-04-20

This note explains the practical implications of adding stronger archival layers and suggests when records can be treated as effectively frozen for colder storage tiers.

## Current live storage baseline

Observed current workspace usage:
- total archive workspace: 1.6G
- `csj_jobs`: 116M
- `csj_history`: 160M
- `csj_asset_history`: 151M
- `csj_attachments`: 1.1G
- `csj_asset_manifests`: 43M
- `csj_events.jsonl`: 53.6M
- `csj_asset_events.jsonl`: 34.4M

Observed current counts:
- current job records: 1696
- job history versions: 2897
- asset history versions: 36967
- current statuses:
  - active: 1095
  - closed: 601
  - missing_unconfirmed: 21

Important current observation:
- the archive window is still young
- earliest `first_seen` in current job files is 2026-04-12
- current closed records are only 0–7 days past closing date

So we can reason about freeze policy now, but the live dataset is not yet old enough to validate long freeze thresholds empirically.

## What the extra archival layer means

Adding stronger archival layers changes the project from:
- current-state dataset with history

to:
- evidence-backed archive

The implications are:

1. storage becomes driven by preserved observations, not just current records
2. some apparently redundant data becomes intentionally valuable evidence
3. provenance and retention policy matter more
4. source evidence, derived records, and operational artifacts should be treated differently

## Not all stored data should live on the same tier

The right way to think about this is by data class, not by one big archive bucket.

### Class A — hot operational data
Needs fast local access because the collector mutates or reads it often.

Examples:
- `csj_jobs/`
- `csj_state.json`
- `csj_latest.json`
- `csj_asset_manifests/`
- recent `csj_history/`
- recent `csj_asset_history/`
- recent events and recent run summaries

### Class B — warm historical data
Useful for frequent inspection and analysis, but not needed for every run.

Examples:
- older `csj_history/`
- older `csj_asset_history/`
- older run summaries
- maybe older event segments if you ever partition logs

### Class C — cold source evidence
Important to preserve, rarely needed for active collector logic.

Examples:
- pooled attachments in `csj_attachments/_pool/`
- preserved raw HTML, if you add it
- immutable per-run manifests, if you add them

This class is the best candidate for object storage like R2.

## When does a record become effectively frozen?

This should be defined conservatively.

A record is not truly frozen just because it is closed.
It is frozen when both:
- the upstream source is very unlikely to change again
- your own collector logic is unlikely to need to revise its interpretation

### Good practical freeze states

#### 1. Strongest freeze candidate: `closed` and well past close date
Suggested condition:
- `status == "closed"`
- `lifecycle_status == "closed"`
- close date passed by a safety window
- no later reopen/missing transitions observed
- asset manifest is complete enough for your standards

Suggested initial safety window:
- 30 days after `closes_iso` for a first cold-tier candidate
- 90 days after `closes_iso` for a stronger freeze boundary

Why not immediately at close?
Because late fixes, reopenings, and archive-repair logic can still happen shortly after closure.

#### 2. Strong freeze candidate: `withdrawn_confirmed` after stability period
Suggested condition:
- `lifecycle_status == "withdrawn_confirmed"`
- unchanged for a stability window
- no later reappearance in search results

Suggested initial window:
- 30 days after confirmation for cold candidate
- 90 days for stronger freeze

#### 3. Not frozen: `active`
Active records should stay hot.
They are still subject to:
- content revisions
- date changes
- salary changes
- attachment changes
- disappearance/reappearance

#### 4. Not frozen: `missing_unconfirmed`
These are explicitly unresolved.
Keep them hot.
They are still in interpretive limbo.

## The more important concept: freeze the evidence, not necessarily the current projection

For archival design, the best thing to freeze is usually:
- immutable history snapshots
- immutable run manifests
- immutable asset versions
- immutable raw captures

The current job record in `csj_jobs/` is more like a projection or latest materialized view.
That file may always remain mutable, because it is your best current understanding.

So the most robust policy is:
- keep the current projection hot and mutable
- freeze older immutable evidence layers

That is cleaner than trying to freeze the entire current-state layer wholesale.

## Recommended storage-tier strategy

### Tier 1 — local hot storage
Keep locally:
- all `active` records
- all `missing_unconfirmed` records
- all records changed in last 30 days
- `csj_jobs/`
- `csj_state.json`
- `csj_latest.json`
- current asset manifests
- recent run summaries and recent event logs

### Tier 2 — local warm or cheap attached disk
Move or compact here:
- history snapshots for records closed 30+ days ago
- asset history for records closed 30+ days ago
- older run summaries
- partitioned older event log segments if you later split logs

### Tier 3 — object storage / R2 / blob storage
Best candidates:
- pooled attachments for records closed 30–90+ days ago
- immutable job history snapshots older than 90 days
- immutable asset history older than 90 days
- future raw HTML captures
- run manifests

R2 is especially attractive for immutable evidence objects because:
- read frequency should be low
- object semantics are a good fit
- cost is usually better than keeping everything on local disk

## What should probably stay local even if colder tiers exist

Even with R2, I would still keep local copies or local materialized views of:
- `csj_jobs/` current records
- `csj_asset_manifests/`
- enough recent history to support ordinary debugging and research
- state and latest summary files

Why:
these support active collector operation and fast day-to-day analysis.

## Current data suggests where the cost really is

Right now the biggest storage bucket is already:
- `csj_attachments`: 1.1G

That means the first high-value tiering move is not current JSON.
It is auxiliary source evidence, especially immutable attachments.

So if you want the earliest storage win, the best candidates are:
1. pooled attachments
2. future raw HTML capture if added
3. older immutable history snapshots

The least urgent thing to offload is:
- `csj_jobs/`

## Rough cost of the next archival layer

Very rough annual scenario estimates:
- run manifests at ~100 KiB each for 7 runs/day: ~0.24 GiB/year
- detail HTML at 50 KiB for 20 changed records/day: ~0.35 GiB/year
- detail HTML at 100 KiB for 100 changed records/day: ~3.48 GiB/year
- detail HTML at 150 KiB for 200 changed records/day: ~10.44 GiB/year

Interpretation:
- per-run manifests are cheap
- raw detail HTML is still manageable unless you preserve lots of it very frequently
- attachments remain the dominant storage pressure in most realistic scenarios

## My recommended freeze policy

Start simple.

### Operational rule set

Keep hot:
- all `active`
- all `missing_unconfirmed`
- all records changed within last 30 days
- all records with incomplete auxiliary archive status that may still be repaired

Eligible for cold tier candidate:
- `closed` for 30+ days
- `withdrawn_confirmed` for 30+ days
- no observed changes within that window
- asset completeness accepted as final-enough for archive policy

Eligible for deeper archive tier:
- `closed` or `withdrawn_confirmed` for 90+ days
- no later reopen/repair activity
- all immutable history/assets already written and verified

## Important caution

Do not offload only the current job JSON and assume you have preserved history.
If you tier storage, tier the immutable evidence layers first.

The archive-grade principle is:
- latest view is convenient
- immutable evidence is what really matters

## Best next move

If you continue down this path, the best next design task is:
- define a formal data-tiering policy by artifact type and lifecycle state
- then implement object-storage offload for pooled attachments and later immutable history/manifests

That gives you the most storage relief with the least damage to active collector behavior.
