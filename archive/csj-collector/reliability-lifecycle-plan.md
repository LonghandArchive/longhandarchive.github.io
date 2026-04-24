---
title: "CSJ Reliability Lifecycle Plan"
permalink: /archive/csj-collector/reliability-lifecycle-plan/
---

# CSJ Collector Reliability & Lifecycle Updates Plan

> **Execution note:** This plan is being updated live during implementation if design or scope changes are discovered.

**Goal:** Make the collector classify missing/refresh failures more accurately, reduce false operational-failure signals, and close lifecycle gaps exposed by recent runs.

**Architecture:** Keep the existing single-file collector architecture, but tighten the decision points around refresh, missing-job verification, and end-of-run reporting. Add a small pytest test layer around lifecycle/verification helpers before changing behaviour.

**Tech Stack:** Python 3, collector at `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`, pytest for new tests.

---

## Learnings driving this work

1. Refresh failures for jobs absent from search are often expected because stored detail URLs contain expiring SID tokens.
2. Jobs genuinely disappear and reappear, so the nuanced missing/reopened lifecycle needs to stay.
3. Missing jobs with no closing date can get stuck in `missing_unconfirmed` limbo.
4. Past-due active jobs can remain wrongly active due to lifecycle blind spots.
5. Run reporting still lacks enough operational nuance for unattended cron summaries.

---

## Planned tasks

### Task 1: Add a minimal pytest harness
- Create tests for lifecycle and failure classification helpers.
- Mock HTTP responses instead of hitting live CSJ.
- Cover 404, unavailable text, homepage/meta-refresh HTML, detail page still containing the reference, and active job with past `closes_iso`.

### Task 2: Introduce explicit detail-fetch failure classification
- Distinguish `likely_removed_or_withdrawn`, `transient`, and `unknown`.
- Preserve failure metadata instead of dropping failures into a generic list.

### Task 3: Detect expired-SID homepage redirects explicitly
- Identify CSJ homepage/meta-refresh patterns from expired stored URLs.
- Use that as evidence for likely removed/withdrawn jobs, especially for refresh targets absent from search.

### Task 4: Tighten promotion from `missing_unconfirmed` to `withdrawn_confirmed`
- Keep first-pass missing behaviour.
- After repeated missing full runs, use direct verification evidence to graduate jobs out of limbo.
- Improve handling for jobs with no closing date.

### Task 5: Add retroactive cleanup for stale active jobs whose closing date has passed
- During full lifecycle pass, close any active job absent from search whose `closes_iso` is now past.
- Ensure this fixes older records that predate the newer lifecycle pipeline.

### Task 6: Refactor end-of-run reporting into explicit counters
- Track new/updated fetches, refresh fetches, successes, failures, likely removed/withdrawn, transient failures, and unknown failures.
- Add anomaly thresholds (>20% warning, >50% prominent warning).
- Include lifecycle counters in the summary payload.

### Task 7: Improve `csj_latest.json` semantics
- Add explicit counts like `search_result_count`, `fetched_job_count`, `skipped_existing_count`, and `refresh_target_count` so dedup-heavy runs are not misleading.

### Task 8: Update docs/skill guidance
- Align the skill and edge-case reference docs with the implemented behaviour.
- Recreate/update cron jobs afterwards if prompt guidance changes materially.

### Task 9: Consider a one-off lifecycle repair mode
- If implementation reveals it is necessary or cheap, add a dedicated repair mode/script.
- Otherwise leave as a future follow-up.

---

## Initial execution order

### Phase 1 — High value, low risk
1. Add tests
2. Add failure classification
3. Add expired-SID detection
4. Add richer reporting

### Phase 2 — Lifecycle correctness
5. Improve missing → withdrawn promotion
6. Fix stale-active close-date cleanup

### Phase 3 — Usability
7. Improve `csj_latest.json`
8. Update docs
9. Decide whether repair mode is warranted

---

## Live implementation notes

- Confirmed there is **no git repo** in the skill directory, so implementation is being tracked via the saved workspace plan and direct file diffs rather than commits.
- Found an additional root-cause issue beyond the original plan: **refresh fallback listings injected from stored URLs were polluting the lifecycle pass's `search_refs` set**, making some absent jobs look present. This is now part of the implementation scope.
- The one-off lifecycle repair mode was initially deferred, but has now been implemented after validating the new reporting path.
- Implemented in this pass:
  - homepage/expired-SID detection in `verify_missing_job_url()`
  - structured fetch-failure classification (`likely_removed_or_withdrawn` / `transient` / `unknown`)
  - lifecycle search-ref computation that excludes refresh-only fallback entries
  - richer `csj_latest.json` reporting including structured failures and lifecycle counters
  - pytest coverage for the new helper behaviour
  - one-off `--repair-lifecycle` mode with `--dry-run`
- Verification completed:
  - `pytest -q ~/.hermes/skills/research/civil-service-jobs-collector/tests` → **13 passed**
  - `python3 -m py_compile ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py` → **passed**
  - `python3 .../collector.py --repair-lifecycle --dry-run` → **26 closures, 0 errors** after bugfix
- Real-world outcomes:
  - manual full refresh classified **64/64** failures as `likely_removed_or_withdrawn` with `expired_sid_redirect`
  - inspection of the current **22 missing_unconfirmed** jobs found **no strong promotion candidates yet**; all still verify only as low-confidence `expired_sid_redirect`
  - real repair run closed **26** stale active jobs whose closing dates had already passed
