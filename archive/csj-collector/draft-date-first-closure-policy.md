---
title: "Draft Date-First Closure Policy"
permalink: /archive/csj-collector/draft-date-first-closure-policy/
---

> **Superseded by** [ADR 0002 — Date-first closure policy](../../../docs/decisions/0002-date-first-closure-policy.md).
> Retained as historical record of the draft proposal and deliberation.

---

# Draft policy — date-first closure semantics

Status: draft only — not yet implemented in collector runtime
Last reviewed: 2026-04-20

## Purpose

This note proposes an exact collector-layer rule for handling roles whose declared close date has passed.
It is deliberately limited to collector/archive semantics and does not attempt to infer recruitment outcomes.

## Collector principle

The collector layer should prioritize:
1. preserving source field values accurately
2. recording source visibility/disappearance over time
3. avoiding downstream conclusions such as cancellation, successful hiring, or repost continuity

Under that principle, the declared source closing field (`closes` / normalized `closes_iso`) is the best collector-layer proxy for when the role should be treated as closed in the archive.

## Proposed rule

### Broad rule
If:
- `closes_iso` is present, and
- `closes_iso < now`

then the role's broad archive `status` should be `closed`.

This remains true even if the role still appears in search results.

## Proposed lifecycle behavior

### Case 1 — before close date and visible in search
- `status = active`
- `lifecycle_status = active`

### Case 2 — before close date and disappears from search
- `status = active`
- `lifecycle_status = missing_unconfirmed`

### Case 3 — before close date and disappears with strong evidence of withdrawal
- `status = active`
- `lifecycle_status = withdrawn_confirmed`

### Case 4 — close date has passed
- `status = closed`
- `lifecycle_status = closed`

This applies regardless of whether the role:
- still appears in search results
- previously went through `missing_unconfirmed`
- previously went through `withdrawn_confirmed`

## Why this rule is attractive

### Pros
- aligns archive semantics with declared source field values
- avoids overloading search visibility as the definition of "active"
- matches the project's stated goal of preserving captured source data rather than inferring outcomes
- gives a clear archive-wide validation rule

### Cons / trade-offs
- a role may remain visible in search after its declared close date and still be marked `closed`
- some users may read `closed` as "definitively no longer publicly visible", which this rule does not guarantee
- this may slightly reduce fidelity to current search visibility in the mutable projection

## Important non-goals

This policy does **not** claim that a past-due role:
- was cancelled
- completed hiring successfully
- stopped accepting applications in every practical sense
- was or was not reposted under another reference

Those are insights-layer questions.

## Suggested implementation shape

If adopted, implement conservatively:
1. update lifecycle pass so past-due records become `closed` even if present in current search
2. add targeted tests for "past-due but still visible" roles
3. extend validator severity so `past_due_not_closed` becomes a first-class policy mismatch
4. document that `status=closed` means "declared close date passed" rather than "guaranteed absent from search"

## Suggested wording for maintainers

Use this phrasing:

> In the collector layer, `closed` means the role's declared source close date has passed. Search visibility after that point is preserved as evidence of source behavior, but does not keep the role in broad `active` status.
