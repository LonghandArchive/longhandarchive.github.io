---
title: "CSJ Collector Hardening Implementation Plan"
permalink: /archive/csj-collector/hardening-plan-full/
---

# CSJ Collector hardening — implementation plan

## Status

This plan is now a **reference/backlog document**, not the primary active execution guide.
The trimmed plan became the real execution guide, and its core hardening tranche has been completed.

### Completed from this plan in substance

The following items were completed during the trimmed-plan execution pass:
- run artifact paths in config/state bootstrap
- `run_id` generation
- per-run summary files
- telemetry module and run lifecycle telemetry
- explicit run status (`success` / `degraded` / `failed`)
- phase timing capture
- fixture-backed listing/detail parser tests
- golden normalized-output regression tests
- surfaced attachment/transcript and malformed JSON failures
- atomic JSON write helper
- atomic writes for state, summaries, and per-job records
- targeted verification
- full-suite verification and CLI smoke checks

### Still open / deferred

These items remain optional backlog work rather than immediate requirements:
- first-class `csj/run_model.py`
- typed runtime container
- typed fetch/result objects
- dedicated `csj/failures.py`
- richer stable/internal interface docs
- module invariants / run model architecture docs

### Environment corrections to this plan

- Use `uvx pytest ...` in this environment rather than plain `pytest`, because `pytest` is not installed directly.
- The skill directory is not currently a git repository, so the original per-task commit steps only apply if work is moved into a real repo clone.
- Before major collector refactors or changes touching orchestration, persistence, or output schema, pause cron jobs first and resume only after tests and CLI smoke checks pass.

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Add operational maturity to the CSJ Collector through observability, explicit run semantics, parser regression protection, typed runtime boundaries, and safer persistence.

**Architecture:** Keep the existing modular structure. Do not do another broad refactor. Add a thin observability/runtime layer around the current collector flow, then harden parser tests and persistence seams incrementally.

**Tech Stack:** Python, pytest, dataclasses, JSON/JSONL files, existing CSJ package modules.

---

## Delivery strategy

- **One small commit per task**
- Preserve:
  - `scripts/collector.py` as stable wrapper
  - `scripts/collector_impl.py` as compatibility bridge
  - existing workspace outputs unless deliberately extended
- Additive changes first
- No speculative multi-source abstraction

---

# Phase A — run identity and run summaries

## Task 1: Add run artifact paths to config

**Objective:** Introduce stable filesystem locations for per-run artifacts.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/config.py`
- Test: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py`

**Step 1: Write failing test**

Add assertions that config/state bootstrap exposes and creates:
- `RUNS_DIR`
- `RUN_EVENTS_FILE` (or equivalent if you choose file constant here)

Example test shape:
```python
from csj import config

def test_config_exposes_run_artifact_paths():
    assert config.RUNS_DIR.name == "csj_runs"
    assert config.RUN_EVENTS_FILE.name == "csj_run_events.jsonl"
```

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py -k run_artifact
```

Expected: FAIL — missing config constants.

**Step 3: Implement minimal code**

In `csj/config.py`, add:
- `RUNS_DIR = DATA_DIR / "csj_runs"`
- `RUN_EVENTS_FILE = DATA_DIR / "csj_run_events.jsonl"`

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py -k run_artifact
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/config.py tests/test_state_and_paths.py
git commit -m "feat: add config paths for run artifacts"
```

---

## Task 2: Ensure run directories are created during bootstrap

**Objective:** Make run artifact storage part of normal collector bootstrap.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/state.py`
- Test: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py`

**Step 1: Write failing test**

Add a test that patches paths to a temp dir and verifies `ensure_dirs()` creates:
- `RUNS_DIR`

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py -k ensure_dirs
```

Expected: FAIL — run dir not created.

**Step 3: Implement minimal code**

Update `ensure_dirs()` in `csj/state.py` to create `RUNS_DIR`.

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py -k ensure_dirs
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/state.py tests/test_state_and_paths.py
git commit -m "feat: create run artifact directories during bootstrap"
```

---

## Task 3: Add a first-class run model

**Objective:** Create explicit data structures for run state and summaries.

**Files:**
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run_model.py`
- Test: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_model.py`

**Step 1: Write failing test**

