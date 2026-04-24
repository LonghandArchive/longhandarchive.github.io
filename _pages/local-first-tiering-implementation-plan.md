---
title: "Local-First Tiering Implementation Plan"
---

# CSJ local-first archival tiering implementation plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Introduce archive-tier metadata, freeze eligibility rules, and future-offload-compatible conventions for the CSJ Collector while keeping all storage on local disk.

**Architecture:** Keep current local-disk behavior working. Add explicit tier/freeze/storage metadata first, then add run manifests and maintenance tooling. Do not implement remote storage now. Do not force disruptive path migrations up front.

**Tech Stack:** Python 3, existing CSJ modules under `csj/`, pytest, JSON manifests/history files, local disk only.

---

## Non-negotiable constraints

1. All storage remains local disk for now.
2. Do not add R2/object-storage code yet.
3. Do not break existing collector paths or current cron jobs.
4. Do not require immediate directory migration of the workspace.
5. Add semantics first, path churn later.
6. Preserve current collector behavior unless a task explicitly changes it.

---

## Desired outcomes

After this plan:
- the archive can classify artifacts by tier even while everything remains local
- records/assets/manifests know whether they are hot, warm, or cold candidates
- freeze eligibility is explicit rather than implicit
- run manifests exist as immutable archival evidence
- future offload to R2 can be implemented by moving a tier, not redesigning the archive model

---

## New concepts to add

### Tier concepts
- `hot`
- `warm`
- `cold`

### Freeze concepts
- `mutable`
- `freeze_candidate`
- `frozen`

### Storage concepts
- `storage_backend`: `local`
- `storage_locator`: local path or manifest-relative locator

These are semantic fields first, not infrastructure abstractions.

---

## Files likely to change

### Code
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/config.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/records.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/assets.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/history.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/run.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/lifecycle.py`
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/csj/tiering.py`

