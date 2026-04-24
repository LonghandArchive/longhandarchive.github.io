---
title: "Local-First Tiering Architecture"
---

# CSJ Collector local-first archival tiering architecture

Last reviewed: 2026-04-20

This document defines how archival tiering should work now.

Important constraint:
- all data remains on local disk for the time being
- no R2/blob offload is implemented yet
- the local layout should be designed so that a later offload of a whole tier is straightforward

## Goal

Keep the collector operationally simple on local disk while introducing clear archival tiers.

That means:
- active/current data stays in obvious local paths
- immutable historical evidence gets separated structurally
- frozen records can move between local tiers without changing their logical identity
- future offload to object storage should be possible by moving a tier boundary, not redesigning the archive model

## Design principle

Separate by artifact role first, tier second.

Artifact role answers:
- what is this file for?
- is it mutable or immutable?
- is it current state, historical evidence, or telemetry?

Tier answers:
- how quickly does the collector need it?
- how often will it change?
- how expensive is it to keep on primary local disk?

## Logical tiers

### Tier 0 — operational hot
Fast local storage. Frequently read or updated by the collector.

Artifacts:
- `csj_jobs/`
- `csj_state.json`
- `csj_latest.json`
- `csj_runs/`
- `csj_run_events.jsonl`
- `csj_asset_manifests/`
- recent working subsets of history/event data if needed for active operations

Properties:
- mutable
- read/write during normal runs
- should not require indirection through archive manifests just to operate

### Tier 1 — local warm archive
Still on disk, but logically colder and more append-oriented.

Artifacts:
- `csj_history/`
- `csj_asset_history/`
- older run summaries
- partitioned older event logs, if/when log partitioning is added
- per-run corpus manifests

Properties:
- mostly immutable
- useful for analysis and reconstruction
- not required for every scrape loop step

### Tier 2 — local cold evidence
Still on disk for now, but treated as the first offload-ready tier.

Artifacts:
- pooled attachments
- future raw detail HTML captures
- future raw listing HTML captures
- immutable run-manifest payloads if they grow materially

Properties:
- immutable or append-only
- large byte footprint
- low operational mutation frequency
- best future candidate for R2/blob storage

## Local-disk structure recommendation

Current paths already map well to this model, but the tiering should become explicit.

Recommended structure under `~/.hermes/workspace/csj/`:

```text
csj/
├── hot/
│   ├── jobs/                  # current materialized job records
│   ├── state/
│   │   ├── csj_state.json
│   │   ├── csj_latest.json
│   │   └── locks/
│   ├── runs/
│   │   ├── summaries/
│   │   └── run_events.jsonl
│   └── assets/
│       └── manifests/
├── warm/
│   ├── history/
│   │   ├── jobs/
│   │   └── assets/
│   ├── events/
│   │   ├── jobs/
│   │   └── assets/
│   └── manifests/
│       └── runs/
└── cold/
    ├── attachments/
    │   └── pool/
    ├── raw/
    │   ├── detail_html/
    │   └── listing_html/
    └── exported/
```

This is a logical target shape, not an immediate migration requirement.

## Compatibility rule

Do not break current code paths now just to rename directories.

For the next phase, the safer approach is:
- keep current on-disk paths working
- introduce tier metadata/manifests first
- only migrate directory layout after the tier model is stable and tested

So the immediate implementation should be metadata-first, not path-churn-first.

## What should be tier-aware first

Before moving files around, make the archive understand tier membership explicitly.

Add tier awareness to:
- records
- manifests
- maintenance tools
- future reconstruction logic

### Job-level metadata
Current job records should eventually expose archival-placement metadata such as:
- `archive_tier`
- `freeze_status`
- `freeze_candidate_at`
- `frozen_at`
- `storage_class`

These do not need to imply remote storage yet.
They just express archival state.

### Asset-level metadata
Asset manifests are the best place to start.
Each asset record should be able to carry:
- `archive_tier`
- `freeze_status`
- `storage_backend` (`local` for now)
- `storage_locator` (still local path for now)
- `eligible_for_offload_at`

This gives you migration-ready semantics without moving anything off disk.

### Run-manifest metadata
Run manifests should also be tierable.
They are likely warm or cold evidence, not hot operational state.

## Freeze model

Do not freeze the current projection first.
Freeze immutable evidence first.

### Hot forever-ish
These remain hot or operationally current:
- active jobs
- missing-unconfirmed jobs
- current asset manifests
- current state and latest summary

### Freeze candidates
These become eligible for colder treatment once stable:
- closed jobs past a stability window
- withdrawn-confirmed jobs past a stability window
- older immutable history snapshots
- immutable asset history
- pooled attachments tied only to frozen jobs

### Practical rule
A record becomes tier-eligible before it becomes physically relocated.

That means the workflow should be:
1. classify as eligible
2. mark in metadata
3. verify nothing active still depends on hot-local assumptions
4. move later, if desired

## Why metadata-first is the right local-first move

Because you said everything stays on disk for now.

That means the immediate win is not physical relocation.
The immediate win is:
- clearer archive semantics
- explicit freeze eligibility
- less future migration ambiguity
- ability to build maintenance/reporting around tier status now

This also reduces future offload risk because the archive already knows what each artifact is supposed to be.

## Recommended local-first conventions

### Convention 1: storage backend is always explicit
Even while everything is local, represent storage backend explicitly as:
- `storage_backend: "local"`

That avoids assuming local-path semantics forever.

### Convention 2: stable logical identity must not depend on current path
Paths are implementation detail.
Logical identity should stay rooted in:
- job reference + history timestamp/version
- asset_id + asset_version_id
- run_id
- run manifest id

### Convention 3: local paths should be derivable, not identity-bearing
Future offload gets much easier if paths are treated as resolvable locations, not core identity.

### Convention 4: tier transitions should be idempotent
Moving something from hot/warm/cold on local disk later should not mutate its logical identity or break references.

## Immediate design recommendation

Implement in this order:

1. Tier metadata model
- define freeze and tier fields
- define backend/locator conventions

2. Run manifests
- preserve immutable per-run corpus manifests
- classify them as warm/cold evidence from day one

3. Asset manifest tier metadata
- especially for pooled attachments

4. History snapshot tier metadata
- job history and asset history

5. Optional local directory migration
- only after semantics are stable

## What not to do yet

- do not implement remote-storage logic now
- do not rename all workspace paths immediately
- do not force the collector to resolve everything indirectly through a storage abstraction before the metadata model exists
- do not freeze current `csj_jobs/` as if it were immutable evidence

## Practical conclusion

For now, the archive should become tier-structured in meaning, not yet in infrastructure.

That means:
- all bytes remain local
- tier identity becomes explicit
- frozen evidence becomes distinguishable from hot operational state
- later offload becomes a storage decision, not an archive redesign