Add tests for:
- creating a run summary object
- serializing it to dict
- status field accepts expected values

Example:
```python
from csj.run_model import RunSummary

def test_run_summary_to_dict_contains_required_fields():
    summary = RunSummary(
        run_id="2026-04-20T12-00-00Z_test",
        mode="native",
        started_at="2026-04-20T12:00:00",
        completed_at=None,
        status="success",
        counts={},
        timings={},
        warnings=[],
        errors=[],
        anomaly_level="none",
    )
    data = summary.to_dict()
    assert data["run_id"] == "2026-04-20T12-00-00Z_test"
    assert data["status"] == "success"
```

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_model.py
```

Expected: FAIL — module missing.

**Step 3: Implement minimal code**

Create `csj/run_model.py` with dataclasses:
- `RunSummary`
- optional `RunStatus` constants or enum-lite strings

Keep it minimal and serialization-friendly.

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_model.py
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/run_model.py tests/test_run_model.py
git commit -m "feat: add run summary model"
```

---

## Task 4: Generate a run_id at collector startup

**Objective:** Make every run uniquely identifiable.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Test: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py`

**Step 1: Write failing test**

Add a test that exercises `scrape()` with monkeypatched subfunctions and asserts a generated run summary context includes `run_id`.

If direct testing is awkward, test a helper like:
- `build_run_id()`

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k run_id
```

Expected: FAIL

**Step 3: Implement minimal code**

In `csj/run.py`:
- add helper `build_run_id()`
- create run summary/state object at top of `scrape()`

Suggested format:
```python
2026-04-20T09-15-32Z_a1f93b
```

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k run_id
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/run.py tests/test_run_module.py
git commit -m "feat: generate run id for collector runs"
```

---

## Task 5: Write per-run summary files

**Objective:** Persist a durable summary per run while preserving `csj_latest.json`.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Test: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py`

**Step 1: Write failing test**

Add test for `write_run_summary()` asserting:
- latest file still written
- run-specific file written under `RUNS_DIR/{run_id}.json`

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k write_run_summary
```

Expected: FAIL — no run-specific file.

**Step 3: Implement minimal code**

Update `write_run_summary()` to:
- include `run_id`
- write summary to `LATEST_FILE`
- also write `RUNS_DIR / f"{run_id}.json"`

Keep current summary schema intact; add new fields rather than replacing old ones.

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k write_run_summary
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/run.py tests/test_run_module.py
git commit -m "feat: persist per-run summary artifacts"
```

---

# Phase B — telemetry and explicit run status

## Task 6: Add telemetry module for structured event emission

**Objective:** Introduce machine-readable operational events without removing human console output.

**Files:**
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/telemetry.py`
- Test: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_telemetry.py`

**Step 1: Write failing test**

Test that `emit_event()` writes one JSON line with fields:
- `event`
- `run_id`
- `timestamp`
- optional `phase`

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_telemetry.py
```

Expected: FAIL — module missing.

**Step 3: Implement minimal code**

Create helper(s):
- `emit_event(path, event, run_id, **fields)`
- optionally `utc_now_iso()`

Keep it tiny.

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_telemetry.py
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/telemetry.py tests/test_telemetry.py
git commit -m "feat: add structured telemetry event writer"
```

---

## Task 7: Emit run_started and run_completed events

**Objective:** Make run lifecycle visible to machines and operators.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Test: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py`

**Step 1: Write failing test**

Add a monkeypatched test that checks:
- `run_started` emitted when `scrape()` begins
- `run_completed` emitted on successful completion

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k telemetry
```

Expected: FAIL

**Step 3: Implement minimal code**

In `scrape()`:
- emit `run_started` after bootstrap
- emit `run_completed` before return
- include `run_id`, mode, status, counts if available

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k telemetry
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/run.py tests/test_run_module.py
git commit -m "feat: emit lifecycle telemetry for collector runs"
```

---

## Task 8: Add phase timing collection

**Objective:** Track where run time goes.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/native.py`
- Test: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py`

**Step 1: Write failing test**

Add assertions that run summaries include timing keys, for example:
- `startup_ms`
- `discovery_ms`
- `fetch_ms`
- `lifecycle_ms`
- `total_ms`

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k timings
```

Expected: FAIL

**Step 3: Implement minimal code**

