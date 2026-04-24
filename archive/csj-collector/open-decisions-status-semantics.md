---
title: "Open Decisions: Status Semantics"
permalink: /archive/csj-collector/open-decisions-status-semantics/
---

> **Superseded by** [ADR 0002 — Date-first closure policy](../../../docs/decisions/0002-date-first-closure-policy.md), [ADR 0004 — Two-layer lifecycle model](../../../docs/decisions/0004-two-layer-lifecycle-model.md), and [ADR 0005 — Evidence-based lifecycle transitions](../../../docs/decisions/0005-evidence-based-lifecycle-transitions.md).
> Retained as historical record of the open decision state on 2026-04-20.

---

# Open decisions — status semantics and archive validation

Last reviewed: 2026-04-20

This note captures the remaining collector-policy decisions that were surfaced by the lifecycle/status review and validator work.

## 1. Date-first vs search-first closure

### Question
When a role's declared close date has passed but the role still appears in current search results, should the collector treat it as:
- `active` because it is still visible in search, or
- `closed` because the declared source closing field has passed?

### Why this matters
Current validator result:
- `past_due_not_closed = 28`

This is now the clearest archive-wide semantics mismatch still visible in the current data.

### Competing interpretations

#### Search-first
- Pros:
  - matches current search visibility exactly
  - conservative about not closing records while the source still surfaces them
- Cons:
  - `active` can drift away from the declared acceptance window encoded in `closes` / `closes_iso`
  - past-due roles can remain `active` even when archival interpretation would expect closure

#### Date-first
- Pros:
  - keeps collector semantics aligned with declared source field values
  - matches the archival principle that the collector should preserve source timing rather than infer recruitment outcomes
- Cons:
  - may mark roles `closed` while the site still visibly lists them
  - requires a deliberate rule for how to interpret lingering search visibility after deadline

## 2. Scope of validator enforcement

### Question
Which findings should be treated as:
- hard archive/data-integrity errors,
- policy mismatches,
- or advisory evidence gaps?

### Current validator scope
Current checks cover:
- invalid status/lifecycle enums
- invalid `closes_iso`
- `past_due_not_closed`
- `future_dated_closed`
- status/lifecycle incompatibilities
- missing/withdrawn records lacking expected metadata
- withdrawn records without strong evidence

### Candidate extensions
- manifest/projection disagreement for references seen in the latest qualifying full run
- event-log support checks for lifecycle transitions
- run-summary vs archive-state consistency checks
- optional CSV summary output for quick review

### Decision needed
Which anomaly classes should block trust in the archive vs simply flag review work?

## 3. Collector vs insights boundary for lifecycle conclusions

### Question
How far should the collector go in interpreting lifecycle outcomes?

### Current agreed boundary
Collector layer should focus on:
- source field capture
- declared closing fields
- disappearance / reappearance tracking
- strong evidence of non-availability

Insights layer should own:
- cancellation inference
- hire/interview progression inference
- same-role/new-ref continuity or repost analysis
- campaign-level conclusions

### Remaining decision
Whether to keep current lifecycle labels exactly as they are, or introduce a more explicit archive-facing state for ambiguous cases such as "past due but still visible in search".

## Recommended next order
1. Decide date-first vs search-first closure semantics.
2. After that, promote the chosen rule into validator severity and runtime policy.
3. Keep cross-ref replacement/cancellation interpretations in downstream insights, not collector runtime behavior.