### Tests
- Create: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_tiering.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_records.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_assets.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py`
- Modify: `/root/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py`

### Docs
- Modify: `README.md`
- Modify: `ROADMAP.md`
- Modify: `docs/workspace-outputs.md`
- Modify: `docs/storage-tiering-and-freeze-policy.md`
- Modify: `docs/current-status.md`

---

## Phase 1 — define tier/freeze semantics without moving files

### Task 1: Add a dedicated tiering module

**Objective:** Centralize tier/freeze decisions in one place.

**Files:**
- Create: `csj/tiering.py`
- Test: `tests/test_tiering.py`

**Step 1: Write failing tests**
Add tests for helpers like:
- `classify_job_archive_state(job_record, now_dt)`
- `classify_asset_archive_state(asset_record, job_record, now_dt)`

Cover at minimum:
- active job => hot + mutable
- missing_unconfirmed => hot + mutable
- closed 10 days ago => not yet frozen
- closed 35 days ago => freeze candidate
- closed 95 days ago => frozen/cold candidate
- withdrawn_confirmed 40 days ago => freeze candidate

**Step 2: Run test to verify failure**
Run:
```bash
uvx pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_tiering.py
```
Expected: FAIL

**Step 3: Implement minimal code**
Create a small tiering module that:
- defines thresholds in one place
- returns semantic state only
- does not perform file moves

**Step 4: Run tests to verify pass**
Run the same command.
Expected: PASS

---

### Task 2: Add tier/freeze metadata to current job records

**Objective:** Make current job projections explicitly aware of archive state.

**Files:**
- Modify: `csj/records.py`
- Modify: `csj/tiering.py`
- Test: `tests/test_records.py`

**Step 1: Write failing tests**
Add assertions that normalized/finalized jobs include fields like:
- `archive_tier`
- `freeze_status`
- `freeze_candidate_at`
- `storage_backend`
- `storage_locator`

Initial expected values for active records should remain local/hot/mutable.

**Step 2: Run targeted tests**
```bash
uvx pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_records.py -k archive_tier
```
Expected: FAIL

**Step 3: Implement minimal code**
In `normalize_job()` / `finalize_job()`:
- keep current semantics intact
- inject tier metadata from `csj.tiering`
- set `storage_backend = "local"`
- set `storage_locator` to the current local path or a stable logical local locator

**Step 4: Run tests to verify pass**
Run the targeted test, then the full records test file.

---

### Task 3: Add tier/freeze metadata to asset manifests

**Objective:** Make assets first-class tier candidates, especially pooled attachments.

**Files:**
- Modify: `csj/assets.py`
- Modify: `csj/tiering.py`
- Test: `tests/test_assets.py`

**Step 1: Write failing tests**
Add tests ensuring manifest entries can carry:
- `archive_tier`
- `freeze_status`
- `storage_backend`
- `storage_locator`
- `eligible_for_offload_at`

**Step 2: Run targeted test**
```bash
uvx pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_assets.py -k tier
```
Expected: FAIL

**Step 3: Implement minimal code**
For manifest entries:
- keep path behavior unchanged
- add local-storage metadata
- classify pooled attachments as prime cold-tier candidates once the parent job is eligible

**Step 4: Run tests**
Run targeted then broader assets tests.

---

## Phase 2 — add immutable run manifests for point-in-time reconstruction

### Task 4: Add a per-run corpus manifest path and schema

**Objective:** Preserve exact run membership as archival evidence.

**Files:**
- Modify: `csj/config.py`
- Modify: `csj/state.py`
- Modify: `tests/test_state_and_paths.py`

**Step 1: Write failing test**
Add config/path assertions for something like:
- `RUN_MANIFESTS_DIR = DATA_DIR / "csj_run_manifests"`

**Step 2: Run test**
```bash
uvx pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_state_and_paths.py -k manifest
```
Expected: FAIL

**Step 3: Implement minimal code**
Add path constant and ensure dir creation.

**Step 4: Run test**
Expected: PASS

---

### Task 5: Write immutable run manifests on normal full runs

**Objective:** Capture exact visible corpus per qualifying run.

**Files:**
- Modify: `csj/run.py`
- Test: `tests/test_run_module.py`

**Step 1: Write failing test**
Add test ensuring a qualifying run writes a run manifest containing at least:
- `run_id`
- `started_at`
- `completed_at`
- run scope flags (`full`, `limit`, `grade`, `department`, `refresh`, `repair_lifecycle`)
- exact list of references seen in search
- optionally listing-card metadata per seen reference

**Step 2: Run targeted test**
```bash
uvx pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests/test_run_module.py -k manifest
```
Expected: FAIL

**Step 3: Implement minimal code**
Write manifests for:
- full unfiltered runs at minimum
- optionally all runs, but clearly mark scope so later reconstruction knows whether it was a full census

Important:
- manifest should be immutable evidence
- do not overwrite prior manifests for the same run

**Step 4: Run tests**
Targeted then broader run-module tests.

---

### Task 6: Link saved records to producing runs

**Objective:** Improve provenance from snapshot to run.

**Files:**
- Modify: `csj/records.py`
- Modify: `csj/history.py`
- Modify: `csj/run.py`
- Test: `tests/test_records.py`, `tests/test_run_module.py`

**Step 1: Write failing tests**
Assert that current snapshots and/or history snapshots include:
- `captured_by_run_id`
- maybe `captured_from_manifest_id` if useful

**Step 2: Run targeted tests**
Expected: FAIL

**Step 3: Implement minimal code**
Pass `run_id` through save/history calls so that:
- current record knows latest producing run
- history snapshots know the producing run

**Step 4: Run tests**
Expected: PASS

---

## Phase 3 — make freeze status visible but keep all bytes local

### Task 7: Add a maintenance report for freeze candidates

**Objective:** Let operators see what would move tiers later without moving anything now.

**Files:**
- Modify: `csj/cli.py` or `csj/run.py`
- Create test in `tests/test_tiering.py` or `tests/test_cli_module.py`

**Step 1: Write failing test**
Add test for a helper/report function that summarizes counts like:
- hot mutable jobs
- freeze-candidate jobs
- frozen jobs
- pooled assets tied to frozen jobs

**Step 2: Run targeted test**
Expected: FAIL

**Step 3: Implement minimal code**
This can be:
- a helper function only, or
- a CLI/report mode if useful

Do not move files.
Only report.

**Step 4: Run tests**
Expected: PASS

---

### Task 8: Optionally add local tier directories without migration

**Objective:** Prepare future physical layout without changing current behavior.

**Files:**
- Modify: `csj/config.py`
- Modify: `csj/state.py`
- Test: `tests/test_state_and_paths.py`

**Step 1: Write failing test**
Only if you decide to create dormant directories such as:
- `DATA_DIR / "tiers" / "warm"`
- `DATA_DIR / "tiers" / "cold"`

**Step 2: Run test**
Expected: FAIL

**Step 3: Implement minimal code**
Create directories only.
Do not migrate files yet.

**Step 4: Run tests**
Expected: PASS

This step is optional because metadata-first tiering is the real priority.

---

## Phase 4 — documentation and operator clarity

### Task 9: Update docs to reflect the local-first tier model

**Objective:** Make the archive model clear to future maintainers.

**Files:**
- Modify: `README.md`
- Modify: `ROADMAP.md`
- Modify: `docs/workspace-outputs.md`
- Modify: `docs/storage-tiering-and-freeze-policy.md`
- Modify: `docs/current-status.md`

**Step 1: Update README doc map**
Include the local-first tiering architecture and this implementation plan if you keep it as a canonical plan artifact.

**Step 2: Update workspace outputs doc**
Explain that tier state is semantic first, physical later.

**Step 3: Update roadmap**
Make clear that:
- local-disk tiering semantics are the current objective
- remote offload is deferred until later

**Step 4: Verification**
Read docs and make sure they agree with actual code behavior.

---

## Verification stage

Run at minimum:
```bash
uvx pytest -q /root/.hermes/skills/research/civil-service-jobs-collector/tests
python3 /root/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --help
python3 /root/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --repair-lifecycle --dry-run
python3 /root/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py -n 1
```

Expected:
- tests pass
- help still works
- repair dry-run still works
- normal limited run still works
- current records/manifests carry local tier metadata
- run manifest created on qualifying run(s)
- no file movement to remote/off-disk storage occurs

---

## Final outcome

The system should end this phase with:
- all data still on local disk
- explicit tier/freeze semantics
- immutable run manifests for stronger point-in-time reconstruction
- local storage metadata that makes future R2/blob offload a storage-layer migration rather than a redesign
