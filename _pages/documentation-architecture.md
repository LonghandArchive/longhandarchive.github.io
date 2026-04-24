---
title: "Documentation Architecture"
---

# Documentation architecture for the CSJ Collector

This document defines how project documentation should be managed inside this skill.

The goal is simple:
make the skill folder behave like the documentation root of a well-maintained open-source project.

## Why this exists

Before this consolidation, important documentation was split across:
- `SKILL.md`
- `references/`
- multiple workspace docs under `~/.hermes/workspace/csj/`
- historical plans under `~/.hermes/workspace/csj/plans/`

That made the project understandable, but not yet well-homed.
A maintainer should be able to open one project root and understand:
- what this project is
- what it is for
- how it is organized
- what is still under review
- which docs are current versus historical

This skill folder is now that root.

## The documentation layers

Think in layers.

### Layer 1: Canonical maintainer entrypoints

These are the first docs a maintainer should read:
- `README.md`
- `ROADMAP.md`
- `docs/documentation-architecture.md`
- `docs/current-truth-architecture.md`
- `docs/workspace-outputs.md`
- `docs/maintainer-guide.md`
- `docs/current-status.md`
- `docs/archive-accuracy-assessment.md`
- `docs/local-first-tiering-architecture.md`
- `docs/local-first-tiering-implementation-plan.md`
- `docs/point-in-time-reconstruction.md`
- `docs/historical-snapshot-normalization-policy.md`
- `docs/authoritative-collector-and-derived-layers.md`

These should stay short, current, and curated.

Responsibilities:
- `README.md` explains the project and doc map
- `ROADMAP.md` explains priorities and open items
- `documentation-architecture.md` explains how docs are managed
- `current-truth-architecture.md` states the shortest canonical architecture and invariants that should match live code
- `workspace-outputs.md` explains what the collector produces on disk
- `maintainer-guide.md` explains how to change and verify the project safely
- `current-status.md` records the concise live status and important current findings
- `archive-accuracy-assessment.md` evaluates how trustworthy the collector is as a historical archive and where stronger archival guarantees are still needed
- `local-first-tiering-architecture.md` explains the local-disk tier model and how it stays future-offload-compatible
- `local-first-tiering-implementation-plan.md` turns the tiering model into concrete next engineering steps
- `point-in-time-reconstruction.md` defines how to answer historical questions without conflating current projections, manifests, snapshots, and inference
- `historical-snapshot-normalization-policy.md` defines the default rule that immutable historical evidence should not be rewritten in place for metadata-only uniformity
- `authoritative-collector-and-derived-layers.md` defines the architectural boundary between the authoritative archive layer and future derived database/UI layers

### Layer 2: Durable technical references

These capture concepts that should remain valid across multiple implementation changes:
- `references/csj-collector-architecture.md`
- `references/csj-archive-envelope.md`
- `references/csj-field-mapping.md`
- `references/refresh-lifecycle-edge-cases.md`

These are not the place for temporary project status.
They are for architectural concepts, domain rules, and technical invariants.

### Layer 3: Operator/developer usage guide

- `SKILL.md`

This is the Hermes-oriented operating manual.
It should remain strong on:
- how to run the collector
- output schema and paths
- known gotchas
- current code layout
- implementation guidance for future changes

It is not the best place to accumulate every project-status note or roadmap thread.

### Layer 4: Historical planning archive

Imported from the workspace and now stored under:
- `docs/imported-workspace/`
- `docs/imported-workspace/plans/`

These docs are valuable because they capture:
- why design decisions were made
- how the collector evolved
- what previous implementation phases aimed to achieve

But they are historical source material, not the main front door.

## Canonical vs historical

Use this rule:

### Canonical docs
A doc is canonical if a maintainer should rely on it first.
Canonical docs should be:
- short
- curated
- updated when behavior or project direction changes
- linked from `README.md`

### Historical docs
A doc is historical if it preserves context about past thinking or execution.
Historical docs should be:
- retained
- clearly located in an archive-like place
- allowed to be more verbose or phase-specific
- treated as supporting context, not the current source of truth

## What should go where

### Put it in `README.md` if it answers:
- what is this project?
- what are the main docs?
- what are the main code modules?

### Put it in `ROADMAP.md` if it answers:
- what still needs review?
- what are the current priorities?
- what is explicitly deferred?

### Put it in `references/` if it answers:
- what architectural or domain concept should remain true over time?
- what nuance about lifecycle, schema, or archive modeling needs a stable note?

### Put it in `SKILL.md` if it answers:
- how do I run or operate the collector inside Hermes?
- what flags, paths, schemas, or gotchas matter operationally?

### Put it in `docs/imported-workspace/` if it answers:
- what did we previously plan or discover?
- what historical context should be preserved but not promoted to canon?

## Maintainer rules for future docs

1. Prefer updating an existing canonical doc over creating another free-floating status file.
2. Do not create new important project docs only in `~/.hermes/workspace/csj/`.
3. If a workspace note matters long-term, import or absorb it into the skill.
4. Keep roadmap and architecture separate:
   - roadmap = priorities and open items
   - architecture = what the system is and how it is shaped
5. Keep status notes from overrunning architecture docs.
6. Keep conceptual Longhand Archive thinking grounded in current CSJ reality.

## Practical maintainer workflow

When making a meaningful change:

1. Update code/tests.
2. Update `SKILL.md` if operator behavior changed.
3. Update `references/` if a durable architectural rule changed.
4. Update `ROADMAP.md` if priorities or open questions changed.
5. If the change closes or supersedes an imported historical plan, note that in canonical docs rather than editing history out of existence.

## Recommended future additions

If the project continues maturing, the next useful docs would be:
- `docs/maintainer-guide.md`
- `docs/testing-strategy.md`
- `docs/run-artifacts-and-observability.md`

Those should only be added if they are actively maintained.
Better a small, current doc set than a large stale one.
