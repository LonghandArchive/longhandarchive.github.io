---
title: "CSJ Modular Refactor Plan"
permalink: /archive/csj-collector/modular-refactor-plan/
---

# CSJ Skill Modular Refactor Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Status (2026-04-19):** Core modular refactor is now effectively complete.

**Outcome:** This skill should now be treated as the **CSJ Collector**. It is a native-only modular collector for Civil Service Jobs, developed in the context of **The Longhand Archive**. The stable public entrypoint remains `scripts/collector.py`, but most logic has been moved into focused modules under `csj/`.

**Working approach:** Keep architecture, planning, implementation, and documentation work inside this skill until a second real collector exists and shared patterns are proven. Evolve toward broader archive concepts from tested CSJ Collector behavior rather than speculative multi-source abstractions.

**Current Architecture:**
- `scripts/collector.py` — stable public entrypoint wrapper
- `scripts/collector_impl.py` — thin integration layer / top-level scrape flow
- `csj/run.py` — collector orchestration pipeline, including the top-level scrape flow
- `csj/native.py` — native requests + ALTCHA client
- `csj/assets.py`, `csj/lifecycle.py`, `csj/history.py`, `csj/hashing.py`, `csj/normalize.py`, `csj/records.py`, `csj/state.py`, `csj/config.py`

**Tech Stack:** Python 3.11+, existing skill at `~/.hermes/skills/research/civil-service-jobs-collector/`, pytest, requests, optional markitdown / youtube-transcript-api.

---

## Non-Negotiable Constraints

1. **The CSJ system must continue functioning fully as a Hermes skill throughout the refactor.**
2. **Keep `scripts/collector.py` working** as the canonical entrypoint used by the skill docs and cron jobs.
3. **Do not change output paths** under `~/.hermes/workspace/csj/`.
4. **Do not change current JSON schema or lifecycle semantics** unless a task explicitly says otherwise.
5. **Do not break current cron jobs** (`b7b55c30587d`, `0f2135aa9efa`) — they should still call the same script path.
6. **Refactor under TDD**: every new module extraction step starts with a failing test or a strengthened existing regression test.
7. **Prefer small extractions** over a big-bang rewrite.

---

## Current Structure Snapshot

Current modular structure is:
- `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector_impl.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/config.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/state.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/hashing.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/history.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/lifecycle.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/assets.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/normalize.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/records.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/native.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/csj/cli.py`

Current tests include:
- `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_asset_runtime.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_cli_wrapper.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_hashing.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_lifecycle.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_normalize.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_repair_mode.py`
- `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py`

## Final Internal Package Layout

Implemented internal package:

```text
~/.hermes/skills/research/civil-service-jobs-collector/csj/
├── __init__.py
├── cli.py
├── config.py
├── state.py
├── hashing.py
├── history.py
├── lifecycle.py
├── assets.py
├── normalize.py
├── native.py
├── run.py
└── collector_impl.py (indirectly via scripts wrapper, not inside csj/)
```

Public entrypoint remains:

```text
~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py
```

Current wrapper/import chain is effectively:

```python
scripts/collector.py -> csj.cli.main() -> scripts/collector_impl.py::scrape()
```

`BrowserCollector` / browser-use support is intentionally no longer part of the target layout.

---

# Full Test Plan

## Test Strategy Overview

The refactor is successful only if three layers remain green:

1. **Unit/regression tests** for extracted helpers
2. **CLI smoke tests** for the public entrypoint
3. **Behavioural parity checks** on summary / repair logic and no-regression outputs

## Test Layers

### Layer A — Existing regression tests must stay green
These are the baseline guardrails and must pass after every task:

```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
```

Expected: all pass.

### Layer B — New module-import tests
Add tests that verify the wrapper script still exposes the same callable behaviour after moving code into `csj/`.

Examples:
- importing `scripts/collector.py` still works
- importing `csj.lifecycle`, `csj.assets`, `csj.normalize` works directly
- CLI `--help` still works

### Layer C — Extraction parity tests
For each extracted cluster, add tests that compare old expected behaviour with new module calls.

Examples:
- lifecycle classification parity
- repair evaluation parity
- hash/diff parity for sample job dicts
- normalization parity for salary, grade, close date, location

### Layer D — Smoke tests for CLI entrypoint
Run after major milestones:

```bash
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --help
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --repair-lifecycle --dry-run
python3 -m py_compile ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py
```

Expected:
- `--help` prints usage
- repair dry-run completes without import/path errors
- py_compile passes

### Layer E — Optional live smoke test after final integration
Run one safe real-world command:

```bash
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --repair-lifecycle --dry-run
```

Then optionally a normal scrape command if desired:

