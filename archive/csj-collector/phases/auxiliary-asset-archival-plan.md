---
title: "CSJ Auxiliary Asset Archival Plan"
permalink: /archive/csj-collector/phases/auxiliary-asset-archival-plan/
---

# CSJ Auxiliary Asset Archival Implementation Plan

> For Hermes: Use subagent-driven-development skill to implement this plan task-by-task.

Goal: Make attachments and YouTube transcripts first-class archival assets, managed with the same rigor as role description changes.

Architecture: Separate asset capture/versioning from main job-detail parsing. The final job JSON should reference stable asset metadata rather than volatile local filenames. Asset content hashes and stable identifiers must be computed before the final job record is hashed and saved, so job history reflects the actual saved record.

Tech Stack: Python collector at `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`, markitdown, youtube-transcript-api, JSON manifests/events, file hashing via hashlib.

---

## Problem Summary

The current collector can now:
- capture contact details more reliably
- detect attachments and YouTube links
- download attachments
- convert attachments to Markdown
- fetch YouTube transcripts

But auxiliary assets are not yet managed to the same archival standard as role descriptions because:
- the job record is hashed before attachments/transcripts are enriched
- reruns create duplicate files and path churn
- local paths are treated like important data even though they are implementation details
- attachments/transcripts do not have first-class versioning/history/events
- historical job snapshots do not point to immutable historical asset versions

---

## Phase 1 — Make the Current Model Internally Correct

### Task 1: Move asset enrichment before final hashing

Objective: Ensure the saved job JSON matches its stored `content_hash`, `field_hashes`, and `last_changed_at`.

Files:
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`

Implementation:
1. Refactor `fetch_one()` so the flow is:
   - fetch detail page
   - parse raw fields
   - merge listing fields
   - enrich raw record with downloaded attachments/transcripts
   - only then normalize/hash/diff
   - save final record
2. If needed, split `normalize_job()` into:
   - pre-hash cleanup/normalization
   - post-enrichment finalization + hashing
3. Ensure no enrichment happens after the final comparable record is produced.

Verification:
- Run a known job with attachments/transcripts
- Save JSON
- Recompute comparable hash from the final saved JSON and confirm it matches stored metadata

### Task 2: Stop using volatile local paths as change-significant identity

Objective: Prevent false-positive history events caused by filename/path churn.

Files:
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`

Implementation:
1. Define volatile asset fields that should not drive job-level meaningful change:
   - `local_path`
   - `local_filename`
   - `markdown_path`
   - `transcript_path`
   - `transcript_md_path`
   - `download_error`
   - `transcript_error`
2. Update the comparable-record hashing logic to ignore these fields.
3. Compare stable asset fields instead:
   - normalized source URL
   - logical title/name
   - content hash
   - media type
   - byte size
   - asset type/category

Verification:
- Re-run an unchanged job twice
- Confirm no meaningful change is emitted just because the physical path differs

### Task 3: Add real content hashes for assets

Objective: Give attachments/transcripts stable identities based on actual content.

Files:
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`

Implementation:
1. For each downloaded attachment, compute and store:
   - `content_hash` (sha256)
   - `byte_size`
   - `media_type`
   - `fetched_at`
2. For each Markdown derivative, compute and store:
   - `markdown_content_hash`
3. For each transcript, compute and store:
   - `transcript_content_hash`
   - `transcript_md_content_hash`
   - `transcript_length`
   - `video_duration`
   - `transcript_fetched_at`
4. Reuse these stable hashes in job-level comparison instead of local paths.

Verification:
- Re-download same attachment twice and confirm identical content hash
- Re-fetch same transcript twice and confirm identical transcript hash

### Task 4: Make attachment/transcript capture idempotent

Objective: Stop duplicate files and synthetic changes on reruns.

Files:
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`

Implementation:
1. Replace filename-suffix dedup (`_1`, `_2`, etc.) with content-based identity.
2. Recommended naming scheme:
   - `csj_attachments/{reference}/{slug}__{short_hash}.ext`
3. If the same content already exists for that job, reuse it.
4. Only create a new file when content actually changes.
5. Apply the same principle to transcript files and Markdown derivatives.

Verification:
- Run the same role twice
- Confirm no new files are created when source content is unchanged

---

## Phase 2 — Promote Auxiliary Files to First-Class Archival Assets

### Task 5: Introduce a per-job asset manifest

Objective: Separate stable asset metadata from transient job JSON details.

Files:
- Modify: `~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py`
- Create directory: `~/.hermes/workspace/csj/csj_asset_manifests/`

Implementation:
Create `csj_asset_manifests/{reference}.json` with entries containing:
- `asset_id`
- `reference`
- `asset_type`
- `logical_name`
- `source_url_raw`
- `source_url_normalized`
- `content_hash`
- `media_type`
- `byte_size`
- `first_seen_at`
- `last_seen_at`
- `fetched_at`
- `status`
- `derived_from_asset_id`
- `local_path`

Verification:
- Capture role 454928
- Confirm two attachment assets + transcript assets appear in the manifest

