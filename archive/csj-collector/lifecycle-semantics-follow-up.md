---
title: "Lifecycle Semantics Follow-up Plan"
permalink: /archive/csj-collector/lifecycle-semantics-follow-up/
---

> **Superseded by** [ADR 0004 — Two-layer lifecycle model](../../../docs/decisions/0004-two-layer-lifecycle-model.md) and [ADR 0005 — Evidence-based lifecycle transitions](../../../docs/decisions/0005-evidence-based-lifecycle-transitions.md).
> Retained as historical record of the implementation plan and TDD approach used to deliver the lifecycle corrections.

---

# CSJ lifecycle semantics follow-up plan

> **For Hermes:** implement this with strict TDD. Add the failing tests first, watch them fail, then make the smallest code changes needed.

**Goal:** tighten three semantics problems surfaced by live pressure-testing: status-only misclassification of withdrawn roles, manifest/projection drift when closed roles reappear in search, and over-strong promotion on `expired_sid_redirect`.

**Architecture:** keep the existing two-layer `status` / `lifecycle_status` model for now, but make downstream code prefer `lifecycle_status` where the distinction matters. Preserve run manifests as immutable corpus-membership evidence, and make the mutable current projection track observed reappearance in search. Treat `expired_sid_redirect` as medium-confidence evidence, not hard withdrawal confirmation.

**Files likely touched:**
- `tests/test_tiering.py`
- `tests/test_run_module.py`
- `tests/test_lifecycle.py`
- `tests/test_repair_mode.py`
- `csj/tiering.py`
- `csj/run.py`
- `csj/lifecycle.py`
- `docs/current-status.md`
- `docs/maintainer-guide.md`
- `SKILL.md`

---

## Task 1: Lock in the desired semantics with tests

1. Add a tiering test proving `lifecycle_status='withdrawn_confirmed'` must outrank broad `status='active'`.
2. Add a refresh-queue test proving stale `withdrawn_confirmed` records must not be selected by `collect_refresh_refs()`.
3. Replace the old "do not reactivate closed records" expectation with a test proving that a currently closed projection is reopened when the reference is seen in search again.
4. Add a lifecycle test proving `expired_sid_redirect` remains `missing_unconfirmed` even after the repeated-missing threshold.
5. Add a repair-mode test proving medium-confidence `expired_sid_redirect` evidence is not enough to promote to `withdrawn_confirmed`.

Verification:
- run each new/changed test directly and confirm it fails for the expected semantic reason before changing production code.

## Task 2: Make the minimal runtime changes

1. In `csj/tiering.py`, classify `closed` / `withdrawn_confirmed` before broad `status='active'` handling.
2. In `csj/run.py`, make `collect_refresh_refs()` key off resolved lifecycle state rather than broad `status`.
3. In `csj/run.py`, make `refresh_existing_listing_state()` reopen any non-active record seen in search, including `closed`.
4. In `csj/lifecycle.py`, stop promoting `expired_sid_redirect` to `withdrawn_confirmed`; keep it as `missing_unconfirmed` with medium/low confidence.
5. In `csj/lifecycle.py`, harden `evaluate_repair_action()` so only strong verification (`http_404`, `withdrawn_text`, or other future high-confidence evidence) can produce `withdrawn_confirmed`.

## Task 3: Verify and document

1. Run targeted pytest commands for the changed modules.
2. Run the full CSJ test suite.
3. Run `scripts/collector.py --help`.
4. Run `scripts/collector.py --repair-lifecycle --dry-run`.
5. Update maintainer-facing docs to capture:
   - downstream code must prefer `lifecycle_status` when distinguishing live vs missing/withdrawn
   - manifests are stronger point-in-time evidence than the mutable current projection
   - `expired_sid_redirect` is medium-confidence evidence only

## Expected outcome

After this follow-up:
- withdrawn-confirmed records no longer poison active counts, refresh selection, or tiering
- search reappearance can correctly reactivate previously closed current projections
- `expired_sid_redirect` stays a warning/evidence signal, not hard withdrawal confirmation
- docs are explicit about what manifests prove and what current projections do not