```bash
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --details -w 5 --full --refresh
```

Use only after all unit tests are green.

---

## New Test Files To Add

```text
~/.hermes/skills/research/civil-service-jobs-collector/tests/test_cli_wrapper.py
~/.hermes/skills/research/civil-service-jobs-collector/tests/test_hashing.py
~/.hermes/skills/research/civil-service-jobs-collector/tests/test_normalize.py
~/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py
```

Keep existing:
- `test_lifecycle.py`
- `test_repair_mode.py`

## Specific Behaviour To Test

### Hashing / diff tests
- `normalize_url_for_diff()` strips volatile CSJ query state consistently
- `normalize_asset_source_url()` preserves stable attachment identity
- `compute_field_hashes()` returns stable results for same input
- `compute_content_hash()` changes when meaningful fields change
- `diff_job_records()` distinguishes meaningful vs non-meaningful changes

### Normalization tests
- salary parsing single value / range
- grade normalization for common raw strings
- close-date parsing
- location cleanup
- security clearance extraction
- working pattern normalization

### State/path tests
- data directories point to current workspace locations
- lock file path is unchanged
- latest/state/event file paths are unchanged
- script wrapper adds the correct package root to `sys.path`

### CLI wrapper tests
- `scripts/collector.py --help` works
- `scripts/collector.py --repair-lifecycle --dry-run` reaches package entrypoint
- module import path works when executed from outside the skill root

### Lifecycle tests
Continue existing coverage plus add:
- module import path for `csj.lifecycle`
- no behaviour drift after extraction

---

# Implementation Tasks

### Task 1: Create the internal package skeleton

**Objective:** Introduce a package structure without changing runtime behaviour.

**Files:**
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/__init__.py`
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/config.py`
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/cli.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`
- Test: `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_cli_wrapper.py`

**Step 1: Write failing test**

Add a CLI smoke test like:

```python
import subprocess


def test_collector_wrapper_help_runs():
    result = subprocess.run(
        [
            "python3",
            "/root/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py",
            "--help",
        ],
        capture_output=True,
        text=True,
    )
    assert result.returncode == 0
    assert "Civil Service Jobs Collector" in result.stdout
```

**Step 2: Run test to verify failure**

Run:
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests/test_cli_wrapper.py -v
```

Expected: FAIL until wrapper/package is created.

**Step 3: Write minimal implementation**
- add empty package
- move only argparse/main wiring into `csj.cli`
- make `scripts/collector.py` a thin wrapper importing `csj.cli.main`
- for now, `csj.cli` can still call back into legacy functions in `scripts/collector.py` if needed via an intermediate compatibility layer, but avoid circular imports

**Step 4: Run tests**

```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests/test_cli_wrapper.py -v
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
python3 -m py_compile ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py
```

Expected: all pass.

**Step 5: Commit**

```bash
git add ~/.hermes/skills/research/civil-service-jobs-collector
git commit -m "refactor: add csj package skeleton and wrapper entrypoint"
```

---

### Task 2: Extract configuration and path constants

**Objective:** Centralize constants and file paths without changing values.

**Files:**
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/config.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`
- Test: `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py`

**Step 1: Write failing test**

Add assertions such as:

```python
from csj import config


def test_config_paths_match_existing_workspace_layout():
    assert str(config.DATA_DIR).endswith('/.hermes/workspace/csj')
    assert str(config.JOBS_DIR).endswith('/.hermes/workspace/csj/csj_jobs')
    assert config.SCHEMA_VERSION == '2.2'
```

**Step 2: Verify failure**
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py -v
```

**Step 3: Implement**
- move version constants, directory constants, `GRADE_VALUES`, `GRADE_NORMALIZE`, `DEPT_ALIASES`, regex patterns into `csj.config`
- import them back into the legacy flow

**Step 4: Verify**
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py -v
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
```

---

### Task 3: Extract state and lock management

**Objective:** Move filesystem bootstrapping/state I/O into a small stable module.

**Files:**
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/state.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`
- Test: `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py`

**Functions to move:**
- `ensure_dirs`
- `load_state`
- `save_state`
- `acquire_lock`
- `release_lock`

**Tests to add:**
- lock file path unchanged
- state save/load roundtrip
- ensure_dirs creates expected directories under temp monkeypatched paths if practical

**Verification:**
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py -v
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
```

---

### Task 4: Extract hashing and diff logic

**Objective:** Make content hashing reusable and independently testable.

**Files:**
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/hashing.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`
- Test: `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_hashing.py`

