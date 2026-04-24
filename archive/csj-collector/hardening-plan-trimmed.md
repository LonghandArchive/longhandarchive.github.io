---
title: "CSJ Collector Hardening Trimmed Plan"
permalink: /archive/csj-collector/hardening-plan-trimmed/
---

# CSJ Collector hardening — trimmed plan

## Status

### Completed

The original **Do now** hardening tranche is complete and verified.

Completed work:
- added `run_id` for each collector run
- added per-run summaries in `~/.hermes/workspace/csj/csj_runs/`
- preserved `csj_latest.json` as latest snapshot
- added `csj/telemetry.py` with JSONL event output
- added explicit run statuses: `success`, `degraded`, `failed`
- surfaced major previously-silent failures in `csj/run.py`, `csj/records.py`, and `csj/state.py`
- added parser fixtures for listing, normal detail, detail-with-assets, and missing/unavailable cases
- added golden normalized-output regression coverage
- added `atomic_write_json()` and applied it to critical JSON outputs
- verified with targeted tests, full suite, and CLI smoke checks

Verification completed:
- targeted hardening suite passed
- full test suite passed: `69 passed`
- `python3 scripts/collector.py --help` passed
- `python3 scripts/collector.py --repair-lifecycle --dry-run` passed

### Remaining

Only optional cleanup remains from this plan:
- typed runtime object
- typed result objects
- broader module-level telemetry
- richer architecture docs / invariants

Operational follow-up now matters more than further hardening:
- review lifecycle repair candidates
- decide whether to apply repair/backfill changes
- resume paused cron jobs when comfortable

### Environment notes

- In this environment, tests were run with `uvx pytest ...`, not plain `pytest`, because `pytest` was not installed directly.
- The skill directory is not currently a git repository, so the original plan's commit steps are not directly applicable here.
- Before major collector refactors or changes to orchestration/persistence/schema outputs, pause cron jobs first and resume only after tests and CLI smoke checks pass.

## Must do now

These are the changes with the best payoff-to-complexity ratio.

### 1. Run identity and durable run artifacts
Implement:

- `run_id` for every run
- per-run summary files under something like:
  - `~/.hermes/workspace/csj/csj_runs/{run_id}.json`
- keep `csj_latest.json` as the latest snapshot

### Why
This is the backbone for:
- debugging
- cron review
- comparing runs
- anomaly tracking

### Keep it simple
You do **not** need an elaborate run framework first.
A lightweight summary object or even a disciplined dict is enough initially.

---

### 2. Structured telemetry
Add:

- a small `csj/telemetry.py`
- JSONL event output
- at minimum:
  - `run_started`
  - `run_completed`
  - `run_warning`
  - `run_failed`
  - `detail_fetch_failed`

### Why
This gives you real observability without changing the collector’s core logic much.

### Keep it simple
Start with:
- file-based JSONL events
- keep existing human `print()` output

Do not try to build a logging platform.

---

### 3. Explicit run status
Every run should end as one of:

- `success`
- `degraded`
- `failed`

### Why
This is more important than lots of sub-classification.

### Suggested rule
- `failed`: collector could not meaningfully run
- `degraded`: collector completed but with meaningful failure volume or subsystem loss
- `success`: normal completion

---

### 4. Surface swallowed failures
Clean up the highest-value silent failure points, especially in:

- `csj/run.py`
- `csj/records.py`
- `csj/state.py`

Focus on places where the code currently does some form of:
- `except Exception: pass`
- malformed JSON fallback with no visibility
- attachment/transcript failure suppression

### Why
This is one of the biggest trust gaps in the current architecture.

### Important
Do **not** try to eliminate every broad exception immediately.
Just make swallowed failures:
- visible
- counted
- attached to run summary / telemetry

---

### 5. Parser fixture tests
Add only a **small initial fixture set**:

#### Listing fixtures
- one representative search results page

#### Detail fixtures
- one normal detail page
- one detail page with attachments
- one anomalous/missing/withdrawn page

### Why
This protects against the most likely real-world breakage: upstream CSJ drift.

