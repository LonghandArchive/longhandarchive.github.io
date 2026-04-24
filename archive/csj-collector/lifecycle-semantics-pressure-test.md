---
title: "Lifecycle Semantics Pressure Test"
permalink: /archive/csj-collector/lifecycle-semantics-pressure-test/
---

> **Superseded by** [ADR 0004 — Two-layer lifecycle model](../../../docs/decisions/0004-two-layer-lifecycle-model.md) and [ADR 0005 — Evidence-based lifecycle transitions](../../../docs/decisions/0005-evidence-based-lifecycle-transitions.md).
> Retained as historical record of the live pressure-testing session and the empirical findings that led to those decisions.

---

# Lifecycle semantics pressure-test note

Last reviewed: 2026-04-20

This note captures the follow-up from live pressure-testing of full-run behavior and current archive semantics.
It is a maintainer note, not a historical handoff.

## Why this review happened

Three semantics issues showed up when comparing qualifying full-run manifests, current `csj_jobs/` projections, and live CSJ behavior:

1. `withdrawn_confirmed` records kept broad `status="active"`, so status-only downstream logic treated them as live.
2. Run manifests and the mutable current projection could disagree when a previously closed role reappeared in search.
3. `expired_sid_redirect` was being treated as strong enough to promote a role to `withdrawn_confirmed`, even though live pressure-testing showed it was only medium-confidence evidence.

## What was pressure-tested

### 1. Status vs lifecycle semantics

Observed before the fix:
- status-only active counts overcounted real live roles
- refresh selection kept re-queueing stale `withdrawn_confirmed` records
- tiering left `withdrawn_confirmed` records `hot` / `mutable`

The important architectural conclusion was:
- broad `status` is not specific enough to distinguish true live roles from `missing_unconfirmed` / `withdrawn_confirmed`
- any downstream logic that needs that distinction must prefer `lifecycle_status`

### 2. Manifest vs projection semantics

Observed before the fix:
- qualifying full-run manifests correctly recorded references seen in that run
- the current mutable `csj_jobs/{reference}.json` projection could still remain `closed` if a previously closed role reappeared in search and was skipped/deduped rather than refetched

That means:
- run manifests are stronger point-in-time corpus-membership evidence
- the current projection is mutable and not guaranteed to mirror an older run without lifecycle/reopen logic keeping up

### 3. Evidence quality of `expired_sid_redirect`

Live pressure-testing showed:
- the 21 questioned roles were not present in current live search results
- their stored detail URLs no longer yielded a usable vacancy page
- but `is_probable_csj_homepage()` is broad enough to match live detail pages too; the reference-text check is what prevents many false positives

Conclusion:
- `expired_sid_redirect` is useful evidence that the stored URL is no longer resolving to a vacancy page
- it is not hard proof on the level of `http_404` or explicit "no longer available" copy
- it should be treated as medium-confidence evidence, not confirmation

## Code changes made in this follow-up

### Refresh selection now prefers lifecycle state

Changed in `csj/run.py`:
- `collect_refresh_refs()` now resolves lifecycle state and only refreshes records whose effective lifecycle state is `active`
- stale `withdrawn_confirmed` and `missing_unconfirmed` records are no longer re-queued just because broad `status` still says `active`

### Reappearance in search now reopens current projections, including closed ones

Changed in `csj/run.py`:
- `refresh_existing_listing_state()` now reactivates any non-active current projection when the reference is seen in search again, including previously `closed` records
- reopened history writes now include `status` as a changed field when broad status changed too

This keeps the mutable current projection aligned with current search presence more reliably.

### Tiering now prioritizes lifecycle state over broad active status

Changed in `csj/tiering.py`:
- `withdrawn_confirmed` is now classified using the closed/withdrawn branch before the broad `status="active"` branch can short-circuit it

This prevents withdrawn records from remaining indefinitely `hot` / `mutable` purely because of the broad status field.

### `expired_sid_redirect` no longer promotes to `withdrawn_confirmed`

Changed in `csj/lifecycle.py`:
- `verify_missing_job_url()` now keeps `expired_sid_redirect` as `missing_unconfirmed` (with low/medium confidence depending on repetition), not `withdrawn_confirmed`
- `evaluate_repair_action()` now only accepts strong confirmation (`http_404`, `withdrawn_text`, or future high-confidence evidence) for promotion to `withdrawn_confirmed`

## Tests added/updated

The following regression expectations were locked in:
- withdrawn lifecycle state outranks broad active status for tiering
- refresh selection ignores stale `withdrawn_confirmed` records
- closed records seen in search again are reopened
- `expired_sid_redirect` remains `missing_unconfirmed` even after repeated missing runs
- repair mode does not promote medium-confidence `expired_sid_redirect` evidence to `withdrawn_confirmed`

## Remaining caveats

These are still true after the fix:

1. The project still uses a two-layer model:
   - `status` = broad state
   - `lifecycle_status` = specific state

   That can work, but maintainers must remember that `status="active"` does **not** necessarily mean "currently live and healthy" unless `lifecycle_status` is also `active`.

2. Run manifests remain stronger evidence than the mutable current projection for point-in-time questions.

3. `expired_sid_redirect` is still operationally useful and may justify wording like "likely disappeared" in monitoring/reporting, but it should not be treated as archival confirmation.

## Practical maintainer rule

Use this interpretation order when reasoning about a role:

1. **Manifest / reconstruction** for point-in-time run membership questions
2. **`lifecycle_status`** for specific lifecycle meaning
3. **`status`** only as a broad bucket, not a source of fine-grained truth

If those layers disagree, trust the more specific / more evidential one, not the broad convenience field.