**Functions to move:**
- `stable_json_dumps`
- `normalize_text_for_diff`
- `normalize_list_for_diff`
- `normalize_url_for_diff`
- `normalize_asset_source_url`
- `strip_volatile_asset_fields`
- `build_comparable_record`
- `hash_value`
- `compute_field_hashes`
- `compute_content_hash`
- `diff_job_records`

**Tests:**
- unchanged input => same content hash
- meaningful field change => different content hash
- volatile URL query change alone does not create false diff
- asset path noise is ignored correctly

**Verification:**
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests/test_hashing.py -v
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
```

---

### Task 5: Extract normalization helpers

**Objective:** Isolate parsing/cleanup logic from scraping flow.

**Files:**
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/normalize.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`
- Test: `~/.hermes/skills/research/civil-service-jobs-collector/tests/test_normalize.py`

**Functions to move:**
- `null_if_empty`
- `parse_salary_int`
- `normalize_grade`
- `normalize_working_pattern`
- `extract_security_clearance`
- `parse_closes_iso`
- `clean_location_primary`
- any tiny pure helpers they depend on

**Test cases:**
- salary range parsing
- single salary parsing
- common grade aliases
- ISO close date parsing from CSJ strings
- location cleanup edge cases
- clearance extraction from free text

**Verification:**
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests/test_normalize.py -v
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
```

---

### Task 6: Extract history/event persistence

**Objective:** Separate write-side persistence from orchestration.

**Files:**
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/history.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`
- Extend tests: existing lifecycle tests + path/state tests

**Functions to move:**
- `append_event`
- `write_history_version`
- `save_job_record`

**Testing approach:**
Use temporary directory monkeypatching where possible so tests assert:
- event file append shape
- history snapshot file naming exists
- first save vs changed save produce expected history/event writes

**Verification:**
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
```

---

### Task 7: Extract lifecycle logic

**Objective:** Move the newly-stabilized lifecycle logic into its own module without behaviour drift.

**Files:**
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/lifecycle.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`
- Existing tests: `test_lifecycle.py`, `test_repair_mode.py`

**Functions to move:**
- `verify_missing_job_url`
- `is_probable_csj_homepage`
- `classify_fetch_failure`
- `compute_search_refs`
- `evaluate_repair_action`
- `run_lifecycle_repair`

**Critical tests:**
- all existing lifecycle and repair tests stay green
- add one CLI dry-run smoke test if not already present

**Verification:**
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests/test_lifecycle.py -v
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests/test_repair_mode.py -v
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --repair-lifecycle --dry-run
```

---

### Task 8: Extract asset and archival logic

**Objective:** Isolate the noisiest subsystem without changing storage semantics.

**Files:**
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/assets.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`
- Optionally update: `scripts/migrate_attachment_pool.py` to import shared helpers only after the main refactor is stable

**Functions to move:**
- `classify_asset_url`
- `extract_supporting_assets_from_html`
- filename/hash/path helpers
- manifest/history helpers
- `download_attachments`
- `fetch_youtube_transcripts`
- asset enrichment helpers

**Test plan:**
Start with pure helper coverage first.
Suggested tests:
- download.cgi classified as pdf
- youtube links extracted as embeds
- asset source normalization stable
- pooled-path generation deterministic
- archive completeness logic stable for small synthetic job dicts

**Note:** This is the highest-risk extraction because it has the most filesystem side effects. Keep steps small.

**Verification:**
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
python3 -m py_compile ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py
```

---

### Task 9: Finish native collector boundary cleanup

**Objective:** Finish separating native transport/auth/search/detail parsing concerns from the remaining CLI-facing glue.

**Files:**
- Continue refining: `~/.hermes/skills/research/civil-service-jobs-collector/csj/native.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector_impl.py`
- Modify as needed: `~/.hermes/skills/research/civil-service-jobs-collector/csj/cli.py`

**Focus areas:**
- keep CSJ-native HTTP/session/auth/search/detail parsing in `csj/native.py`
- remove any remaining transport/parsing leakage from `scripts/collector_impl.py`
- do not reintroduce a browser path

**Test plan:**
Do not attempt full integration tests for live scraping here. Instead:
- keep existing regression tests green
- add import smoke tests
- verify CLI `--help` and repair mode still work
- only after extraction, optionally run a single live smoke command manually

**Verification:**
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --help
```

Optional manual smoke test:
```bash
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --repair-lifecycle --dry-run
```

---

### Task 10: Extract orchestration and leave the script as a thin wrapper

**Objective:** Finish the modularization while preserving the public entrypoint.

