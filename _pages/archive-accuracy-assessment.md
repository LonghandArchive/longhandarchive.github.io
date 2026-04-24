---
title: "Archive Accuracy Assessment"
---

# CSJ Collector archive-accuracy assessment

Last reviewed: 2026-04-20

This note assesses whether the CSJ Collector currently behaves like a historically reliable archive of Civil Service Jobs data.

The benchmark here is not “good scraper output”.
It is closer to archival questions such as:
- can we reconstruct what was known at a given time?
- can we explain where that knowledge came from?
- can we detect meaningful change over time?
- can we preserve supporting material with stable identity?
- can we trust the record after crashes, retries, and later code changes?

## Bottom-line judgment

Current state:
- good historical collector
- not yet full archival-record system in the Wayback Machine / National Archives sense

The collector is already strong on:
- per-record normalization
- change detection
- lifecycle state tracking
- asset capture/versioning
- run telemetry
- current-snapshot plus history layering

But it is still weaker than an archive-grade record system on:
- exact point-in-time reconstruction of the full visible corpus
- source provenance at the raw-capture level
- tamper-evident / audit-grade fixity
- full run-scope documentation
- uniform historical quality across older data vintages

So the right conclusion is:
- the system is already capable of preserving historically meaningful job records
- but it does not yet provide fully archival-grade guarantees for exact reconstruction of “what the site showed at time T”

## Assessment framework

This review uses six archival dimensions:
1. record authenticity
2. temporal reconstruction
3. provenance and context
4. integrity/fixity
5. completeness
6. reproducibility and auditability

## 1. Record authenticity

### What is good now

The collector preserves strong job-level structured records:
- `first_seen`
- `last_seen`
- `status`
- `lifecycle_status`
- `last_changed_at`
- `content_hash`
- `field_hashes`
- `asset_versions`
- `archive_completeness`

`csj/records.py` and `csj/hashing.py` do real work to ensure the saved record is the thing being hashed and diffed.
That is a strong authenticity property for the normalized record.

Important strengths:
- final hash is computed after enrichment
- meaningful changes create history snapshots
- lifecycle transitions also write history/events
- asset manifests produce exact asset-version pointers for job snapshots

### What is weaker than archival-grade authenticity

The collector usually preserves normalized structured output and `full_text`, but it does not preserve the full raw HTTP response envelope, for example:
- raw HTML body snapshot for every fetch
- response status code on successful captures
- response headers
- fetch URL after redirects
- capture method metadata at per-record level

That means the archive is strong as a normalized historical data archive, but weaker as a forensic source capture.

Wayback-style archival systems preserve the original page representation more directly.
This collector preserves a faithful extracted record, not a full browser-accurate page capture.

## 2. Temporal reconstruction

This is the most important gap for your stated goal.

Your goal is to be able to say:
- what jobs were available at a particular point in time?

### What is good now

The collector records intervals well enough to infer broad availability windows:
- `first_seen`
- `last_seen`
- closure/missing/reopened transitions
- version snapshots when meaningful changes occur

For an individual job, this already gives a credible temporal narrative.

### The critical limitation

The collector does not currently preserve a canonical full-search-result manifest per run.

It preserves:
- current per-job state
- per-job history snapshots
- lifecycle events
- run summaries
- immutable run manifests for qualifying unfiltered runs

That means “what jobs were available at time T?” can now be answered directly for runs that produced a qualifying manifest, while still falling back to interval-based reasoning for periods without such a manifest.

This matters because:
- `last_seen` is an interval marker, not a full site snapshot
- limited or filtered runs can update some records without representing the full corpus
- a job available between two runs can only be bracketed, not proven at sub-run precision

### Important code-level reason

`refresh_existing_listing_state()` updates `last_seen` for skipped listings during normal runs, including runs that may be limited or filtered.
That is useful operationally, but it means `last_seen` is not by itself a guaranteed “seen in a full archive census” field.

### Archival conclusion

Current status:
- good interval-based historical reconstruction
- not yet exact point-in-time corpus reconstruction

To reach archive-grade point-in-time reconstruction, the system should preserve for every full unfiltered run:
- run_id
- run_started_at / run_completed_at
- run scope metadata
- exact set of references seen in search
- ideally listing-card metadata as seen in that run

## 3. Provenance and context

### What is good now

There is meaningful provenance in the system:
- `scraped_at`
- `parser_version`
- `schema_version`
- `run_id` in run summaries
- run events in `csj_run_events.jsonl`
- job events in `csj_events.jsonl`
- asset events and asset history

This is much better than a flat scraper dump.

### What is missing for stronger provenance

Per-job provenance is still thinner than ideal.
Useful missing provenance fields would include:
- `capture_method` / collector mode recorded per history snapshot
- capture URL after redirect normalization
- HTTP status and response metadata for successful captures
- explicit source-evidence classification for lifecycle changes

Important current nuance:
- `captured_by_run_id` now exists on current records and history snapshots when provenance is known
- but a current record with matching `captured_by_run_id` is still weaker than a history snapshot because current records remain mutable projections

At the moment, provenance exists, but it is distributed and not perfectly stitched together at the individual snapshot level.

## 4. Integrity and fixity

### What is good now

The system has meaningful internal fixity:
- `content_hash`
- `field_hashes`
- asset `content_hash`
- markdown/transcript hashes
- content-addressed pooled asset storage

