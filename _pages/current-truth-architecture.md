---
title: "Current Truth Architecture"
---

# CSJ Collector current-truth architecture

Last reviewed: 2026-04-20

This is the shortest canonical statement of what the CSJ Collector is now.
It is intended to reduce drift between code, docs, and future maintenance work.

## Core architecture

The CSJ Collector is a source-specific archival collector for UK Civil Service Jobs.

It sits in a three-layer model:
1. authoritative collector/archive layer
2. derived database/search/analysis layer
3. derived web/UI/reporting layer

Rule:
- the collector archive is authoritative
- downstream DB/UI layers must remain rebuildable from it

## Important internal distinction

Inside the collector layer, not every artifact has the same evidentiary weight.

### Stronger immutable evidence
- `csj_history/`
- `csj_asset_history/`
- `csj_run_manifests/`
- downloaded attachment/transcript bytes in `csj_attachments/`

### Mutable current/operational projection
- `csj_jobs/`
- `csj_asset_manifests/`
- `csj_latest.json`
- `csj_runs/`
- `csj_state.json`
- JSONL telemetry/event streams

Rule:
- current records are authoritative current projections
- immutable history/manifests are stronger historical evidence
- do not treat mutable current projections as equivalent to immutable archival snapshots

## Run semantics

### Qualifying full unfiltered run
A run only counts as a qualifying full unfiltered corpus run if it is:
- native collection mode
- `--full`
- no `--limit`
- no `--grade`
- no `--department`
- empty `keywords`

Only those runs may:
- write authoritative `csj_run_manifests/` corpus-membership artifacts
- advance iterative state in `csj_state.json`
- trigger the full lifecycle pass that can close/miss/reopen roles against the visible corpus

### Filtered runs
The following all count as filtered runs:
- non-empty keyword queries
- `--limit`
- `--grade`
- `--department`

Filtered runs may still update current records operationally, but must not masquerade as full-corpus evidence.

## Reconstruction semantics

Use evidence in this order:
1. run manifest
2. job history snapshot linked to the producing run
3. current record linked to the producing run, but only as provenance-linked convenience
4. lifecycle/events/interval fields as inference support
5. operational summaries as context only

Rule:
- `current_record_by_run_id` is not direct immutable evidence
- `captured_by_run_id` on a current record is useful provenance, but current records remain rewriteable projections

## Persistence model

### Whole-file JSON artifacts
These should use atomic whole-file writes:
- current records
- manifests
- history snapshots
- summaries

### JSONL event streams
These should use the shared append helper with flush/fsync semantics.

Rule:
- if you harden one write path, harden the matching sibling paths too
- avoid ad-hoc `write_text()` / plain append patterns for archival artifacts when shared helpers exist

## Concurrency/runtime model

### Parallel fetching
Parallel detail fetching should use isolated per-worker collectors/sessions.
Do not share one mutable `requests.Session` / `NativeCollector` across threads.

### Locking
Single-instance lock acquisition should use atomic exclusive create semantics, not plain check-then-write.

## Attachment security model

Attachment downloads should:
- allow only `http` / `https`
- reject localhost/private-IP targets
- enforce a hard size cap before reading the full body
- redact sensitive query params from stored raw source URLs

Rule:
- keep enough source URL shape for debugging
- do not preserve session-bearing query params like `SID`/tokens verbatim in archival metadata

## Mutability rule

Default rule:
- do not rewrite immutable historical snapshots in place for metadata-only uniformity

Allowed:
- rewrite current projections/manifests/summaries as operational layers evolve

Not allowed by default:
- sweeping in-place rewrites of immutable historical evidence just to make older vintages look uniform

## Current code truth

Primary modules:
- `csj/native.py` — source-specific HTTP/auth/search/detail parsing
- `csj/run.py` — orchestration and run semantics
- `csj/records.py` — current record shaping/finalization/persistence
- `csj/lifecycle.py` — lifecycle classification and repair logic
- `csj/assets.py` — attachment/transcript capture plus asset manifests/history/events
- `csj/history.py` — job history snapshots and lifecycle/content event logging
- `csj/state.py` — state and locking
- `csj/io_utils.py` — shared crash-safer I/O helpers
- `csj/telemetry.py` — run-level telemetry

## Maintainer rule of thumb

When changing the system, ask:
- am I changing authoritative evidence,
- a mutable current projection,
- or a derived operational/reporting artifact?

If those get blurred, the architecture drifts.