### Task 6: Add asset-level history and events

Objective: Track asset changes explicitly.

Files:
- Modify: `collector.py`
- Create:
  - `~/.hermes/workspace/csj/csj_asset_events.jsonl`
  - `~/.hermes/workspace/csj/csj_asset_history/{reference}/`

Implementation:
Emit events:
- `asset_added`
- `asset_removed`
- `asset_changed`
- `asset_download_failed`
- `transcript_added`
- `transcript_changed`
- `transcript_unavailable`

Store immutable per-version asset metadata snapshots.

Verification:
- Simulate an attachment content change
- Confirm `asset_changed` event and preserved previous version

### Task 7: Link job history snapshots to exact asset versions

Objective: Allow full reconstruction of historical advert state.

Files:
- Modify: history-writing logic in `collector.py`

Implementation:
Each job history snapshot should record the asset IDs + content hashes active at that time.
Do not rely on mutable current local paths.

Verification:
- Create two versions of a role with changed attachment
- Confirm old history snapshot still references old asset version

### Task 8: Define closed-job archival completeness

Objective: Give closed jobs an explicit archive state.

Files:
- Modify: lifecycle handling in `collector.py`

Implementation:
Add `archive_completeness`, e.g.:
- `complete`
- `partial_missing_assets`
- `partial_failed_transcripts`
- `no_auxiliary_assets`

On closure:
- freeze current asset manifest state
- retain all captured versions
- record unresolved failures/incompleteness

Verification:
- Close a role with full capture and one with missing assets
- Confirm different archive completeness states

---

## Phase 3 — Improve Extraction Quality and Reliability

### Task 9: Scope extraction to the vacancy content region

Objective: Reduce noisy `supporting_links` and accidental captures from page chrome/footer.

Files:
- Modify: `collector.py`

Implementation:
- Isolate the vacancy content container first
- Only scan links/embeds inside that subtree

Verification:
- Compare before/after link capture on known jobs

### Task 10: Improve YouTube metadata capture

Objective: Make transcript records more useful and stable.

Files:
- Modify: `collector.py`

Implementation:
- Support watch/shorts/youtu.be/embed URLs consistently
- Derive better transcript title from surrounding sentence/context, not just link text like `here`
- Store transcript language when available

Verification:
- 454928 transcript title should reflect Emran Mian / DSIT context

### Task 11: Make dependency failures explicit

Objective: Avoid silent archive gaps.

Files:
- Modify: `collector.py`

Implementation:
If `markitdown` or `youtube-transcript-api` is missing:
- do not fail the whole job scrape
- do record explicit asset/job-level failure metadata/events

Verification:
- Simulate missing dependency
- Confirm archive state records the failure

### Task 12: Add dedicated asset refresh/backfill policy

Objective: Treat asset freshness separately from role-text freshness.

Files:
- Modify: CLI/refresh flow in `collector.py`

Implementation:
Future flags:
- `--refresh-assets`
- `--retry-failed-assets`
- `--backfill-assets`

Use these to:
- retry failed downloads
- retry transcript fetches
- rerun markdown conversion
- verify stored asset hashes where needed

Verification:
- Mark an asset failed, rerun backfill, confirm selective repair

---

## Recommended Target Data Model

### Job JSON should contain
- stable attachment/embed summaries
- `asset_manifest_path`
- `archive_completeness`

### Asset manifest entries should contain
- `asset_id`
- `asset_type`
- `logical_name`
- `source_url_raw`
- `source_url_normalized`
- `content_hash`
- `media_type`
- `byte_size`
- `first_seen_at`
- `last_seen_at`
- `fetched_at`
- `status`
- `derived_from_asset_id`
- `local_path`

Design rule: job-level comparable hashes must use stable asset identities and content hashes, not volatile local file paths.

---

## Implementation Order

1. Fix hash timing bug
2. Make attachment/transcript capture idempotent with content hashes
3. Strip volatile local-path fields from change-significant comparisons
4. Add per-job asset manifest
5. Add asset events/history
6. Add closed-job archive completeness
7. Improve extraction scoping and YouTube metadata
8. Add dedicated asset refresh/backfill commands

---

## Success Criteria

The implementation is in a good state when:
- rerunning an unchanged job creates no new asset files
- attachment-only changes produce real historical events
- transcript-only changes produce real historical events
- old job snapshots reconstruct the exact asset versions current at the time
- closed roles record archive completeness
- missing dependencies create explicit archive warnings, not silent gaps

---

## Verification Commands

Use these after each phase:

```bash
python3 -m py_compile ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py
```

Known useful test references:
- `457480` — contact fix case
- `453120` — attachment + contact
- `453156` — PDF + contact
- `454928` — two attachments + YouTube transcript

Suggested manual verification runs:
```bash
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --details -n 1 --full --force
python3 ~/.hermes/skills/research/civil-service-jobs-collector/scripts/collector.py --details -n 1 --refresh
```

Expected outcomes:
- no duplicate unchanged files on rerun
- stable asset metadata
- consistent job hashes after enrichment
- transcript and attachment files stored in predictable stable locations