Also good:
- current snapshots and run summaries use atomic JSON writes

### Important limitations

The archive is not yet tamper-evident in the stronger archival sense.
For example, there is no:
- hash chaining across events
- signed manifests
- external fixity log
- immutable write-once storage strategy

Also, not every persistence path appears equally crash-safe.
Current snapshots use `atomic_write_json()`, but some history/event writes still use direct append/write patterns.
That is usually fine for a collector, but weaker than archival-grade write discipline.

Examples:
- `history.py` writes history snapshots with direct `write_text()`
- `append_event()` appends JSONL without fsync/hash chaining
- asset history/manifests also use ordinary writes/appends in places

### Archival conclusion

The system has good internal consistency checking.
It does not yet have strong audit/fixity guarantees against corruption or silent tampering.

## 5. Completeness

### What is good now

The collector is unusually strong on auxiliary completeness:
- attachment download at capture time
- transcript capture
- asset manifests/history/events
- `archive_completeness` field
- distinction between missing assets and failed transcripts

This is exactly the kind of thing that improves historical usefulness.

### Important completeness gaps

1. Raw page representation is not fully preserved.
2. Search-result pages themselves are not archived as first-class artifacts.
3. Point-in-time corpus membership is preserved for qualifying unfiltered runs, but not for every historical period or every filtered/maintenance run.
4. Historical quality is not uniform across all vintages.

The last point matters.
Older records may reflect pre-fix behavior.
For example, historical attachment identity was once too coarse, and older records can still show generic `csj-download` normalization behavior.
That means the archive is not equally precise across all earlier periods.

## 6. Reproducibility and auditability

### What is good now

The collector is testable and structured:
- parser fixtures
- golden-output tests
- modular code
- telemetry and run summaries

That is very good engineering support for a historical system.

### What is still weaker than ideal

If someone asked:
“Show me exactly why this record looked like this on that day,”
you could often explain it, but not always with a complete archival evidence chain.

Missing pieces for stronger auditability include:
- immutable run manifests of all refs seen
- per-history-snapshot link to the run that created it
- preserved raw HTML evidence for listing/detail capture
- explicit backfill/repair provenance on records changed by repair workflows

## Specific strengths to keep

These are already very valuable and should be preserved:
- `content_hash` / `field_hashes`
- per-record history snapshots
- event logs for lifecycle changes
- asset version references pinned into job snapshots
- `archive_completeness`
- parser fixtures and golden-output tests
- atomic writes for current snapshots and summaries

## Most important risks

### Risk 1: run-manifest coverage is still incomplete
This remains the biggest gap against your stated goal.
Qualifying unfiltered runs now preserve immutable corpus manifests, but periods without such a run are still interval-based rather than exact.

### Risk 2: limited/filtered runs can affect observational fields
Fields like `last_seen` are operationally useful, but not pure “full census” evidence unless run scope is recorded alongside them.

### Risk 3: no raw HTML evidence layer
You preserve structured output and `full_text`, but not the exact page representation needed for stronger source-level authenticity.

### Risk 4: older historical data is not uniform
Some older archive periods may reflect earlier attachment/lifecycle logic before later fixes.
That does not make the archive useless, but it does mean historical confidence is versioned over time.

### Risk 5: history/event persistence is not fully atomic/audit-grade
This is more about archival robustness than logical correctness.

## Recommended next improvements

If the aim is to move from “good historical collector” toward “archive-grade historical record”, these are the highest-value next steps.

### Priority A — Preserve per-run corpus manifests
For every full unfiltered run, store an immutable run manifest containing at minimum:
- run_id
- started_at / completed_at
- run mode and scope
- exact list of references seen in search
- ideally listing-card fields as seen in that run

This is the single biggest improvement for point-in-time reconstruction.

### Priority B — Add `captured_by_run_id` to job snapshots/history records
Every current snapshot/history version should point back to the run that produced it.
That closes a major provenance gap.

### Priority C — Preserve raw source captures for high-value evidence
At least for detail pages, consider preserving:
- raw HTML snapshot
- canonicalized fetch URL
- response status code
- maybe selected headers

This can be selective if storage is a concern, but some raw-evidence layer would materially improve archival credibility.

### Priority D — Separate “full-census seen_at” from “observed in limited/filtered run”
Do not rely on a single `last_seen` field to carry all meanings.
Consider distinct fields such as:
- `last_seen_in_full_run`
- `last_seen_in_any_run`

Or preserve that distinction only in run manifests and reconstruction logic.

### Priority E — Improve fixity for history/event logs
Consider:
- atomic writes for history snapshots everywhere
- stronger append discipline for JSONL logs
- optional periodic manifest hashing or checkpoint hashes

### Priority F — Record repair provenance explicitly
When lifecycle repair mutates a record, preserve explicit provenance fields or events that make it obvious the state was changed by a repair workflow rather than ordinary scrape discovery.

## Practical conclusion

If your standard is:
“Can I meaningfully study jobs, their content, and how they changed over time?”
then yes: the collector is already pretty good.

If your standard is:
“Can I reconstruct with archival precision exactly what Civil Service Jobs showed at a given moment, with strong provenance and fixity guarantees?”
then not yet.

The system is best described as:
- a strong historically-aware collector
- evolving toward a true archival record system
- currently needing per-run corpus manifests and stronger provenance/fixity to cross that line
