---
title: "Point-in-Time Reconstruction"
---

# CSJ Collector point-in-time reconstruction model

Last reviewed: 2026-04-20

This document defines how to answer the archival question:

- what jobs were visible at a particular point in time?
- what did we know about an individual job at that point?
- what is directly evidenced versus inferred?

## Why this exists

The CSJ Collector now has three important evidence layers:
- run manifests
- current/history job records
- lifecycle/asset/run event logs

To use them well, we need a clear reconstruction rule set.

Without that, people will over-read fields like `last_seen` or mistake a current job record for a point-in-time snapshot.

## Core reconstruction principle

Use the strongest direct evidence first.

Order of trust:
1. immutable run manifest
2. job history snapshot linked to the producing run
3. current job record linked to the producing run, but only as a provenance-linked current projection
4. lifecycle events and interval fields as inference support
5. operational summaries as context, not primary evidence

## What each artifact means

### Run manifest
Stored under:
- `~/.hermes/workspace/csj/csj_run_manifests/{run_id}.json`

Meaning:
- the best direct evidence of which references were seen in a qualifying full unfiltered run
- contains run scope and reference membership
- may also carry listing-card metadata for that run

Use run manifests to answer:
- was reference X visible in this full run?
- what set of jobs did the collector observe in that run?

Do not use a limited or filtered run manifest as if it were a full census.
For CSJ, “filtered” includes non-empty keyword queries as well as `--limit`, `--grade`, and `--department`.
Always inspect `scope` first.

### Current job record
Stored under:
- `csj_jobs/{reference}.json`

Meaning:
- the latest materialized current understanding of a vacancy
- not a historical snapshot by itself
- useful for current state and as the latest known projection
- even when `captured_by_run_id` matches a run, still weaker than an immutable history snapshot because the current record can be rewritten later

Use current job records to answer:
- what do we currently believe about this vacancy?

Do not use them alone to answer:
- what exactly was true at some earlier point in time?

### Job history snapshot
Stored under:
- `csj_history/{reference}/{timestamp}.json`

Meaning:
- immutable historical version saved when a meaningful change or lifecycle transition was recorded
- strongest record-level evidence for how a job changed over time

Use history snapshots to answer:
- what normalized record did we preserve at a particular change point?
- how did content/lifecycle differ from earlier versions?

### Event logs
Stored under:
- `csj_events.jsonl`
- `csj_asset_events.jsonl`
- `csj_run_events.jsonl`

Meaning:
- append-only narrative/audit layer
- good for explaining transitions and anomalies
- weaker than manifests/snapshots for exact reconstruction

Use event logs to answer:
- why did a status transition happen?
- when did a lifecycle or asset event get recorded?

## Reconstruction questions and answers

### Question 1: what jobs were visible at time T?

Preferred method:
1. find the nearest qualifying full unfiltered run manifest at or before time T
2. inspect its `references`
3. treat that as the directly evidenced corpus at that run time

Important nuance:
- this answers what the collector observed in that run
- it does not prove what happened between runs

If there is no full run manifest close enough, then the answer becomes interval-based rather than exact.

### Question 2: what did job X look like at time T?

Preferred method:
1. find the latest history snapshot for reference X at or before time T
2. if none exists, use the earliest current record only if its timestamps make that appropriate
3. use `captured_by_run_id` to connect the snapshot/record back to the producing run
4. use the run manifest for that run to determine whether the job was actually present in the run corpus

### Question 3: was job X active, missing, withdrawn, or closed at time T?

Preferred method:
1. inspect history snapshots around time T
2. inspect lifecycle events around time T
3. use `first_seen`, `last_seen`, `first_missing_at`, and `consecutive_missing_runs` as interval-support fields

Important rule:
- lifecycle fields support interpretation
- the strongest direct evidence of visibility remains the run manifest

## Direct evidence vs inference

This distinction matters a lot.

### Direct evidence
- presence of a reference in a qualifying full run manifest
- immutable job history snapshot
- immutable asset history snapshot
- downloaded attachment/transcript bytes

### Inference
- a job was likely still available between two observed runs
- `last_seen` implies likely continued presence until next contradictory evidence
- missing/withdrawn interpretation based on lifecycle rules and direct-URL verification

When presenting historical findings, label the strength of the claim.

Examples:
- “Directly observed in full run manifest 2026-04-20T...”
- “Inferred from interval fields between full runs”
- “Lifecycle interpretation based on missing-before-expiry logic”

## Run scope rules

Not every run is equal.

### Full unfiltered native run
Strongest corpus evidence.
Use for point-in-time availability questions.

### Limited run (`-n`)
Not a corpus snapshot.
Only partial operational evidence.

### Filtered run (`--grade`, `--department`)
Not a corpus snapshot.
Only evidence for the filtered subset.

### Repair run
Not a source-observation run.
It is a maintenance/provenance event.
Do not use it as evidence of public-site visibility.

## `captured_by_run_id` rule

If a saved record/history snapshot has `captured_by_run_id`, interpret it as:
- this version was produced or rewritten during that collector run

That is provenance, not proof of corpus visibility by itself.
To prove visibility, pair it with the run manifest for the same run if one exists.

## Practical query model

To reconstruct “what was visible and what it looked like” for a full run:

1. open the run manifest
2. enumerate `references`
3. for each reference:
   - find the current/history record linked to that run or the latest record at/before that run
   - use that as the normalized record view
4. use asset version pointers from the record to reconstruct supporting collateral

That is the current best CSJ point-in-time reconstruction workflow.

## Derived reconstruction layer

The collector can now materialize a derived reconstruction file for a qualifying run:
- CLI: `collector.py --reconstruct-run <RUN_ID>`
- output: `~/.hermes/workspace/csj/csj_reconstructions/<RUN_ID>.json`

Important properties:
- derived, not original evidence
- built from:
  - the immutable run manifest
  - history snapshots
  - current records as fallback
- records an `evidence_basis` per reference such as:
  - `history_snapshot_by_run_id`
  - `current_record_by_run_id`
  - `latest_history_snapshot_at_or_before_run`
  - `current_record_fallback`
- includes `direct_evidence` so consumers can distinguish exact run-linked evidence from inference/fallback reconstruction

This is the preferred convenience layer for downstream historical analysis because it does not mutate immutable evidence and can be regenerated whenever the reconstruction rules improve.

Caution:
- reconstruction files can be large for full runs
- they are best treated as warm/derived analysis artifacts rather than core source evidence

## Current limitations

Even with run manifests and `captured_by_run_id`, some limitations remain:
- historical records created before these fields existed may not be fully linked
- there is still no raw HTML evidence layer
- presence between runs remains interval-based, not continuous
- repair-mode mutations may create provenance without corresponding run-manifest visibility evidence

## Maintainer rule

When answering historical questions, prefer:
- run manifest for corpus membership
- history snapshot for record content
- events for explanation
- current record only as latest projection

Do not collapse these into one concept.
That is how historical accuracy drifts.