Use `time.perf_counter()` around:
- discovery
- fetch execution
- lifecycle
- summary write
- total run

Keep native internals optional for now; do top-level phase timing first.

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k timings
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/run.py csj/native.py tests/test_run_module.py
git commit -m "feat: add phase timing to run summaries"
```

---

## Task 9: Make run status explicit

**Objective:** Classify runs as `success`, `degraded`, or `failed`.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run_model.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_status.py`

**Step 1: Write failing test**

Create tests for:
- success run
- degraded run when failure rate exceeds threshold
- failed run on fatal startup/search exception

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_status.py
```

Expected: FAIL

**Step 3: Implement minimal code**

Use current failure/anomaly data from `summarize_fetch_results()`:
- warning/high anomaly => degraded
- fatal exception => failed
- else success

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_status.py
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/run_model.py csj/run.py tests/test_run_status.py
git commit -m "feat: classify collector runs by explicit status"
```

---

# Phase C — parser drift protection

## Task 10: Add fixture directory and first listing fixture

**Objective:** Establish fixture-backed parser testing.

**Files:**
- Create directory: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/fixtures/listings/`
- Create: one representative HTML fixture
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_native_parser_fixtures.py`

**Step 1: Write failing test**

Test `_extract_listings()` against fixture and assert at least:
- count
- first reference
- title
- department

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_native_parser_fixtures.py -k listings
```

Expected: FAIL — missing fixture/test/module path.

**Step 3: Implement minimal code**

- add saved listing HTML fixture from a representative CSJ page
- write the fixture-based test
- avoid network access in tests

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_native_parser_fixtures.py -k listings
```

Expected: PASS

**Step 5: Commit**
```bash
git add tests/fixtures/listings tests/test_native_parser_fixtures.py
git commit -m "test: add fixture-backed listing parser regression test"
```

---

## Task 11: Add detail-page fixture tests

**Objective:** Protect detail parsing against HTML drift.

**Files:**
- Create directory: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/fixtures/details/`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_native_parser_fixtures.py`

**Step 1: Write failing test**

Add fixture tests for:
- standard detail page
- detail page with attachments
- detail page with embed/transcript links

If parsing logic is buried in `fetch_detail()`, test the current method with a stubbed session response.

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_native_parser_fixtures.py -k detail
```

Expected: FAIL

**Step 3: Implement minimal code**

Add fixtures and assertions for:
- `reference`
- `grade`
- `closes`
- `attachments`
- `embeds`
- `full_text` presence

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_native_parser_fixtures.py -k detail
```

Expected: PASS

**Step 5: Commit**
```bash
git add tests/fixtures/details tests/test_native_parser_fixtures.py
git commit -m "test: add detail page parser fixture coverage"
```

---

## Task 12: Add golden normalized-record tests

**Objective:** Catch silent output drift after parsing/normalization changes.

**Files:**
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_record_golden_outputs.py`
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/fixtures/golden/`

**Step 1: Write failing test**

Take representative parsed raw input and assert stable normalized output from:
- `normalize_job()`
- `finalize_job()`

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_record_golden_outputs.py
```

Expected: FAIL

**Step 3: Implement minimal code**

Add one or two canonical expected JSON outputs covering:
- salary parsing
- closes_iso
- grade normalization
- lifecycle defaults
- archive completeness defaults

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_record_golden_outputs.py
```

Expected: PASS

**Step 5: Commit**
```bash
git add tests/test_record_golden_outputs.py tests/fixtures/golden
git commit -m "test: add golden normalized record regression coverage"
```

---

# Phase D — typed runtime boundaries

## Task 13: Introduce a typed runtime container

**Objective:** Replace the large dict returned by `build_run_context()` with a typed object.

**Files:**
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/runtime.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/cli.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_cli_module.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py`

**Step 1: Write failing test**

Change/add test expectations so `build_run_context()` returns an object with attributes, e.g.:
- `ctx.SCHEMA_VERSION`
- `ctx.normalize_job`
- `ctx.save_job_record`

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_cli_module.py /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py
```

Expected: FAIL

**Step 3: Implement minimal code**

Create dataclass `CollectorRuntime` with the current fields from `build_run_context()`.

Update `build_run_context()` to return `CollectorRuntime`.