**Files:**
- Create: `~/.hermes/skills/research/civil-service-jobs-collector/csj/orchestration.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/csj/cli.py`
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`

**Functions to move:**
- `finalize_job`
- `normalize_job`
- `scrape`
- possibly small internal fetch orchestration helpers

**End state:**
- `scripts/collector.py` is wrapper-only
- all implementation lives in `csj/`
- public CLI behaviour is unchanged

**Verification:**
```bash
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
python3 -m py_compile ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --help
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --repair-lifecycle --dry-run
```

Optional final live smoke test:
```bash
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --details -w 5 --full --refresh
```

---

### Task 11: Refactor `migrate_attachment_pool.py` only if it reduces duplication cleanly

**Objective:** Share asset-path/hash helpers where useful without creating coupling hazards.

**Files:**
- Modify optionally: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/migrate_attachment_pool.py`
- Possibly import from: `~/.hermes/skills/research/civil-service-jobs-collector/csj/assets.py`

**Rule:**
Only do this if the shared code is clearly stable. Do **not** force this extraction if it risks entangling the migration utility with collector runtime dependencies.

**Verification:**
```bash
python3 -m py_compile ~/.hermes/skills/research/civil-service-jobs-collector/scripts/migrate_attachment_pool.py
pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests
```

---

### Task 12: Documentation update for module layout and stable entrypoints

**Objective:** Document the new internal structure without changing how the skill is operated.

**Files:**
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/SKILL.md`
- Modify: `~/.hermes/workspace/csj/CSJ-ARCHITECTURE.md`
- Optional create: `~/.hermes/skills/research/civil-service-jobs-collector/README.md`

**Docs must state explicitly:**
- `scripts/collector.py` remains the stable public entrypoint
- cron jobs should continue calling the same path
- internal implementation now lives in `csj/`
- dependencies unchanged unless separately documented

**Verification:**
- Manually read docs for consistency
- confirm they match live cron schedules and current repair-mode capability

---

# Final Verification Checklist

Before declaring the modular refactor complete:

- [ ] `scripts/collector.py` still works as the public skill entrypoint
- [ ] both cron jobs can keep using the same script path with no prompt changes needed
- [ ] all existing tests pass
- [ ] all new tests pass
- [ ] `python3 -m py_compile` passes on wrapper and moved modules
- [ ] `--help` works
- [ ] `--repair-lifecycle --dry-run` works
- [ ] at least one safe real-world smoke test runs successfully
- [ ] docs reflect live cron schedule and repair mode
- [ ] no output paths or schema fields changed unintentionally

---

# Recommended Execution Order

1. Package skeleton + wrapper
2. Config/constants
3. State/locks
4. Hashing/diff
5. Normalization
6. History/events
7. Lifecycle
8. Assets
9. Native collector boundary cleanup
10. Orchestration
11. Optional migration utility cleanup
12. Docs

This order front-loads low-risk, pure-function extractions and delays high-risk I/O and collector-class movement until the guardrails are stronger.

---

## Live execution notes

- Implemented the first milestone from this plan.
- `scripts/collector.py` is now a stable wrapper that imports `csj.cli.main` while re-exporting legacy symbols from `scripts/collector_impl.py` for test compatibility during the transition.
- Added internal package skeleton under `csj/` and completed extractions for:
  - `config.py`
  - `state.py`
  - `hashing.py`
  - `normalize.py`
  - `history.py`
  - `lifecycle.py`
  - `assets.py` (including manifest/enrichment/path/archive helpers and live download/transcript routines)
- Current test status after these extractions:
  - `pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests` → **45 passed**
  - `python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --repair-lifecycle --dry-run` → **passed**
  - `python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --help` → **passed**
- Remaining higher-risk cleanup still to do:
  - finish native boundary cleanup between `csj/native.py` and `scripts/collector_impl.py`
  - continue thinning orchestration/CLI glue where useful without breaking the stable entrypoint
- Note: dry-run repair counts can change naturally over time as closing dates pass. During the latest smoke check it reported **1** closeable stale active job (`455795`), which is expected runtime drift rather than a refactor regression.

# Risks To Watch

1. **Import path breakage** — the wrapper must work when invoked directly by cron.
2. **Circular imports** — especially among config/state/history/assets/orchestration.
3. **Filesystem side effects in tests** — use temp dirs or monkeypatch aggressively.
4. **Asset subsystem coupling** — likely the hardest extraction.
5. **Docs drift** — keep plan/status docs aligned with the now-native-only collector.
6. **Terminology/compatibility drift** — prefer Collector wording in docs while preserving compatibility-sensitive `collector` names.

---

# Suggested First Milestone

If you want to de-risk this work, stop after Tasks 1–5 first. That gets you:
- package skeleton
- stable wrapper
- centralized config
- extracted state management
- extracted pure normalization/hash helpers

That would deliver a meaningful modularity win with relatively low operational risk.
