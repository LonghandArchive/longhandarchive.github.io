---
title: "Current Status Snapshot"
permalink: /archive/csj-collector/current-status-snapshot/
---

# CSJ Collector current status

Last reviewed: 2026-04-20

This is the concise current-state note for maintainers.
It should stay shorter and more current than the imported handoff/history docs.

## Executive summary

The CSJ Collector is in a credible modular and archival state.
It should now be treated as a maintained archival data product, not a one-off scraper.

Current picture:
- native-only collector path
- modular code layout under `csj/`
- stable public wrapper at `scripts/collector.py`
- history/event/asset archival layers in place
- parser fixtures, golden-output tests, and telemetry code present
- parallel detail fetch now uses isolated per-worker collector/session state rather than sharing one mutable session across threads
- lock-file acquisition now uses atomic exclusive create semantics rather than plain check-then-write
- CLI argument parsing now rejects key invalid combinations/values up front (`--force` without `--full`, bare `--dry-run`, non-positive worker/hour counts)
- attachment download hardening now rejects non-http(s), localhost/private-IP targets, enforces a size cap, and redacts sensitive query params from stored raw source URLs
- local-first archive-tier metadata now introduced for newly written/rewritten job and asset records
- immutable run-manifest support added for qualifying full unfiltered runs
- saved current/history records can now carry `captured_by_run_id`
- derived run reconstructions can now be materialized under `csj_reconstructions/` without mutating immutable evidence
- a dedicated `--backfill-archive-metadata` mode now exists to normalize existing current records and asset manifests safely
- special maintenance modes now degrade run status if they report internal error counts
- history snapshots and JSONL event streams now route through shared crash-safer I/O helpers rather than scattered ad-hoc write paths
- project documentation now centralized under the skill folder

Important caveat:
- existing historical records on disk are not backfilled automatically by this change
- tier metadata appears as records are newly written or rewritten by normal collector/lifecycle flows until a dedicated backfill is run
- `status` is only a broad bucket; use `lifecycle_status` for true live-vs-missing-vs-withdrawn interpretation
- qualifying run manifests are stronger point-in-time evidence than the mutable current `csj_jobs/` projection
- `expired_sid_redirect` is medium-confidence evidence that a stored detail URL no longer yields a vacancy page, not hard withdrawal confirmation
- see `docs/lifecycle-semantics-pressure-test-note.md` for the focused maintainer write-up

## Confirmed architecture state

Key runtime/code surfaces:
- `scripts/collector.py` — stable public entrypoint
- `scripts/collector_impl.py` — compatibility/integration layer
- `csj/cli.py` — CLI and runtime context
- `csj/run.py` — orchestration, statuses, and summary writing
- `csj/native.py` — source-specific acquisition/parsing
- `csj/lifecycle.py` — lifecycle and repair logic
- `csj/assets.py` — asset capture/manifests/history/events
- `csj/history.py` — version snapshots and event logging
- `csj/hashing.py` — diff/hash semantics
- `csj/telemetry.py` — JSONL run telemetry

## Confirmed workspace/archive state

This is already a substantial archive rather than a small scrape output.
The most important operational takeaway is structural:
- `csj_jobs/` and `csj_asset_manifests/` are the mutable current projection
- `csj_history/`, `csj_asset_history/`, and `csj_run_manifests/` are the stronger historical evidence layers
- `csj_runs/`, `csj_latest.json`, and `csj_run_events.jsonl` are operational/derived run artifacts

## Run-artifact status

Current runtime behavior:
- normal native runs write both `csj_latest.json` and `csj_runs/{run_id}.json`
- repair, archive-metadata backfill, and run-reconstruction modes also write per-run summaries
- `csj_run_events.jsonl` remains the append-only run telemetry stream
- per-run summaries are useful operational records, but not primary archival evidence

Open semantics question:
- should `csj_latest.json` reflect every run mode, or only native collection runs?

## Main open review items

Most useful current review items are:
- keep run-artifact/observability docs concise and current
- continue clarifying compatibility-sensitive vs safe-to-change surfaces
- decide whether older duplicate repair-history artifacts should be left as historical truth or cleaned
- continue using one central roadmap instead of accumulating fresh plan docs in the workspace

## Recommended next step

The most useful immediate follow-up would be:
- keep the run-artifact semantics explicit
- decide whether `csj_latest.json` should remain all-modes or become native-collection-only

That keeps the operational story clear while preserving the archive/derived-layer boundary.