Update `csj/run.py` to use attributes instead of dict indexing in small, mechanical edits.

**Step 4: Run test to verify pass**

Run:
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_cli_module.py /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/runtime.py csj/cli.py csj/run.py tests/test_cli_module.py tests/test_run_module.py
git commit -m "refactor: replace dict runtime context with typed runtime object"
```

---

## Task 14: Add typed fetch result model

**Objective:** Reduce ad hoc dict returns in orchestration.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run_model.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Test: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py`

**Step 1: Write failing test**

Add tests expecting `execute_fetches()` to return an object with:
- `jobs`
- `failed`

**Step 2: Run test to verify failure**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k execute_fetches
```

Expected: FAIL

**Step 3: Implement minimal code**

Create `FetchResult` dataclass and update only the narrow return path from `execute_fetches()`.

**Step 4: Run test to verify pass**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k execute_fetches
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/run_model.py csj/run.py tests/test_run_module.py
git commit -m "refactor: type fetch results in run orchestration"
```

---

# Phase E — failure taxonomy and degraded-path visibility

## Task 15: Add failure taxonomy module

**Objective:** Replace vague failure strings with structured categories.

**Files:**
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/failures.py`
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_failures.py`

**Step 1: Write failing test**

Test a helper that builds structured failure records like:
- `detail_fetch_failed`
- `attachment_download_failed`
- `state_load_failed`

**Step 2: Run test to verify failure**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_failures.py
```

Expected: FAIL

**Step 3: Implement minimal code**

Create a tiny failure model and helper constructors/classifiers.

**Step 4: Run test to verify pass**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_failures.py
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/failures.py tests/test_failures.py
git commit -m "feat: add structured failure taxonomy"
```

---

## Task 16: Make attachment/transcript helper failures observable

**Objective:** Stop silently swallowing enrichment failures in `fetch_one_job()`.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py`

**Step 1: Write failing test**

Add tests where:
- `download_attachments()` raises
- `fetch_youtube_transcripts()` raises

Assert:
- run still completes
- warning/failure is counted or emitted
- result is marked degraded if threshold reached

**Step 2: Run test to verify failure**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k asset_failure
```

Expected: FAIL

**Step 3: Implement minimal code**

Replace:
```python
except Exception:
    pass
```
with:
- capture warning/failure event
- attach error detail to run summary/telemetry
- continue safely

**Step 4: Run test to verify pass**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k asset_failure
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/run.py tests/test_run_module.py
git commit -m "feat: surface asset enrichment failures in run telemetry"
```

---

## Task 17: Surface malformed JSON/state read problems

**Objective:** Make corrupted local data visible instead of silently ignored.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/records.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/state.py`
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_recovery_paths.py`

**Step 1: Write failing test**

Add tests for:
- malformed existing job file
- malformed state file
- malformed refresh target record

Assert:
- warning or failure emitted
- run does not crash unnecessarily

**Step 2: Run test to verify failure**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_recovery_paths.py
```

Expected: FAIL

**Step 3: Implement minimal code**

At each current swallow point:
- classify issue
- emit warning
- continue with safe fallback

**Step 4: Run test to verify pass**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_recovery_paths.py
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/run.py csj/records.py csj/state.py tests/test_recovery_paths.py
git commit -m "feat: surface malformed local state and record failures"
```

---

# Phase F — atomic writes and persistence hardening

## Task 18: Add atomic JSON write helper

**Objective:** Protect critical files from partial writes.

**Files:**
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/io_utils.py`
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_atomic_io.py`

**Step 1: Write failing test**

Test that helper writes JSON to a temp file then renames into place.

**Step 2: Run test to verify failure**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_atomic_io.py
```

Expected: FAIL

**Step 3: Implement minimal code**

Create:
- `atomic_write_json(path, data)`

**Step 4: Run test to verify pass**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_atomic_io.py
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/io_utils.py tests/test_atomic_io.py
git commit -m "feat: add atomic json write helper"
```

---

## Task 19: Use atomic writes for state and run summaries

**Objective:** Harden the most important top-level artifacts first.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/state.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Test: existing tests + `tests/test_atomic_io.py`

**Step 1: Write failing test**

Add tests asserting the helper is used indirectly by:
- `save_state()`
- `write_run_summary()`

**Step 2: Run test to verify failure**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py
```

