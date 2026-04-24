---
title: "Authoritative Collector and Derived Layers"
---

# Authoritative collector and derived database/UI layering

Last reviewed: 2026-04-20

This document defines the architectural layering principle for the CSJ Collector and for future collectors in The Longhand Archive.

## Core rule

The collector archive is authoritative.
Everything downstream is derived.

That means the system should be understood as three layers.

## Layer 1 — authoritative archival collector layer

This is the collector itself and the filesystem archive it produces.

Purpose:
- acquire from the source
- preserve observed records
- preserve historical change
- preserve supporting assets
- preserve provenance
- preserve enough evidence for reconstruction and later validation

Properties:
- authoritative source of truth
- archival, conservative, evidence-preserving
- allowed to retain raw contact details, email addresses, and other source-visible personal information if that is part of the archival policy
- not primarily optimized for query ergonomics
- not the place to apply downstream redaction for presentation use cases

Examples in CSJ:
- `csj_jobs/`
- `csj_history/`
- `csj_asset_manifests/`
- `csj_asset_history/`
- `csj_attachments/`
- `csj_events.jsonl`
- `csj_asset_events.jsonl`
- `csj_run_manifests/`

This layer is the long-term authority.

## Layer 2 — derived database / analytical layer

This is not the collector.
It is the downstream ingestion layer you plan to build.

Purpose:
- ingest from the authoritative collector archive
- normalize into query-friendly relational/search structures
- redact or minimize sensitive fields where required
- support indexing, reporting, joins, filtering, and aggregate analysis
- provide stable models for applications and analytics

Properties:
- derived
- rebuildable from the collector archive
- allowed to reshape and denormalize data
- allowed to redact fields for operational or publication needs
- must never become the only surviving source of truth

Important rule:
- if the database can be rebuilt from the collector archive, the architecture is healthy
- if the collector archive starts depending on the database for truth, the architecture is drifting in the wrong direction

## Layer 3 — web/UI/reporting layer

Purpose:
- present insights
- expose search/reporting
- support user-facing exploration
- drive application workflows

Properties:
- presentation-oriented
- derived from the database layer and, in some cases, other derived indexes
- not authoritative
- safe to iterate quickly
- safe to redesign without threatening archival integrity

## Why this separation matters

### 1. The collector should optimize for evidence, not convenience
The collector’s primary job is to preserve a trustworthy archive.
That means:
- capture fidelity
- provenance
- completeness
- historical traceability
- conservative mutation policy

It does not need to be the fastest or easiest thing to query directly.

### 2. Redaction belongs downstream
If the collector archive is authoritative, it may legitimately retain:
- email addresses
- names
- contact details
- other source-visible personal information

The database/UI layer can then derive:
- redacted views
- searchable safe subsets
- presentation-safe schemas

This avoids destroying archival truth for the sake of application convenience.

### 3. The database should be rebuildable
A good downstream model can always be regenerated from the collector archive.
That lets you change:
- DB schema
- query models
- indexes
- search strategies
- UI features
without altering the archival source of truth.

## Architectural doctrine for future iterations

Use these rules for future code and docs.

1. The collector archive is authoritative.
2. The database layer is derived and rebuildable.
3. The web/UI layer is derived and non-authoritative.
4. Redaction and presentation constraints belong downstream, not in the archival collector by default.
5. The collector should preserve evidence even when that is less convenient for querying.
6. Derived layers may simplify, normalize, index, redact, and optimize — but must not be confused with the archive itself.
7. Future collectors may differ internally, but they should feed a shared downstream ingestion philosophy rather than be prematurely forced into one identical collector runtime.

## What this means for CSJ right now

For the CSJ Collector specifically:
- keep improving archival correctness in the collector
- keep the filesystem archive authoritative
- treat `csj_jobs/` and related archive outputs as the canonical ingest source for future DB work
- build database import/redaction/search on top of that archive
- do not try to make the collector itself become the final query-serving product

## Practical maintainer test

When making a change, ask:

- does this improve the collector as an authoritative archive?
- or am I bending the collector toward downstream query convenience in a way that should really live in the derived database/UI layer?

If it is the second one, strongly consider keeping it out of the collector.