### Important
Do not overbuild the fixture corpus initially.

---

### 6. Golden normalized-output tests
Add 1–2 tests that cover:

- `normalize_job()`
- `finalize_job()`

and compare against stable expected JSON-ish outputs.

### Why
This catches silent output drift better than lots of helper-level tests.

---

### 7. Atomic writes for critical JSON files
Add a tiny helper, e.g.:
- `atomic_write_json(path, data)`

Use it for:
- state file
- latest summary
- per-run summary
- per-job records

### Why
This is high-value and easy to justify.

### Keep it simple
Just:
- write temp file
- `os.replace`

No need for advanced durability engineering right now.

---

## Later if still needed

These are good ideas, but only after the core hardening lands.

### 8. Typed runtime object
Replacing `build_run_context()` dict with a typed `CollectorRuntime` object is nice.

### Why later
It improves maintainability, but it is not the main operational risk today.

---

### 9. Typed fetch/run result objects
Adding:
- `FetchResult`
- `RunSummary` dataclasses
- maybe lifecycle count objects

is good cleanup.

### Why later
Useful, but not as urgent as:
- observability
- parser protection
- atomic writes

---

### 10. Broader module-level telemetry
Instrumenting:
- `csj/assets.py`
- `csj/history.py`
- `csj/state.py`
- `csj/records.py`

with richer structured events may be worth it later.

### Why later
You’ll probably get 80% of the value from instrumenting:
- `csj/run.py`
- `csj/native.py`

first.

---

### 11. Richer run architecture docs
Good to do after the implementation settles:
- stable vs internal interfaces
- module invariants
- run model docs

### Why later
Docs are more useful once the actual shape of the hardening is real.

---

## Drop unless pain appears

These are not bad ideas, but I would not schedule them now.

### 12. Dedicated `csj/failures.py`
A separate failure taxonomy module is only worth it if:
- classification logic starts repeating a lot
- several modules need shared structured failure constructors

For now, structured error records inside telemetry/run summary are enough.

---

### 13. Deep formal failure hierarchy
You do not need:
- many failure subclasses
- complex severity models
- elaborate domain failure trees

Right now that would be architecture overhead.

---

### 14. Large fixture corpus
Do **not** try to build a giant test archive immediately.

Start with:
- 3–4 fixtures
- high-value cases only

Expand only when real breakages justify it.

---

### 15. Broad recovery-playbook engineering
Documenting every theoretical failure mode can wait.

Only harden/document recovery paths that are:
- already observed
- or clearly likely

---

# Recommended revised sequence

## Phase 1 — observability MVP
1. add run artifact paths
2. add `run_id`
3. write per-run summaries
4. add `csj/telemetry.py`
5. emit `run_started` / `run_completed`
6. add timings
7. add explicit run status

## Phase 2 — trust and drift protection
8. surface major swallowed failures
9. add 3–4 parser fixtures
10. add listing/detail parser tests
11. add 1–2 golden normalized-output tests

## Phase 3 — persistence hardening
12. add `atomic_write_json()`
13. use it for state/summaries/job records
14. surface malformed local JSON/state as warnings

## Phase 4 — optional cleanup
15. decide whether typed runtime object is still worth doing
16. decide whether typed result objects are still worth doing
17. update architecture docs

---

# Clear stopping condition

I’d define the hardening pass as complete when all of these are true:

- every run has a `run_id`
- every run writes a per-run summary file
- runs end with explicit `success` / `degraded` / `failed`
- key failures are surfaced instead of silently swallowed
- parser fixtures cover the core listing + detail paths
- normalized-output regression tests exist
- critical JSON writes are atomic
- full suite passes
- CLI smoke checks pass

Once you reach that, stop and reassess before doing more architecture cleanup.

---

# Short version

## Do now
- run_id
- per-run summaries
- telemetry
- explicit run status
- surface silent failures
- parser fixtures
- golden output tests
- atomic writes

## Later if needed
- typed runtime object
- typed result models
- richer docs/invariants

## Drop for now
- dedicated failure module
- large fixture corpus
- elaborate failure hierarchy