Expected: FAIL

**Step 3: Implement minimal code**

Replace direct `write_text()` with `atomic_write_json()` for:
- state
- latest summary
- per-run summary

**Step 4: Run test to verify pass**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/state.py csj/run.py tests/test_state_and_paths.py tests/test_run_module.py
git commit -m "feat: use atomic writes for state and run summaries"
```

---

## Task 20: Use atomic writes for per-job records

**Objective:** Protect core archival records from truncation/corruption.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/records.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_records.py`

**Step 1: Write failing test**

Add test that `save_job_record()` writes valid JSON through the atomic helper path.

**Step 2: Run test to verify failure**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_records.py
```

Expected: FAIL

**Step 3: Implement minimal code**

Replace `fpath.write_text(...)` with `atomic_write_json(...)`.

**Step 4: Run test to verify pass**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_records.py
```

Expected: PASS

**Step 5: Commit**
```bash
git add csj/records.py tests/test_records.py
git commit -m "feat: use atomic writes for job records"
```

---

# Phase G — docs and guardrails

## Task 21: Document stable vs internal interfaces

**Objective:** Clarify what future changes can safely break.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/references/csj-collector-architecture.md`

**Step 1: Draft section**

Add headings:
- Stable interfaces
- Internal interfaces
- Compatibility bridge policy

**Step 2: Add concrete statements**

Stable:
- `scripts/collector.py`
- workspace artifact family
- output semantics users rely on

Internal:
- `collector_impl.py`
- helper signatures
- module organization inside `csj/`

**Step 3: Review**
No test needed; review for clarity.

**Step 4: Commit**
```bash
git add references/csj-collector-architecture.md
git commit -m "docs: define stable and internal collector interfaces"
```

---

## Task 22: Document module invariants and run model

**Objective:** Prevent future boundary drift.

**Files:**
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/references/csj-collector-architecture.md`

**Step 1: Add module ownership table**

For:
- `csj/native.py`
- `csj/run.py`
- `csj/records.py`
- `csj/cli.py`
- `csj/state.py`
- `scripts/collector_impl.py`

**Step 2: Add run architecture section**
Document:
- `run_id`
- per-run summaries
- structured events
- run statuses
- degraded semantics

**Step 3: Commit**
```bash
git add references/csj-collector-architecture.md
git commit -m "docs: add module invariants and run model architecture"
```

---

# Final verification stage

## Task 23: Run targeted suite for new work

**Objective:** Verify the new hardening work before full-suite run.

**Files:** none

**Step 1: Run targeted tests**
```bash
pytest -q \
  /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_model.py \
  /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_telemetry.py \
  /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_status.py \
  /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_native_parser_fixtures.py \
  /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_record_golden_outputs.py \
  /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_recovery_paths.py \
  /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_atomic_io.py
```

Expected: PASS

**Step 2: Commit if needed**
```bash
git add -A
git commit -m "test: verify hardening layers with targeted suite"
```

---

## Task 24: Run full suite and CLI smoke checks

**Objective:** Confirm collector health after hardening.

**Files:** none

**Step 1: Run full test suite**
```bash
pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests
```

Expected: PASS

**Step 2: Run CLI smoke checks**
```bash
python3 /root/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --help
python3 /root/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --repair-lifecycle --dry-run
```

Expected: both pass

**Step 3: Commit**
```bash
git add -A
git commit -m "chore: verify collector after observability and hardening work"
```

---

# Recommended implementation order

If you want the smartest execution order, do these first:

1. Task 1
2. Task 2
3. Task 3
4. Task 4
5. Task 5
6. Task 6
7. Task 7
8. Task 8
9. Task 9
10. Task 10
11. Task 11
12. Task 12
13. Task 13
14. Task 16
15. Task 17
16. Task 18
17. Task 19
18. Task 20
19. Task 21
20. Task 22
21. Task 23
22. Task 24

That sequence gives you:
- observability first
- parser protection second
- runtime typing third
- persistence hardening fourth
- docs last

---

# Minimum cut if you want to stop early

If you only do the highest-value subset, stop after:

- Task 5
- Task 9
- Task 12
- Task 17
- Task 20
- Task 24

That would still materially improve the collector.
