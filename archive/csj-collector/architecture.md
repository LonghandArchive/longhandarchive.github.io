---
title: "CSJ Collector Architecture"
permalink: /archive/csj-collector/architecture/
---

# Civil Service Jobs — Data Architecture

## Overview

The Civil Service Jobs (CSJ) collector maintains a **historical archive** of UK government job postings from civilservicejobs.service.gov.uk. Unlike the public website (which only shows current listings at a point in time), this system captures, version-tracks, and archives every job over its full lifecycle — from first appearance to closure.

The system runs two scheduled jobs:
- **Incremental checks** (6x daily, every 3 hours starting 06:00 UTC) — captures new jobs
- **Full sweep** (daily at 21:00 UTC) — consistency pass, lifecycle management, stale data refresh

## Data Storage

All data lives under `~/.hermes/workspace/csj/` in a flat directory structure:

```
csj/
├── csj_jobs/                    # Current state of every job (one JSON per reference)
│   ├── 456882.json
│   ├── 457103.json
│   └── ...
├── csj_history/                 # Versioned snapshots per job
│   ├── 456882/
│   │   ├── 2026-04-10T14:30:00.json
│   │   ├── 2026-04-12T09:15:00.json
│   │   └── ...
│   └── ...
├── csj_events.jsonl             # Append-only lifecycle event log
├── csj_state.json               # Collector state (last reference, last run, count)
├── csj_latest.json              # Summary of most recent scrape run
├── csj_attachments/             # Downloaded attachments + transcripts
│   ├── _pool/                   # Content-addressed deduplicated storage
│   │   ├── a3f2b1c9...pdf       # Files stored by full SHA-256 hash
│   │   ├── a3f2b1c9...pdf.md    # Markdown conversion alongside original
│   │   └── ...
│   ├── 456882/                  # Per-job directory (symlinks into _pool/)
│   │   ├── role_profile__a3f2b1.pdf -> ../../_pool/a3f2b1c9...pdf
│   │   ├── role_profile__a3f2b1.pdf.md -> ../../_pool/a3f2b1c9...pdf.md
│   │   └── ...
│   └── ...
├── csj_asset_manifests/         # Per-job asset metadata (current state)
│   ├── 456882.json
│   └── ...
├── csj_asset_history/           # Immutable per-asset version snapshots
│   ├── 456882/
│   │   ├── asset_v1.json
│   │   └── ...
│   └── ...
└── csj_asset_events.jsonl       # Asset-level lifecycle events
```

## Job Record Schema

Each job file in `csj_jobs/{reference}.json` contains ~42 fields:

### Identity & Metadata
| Field | Type | Description |
|-------|------|-------------|
| `reference` | string | Unique CSJ reference number (globally sequential, never reused) |
| `title` | string | Job title |
| `url` | string | Direct link to job detail page |
| `schema_version` | string | Record format version (currently "2.2") |
| `parser_version` | string | Parser version (currently "2.7") |

### Core Job Fields
| Field | Type | Description |
|-------|------|-------------|
| `department` | string | Hiring department (138 unique values) |
| `grade` | string | Raw grade string from listing |
| `grade_normalized` | string\|null | Canonical grade: AA/AO/EO/HEO/SEO/G7/G6/SCS1-4 |
| `salary` | string | Raw salary text (e.g. "£48,350 - £57,500") |
| `salary_min` | int\|null | Lower bound parsed from salary string |
| `salary_max` | int\|null | Upper bound parsed from salary string |
| `contract_type` | string | Permanent/Fixed term/etc |
| `business_area` | string\|null | Business area within department |
| `working_pattern` | array | e.g. ["Flexible working", "Full-time"] |
| `location` | string | Raw location text |
| `location_primary` | string\|null | Cleaned city list (postcodes/regions stripped) |
| `closes` | string | Human-readable closing date |
| `closes_iso` | string\|null | ISO datetime closing date |
| `num_roles` | int | Number of positions |
| `security_clearance` | string\|null | BPSS/CTC/SC/DV extracted from text |

### Lifecycle Fields
| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Broad state: "active" or "closed" |
| `lifecycle_status` | string | Specific: "active", "closed", "missing_unconfirmed", "withdrawn_confirmed" |
| `first_seen` | string | ISO timestamp when reference was first scraped |
| `last_seen` | string | ISO timestamp when last confirmed live in search results |
| `first_missing_at` | string\|null | When role first disappeared before confirmed closure |
| `consecutive_missing_runs` | int | Full unfiltered runs the role stayed missing |
| `last_changed_at` | string\|null | Last time meaningful content changed |
| `scraped_at` | string | ISO timestamp of this particular scrape |

### Text Content (large fields)
| Field | Type | Description |
|-------|------|-------------|
| `job_summary` | string\|null | Job summary section |
| `job_description` | string\|null | Full job description |
| `person_spec` | string\|null | Person specification |
| `benefits` | string\|null | Benefits and pension info |
| `contact` | string\|null | Contact details |
| `full_text` | string\|null | Raw page text (source of truth for re-parsing) |

### Archival & Asset Fields
| Field | Type | Description |
|-------|------|-------------|
| `content_hash` | string | Stable hash of the normalized comparable record |
| `field_hashes` | dict | Per-field stable hashes for granular diffing |
| `asset_versions` | array | Pointers to exact asset versions for this job snapshot |
| `archive_completeness` | string | "complete", "partial_missing_assets", "partial_failed_transcripts", "no_auxiliary_assets" |
| `attachments` | array\|null | Downloaded files (PDFs, DOCX, candidate packs) |
| `supporting_links` | array\|null | External links found in job description |
| `embeds` | array\|null | Embedded media (YouTube/Vimeo iframes) |

## How Scraping Works

### 1. Authentication
The CSJ site uses [ALTCHA](https://altcha.org), a proof-of-work CAPTCHA. The collector solves it natively in pure Python (SHA-512 brute force, <0.5s) to establish an authenticated session.

### 2. Search & Listing Extraction
- Searches all jobs (or filtered by grade/department)
- Paginates results (25 jobs per page, up to 50 pages = 1,250 max per run)
- Extracts listing-level data: title, department, locations, salary, closing date, reference, URL
- Sorts by "Most recent"

### 3. Detail Fetch (parallel)
- Filters out jobs already on disk (dedup by reference number)
- Fetches remaining job detail pages with N parallel workers (default 5)
- Extracts full posting text, attachments, supporting links, embeds
- Downloads attachments at scrape time (CSJ URLs expire — session-based CGI tokens)
- Converts attachments to Markdown via `markitdown`
- Fetches YouTube transcripts where present

### 4. Storage & Dedup
- Each job saved as `csj_jobs/{reference}.json`
- Attachments stored once in `csj_attachments/_pool/` (content-addressed by SHA-256)
- Per-job attachment directories contain symlinks into the pool (deduplicates identical files across jobs)
- Already-fetched jobs are skipped on re-run (file-level dedup)

## Job Lifecycle Management

The collector tracks each job through a defined lifecycle:

```
                    ┌─────────────────┐
                    │   first_seen    │
                    │  status: active │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
     Job still in results            Job gone from results
     ┌───────────────┐              ┌───────────────────┐
     │ last_seen     │              │  Closes date past? │
     │ updated       │              └─────────┬─────────┘
     └───────────────┘                    │          │
                                        Yes         No
                                  ┌───────┴──┐  ┌────┴───────────┐
                                  │  closed   │  │missing_unconfirmed│
                                  │ status:   │  │ consecutive_    │
                                  │  closed   │  │ missing_runs++  │
                                  └──────────┘  └────┬───────────┘
                                                     │
                                            After repeated runs
                                            + direct URL check
                                                     │
                                              ┌──────┴───────┐
                                              │  withdrawn_  │
                                              │  confirmed   │
                                              └──────────────┘
```

**Key rules:**
- Lifecycle pass only runs on **full unfiltered scrapes** (no `--limit`, `--grade`, `--department` filters)
- Jobs that vanish before their closing date are tracked as `missing_unconfirmed` (not immediately closed)
- After repeated missing runs, a direct-URL verification can promote to `withdrawn_confirmed`
- If a missing job reappears in results, missing counters reset and a `reopened` event is emitted
- All jobs stay in `csj_jobs/` regardless of status — filter by `status` field, not file presence

## Change Tracking & History

### Content Hashing
Every job has:
- `content_hash` — a stable hash of the full normalized record
- `field_hashes` — per-field hashes for granular change detection

When a re-scrape produces a different `content_hash`, the collector:
1. Saves a timestamped snapshot to `csj_history/{reference}/`
2. Records a `field_changed` event with `changed_fields`, `old_values`, `new_values`
3. Updates `last_changed_at` on the current job

### History Snapshots
Each `csj_history/{reference}/` directory contains immutable JSON snapshots of every meaningful version of the job. Comparing snapshots reveals:
- Salary revisions
- Closing date extensions
- Description updates
- Asset/link changes

### Event Log
`csj_events.jsonl` is an append-only log recording:
| Event Type | Meaning |
|------------|---------|
| `first_seen` | Job first captured by collector |
| `refreshed` | Job detail re-fetched, still active, no meaningful changes |
| `field_changed` | Meaningful content changed between scrapes |
| `missing_from_results` | Job vanished from search results |

## Asset Archival

### Attachments
- Downloaded at scrape time (CSJ attachment URLs are session-based and expire)
- Stored in `csj_attachments/_pool/` using content-addressed naming (`{sha256}.ext`)
- Per-job directories contain relative symlinks into the pool
- Each original file gets a `.md` Markdown conversion alongside it
- Deduplication: identical files across jobs stored only once

### YouTube Transcripts
- Videos linked in job descriptions have transcripts fetched via `youtube-transcript-api`
- Saved as both plain text (`.txt`) and timestamped Markdown (`.md`)
- Pooled and symlinked like attachments

### Asset Manifests
Each `csj_asset_manifests/{reference}.json` tracks:
- All assets associated with a job (attachments, transcripts, supporting links)
- Per-asset metadata: content hashes, media types, source URLs, file sizes
- `first_seen_at`, `last_seen_at`, status

### Asset Versioning
- `csj_asset_history/{reference}/` preserves immutable per-asset metadata versions
- `csj_asset_events.jsonl` records: `asset_added`, `asset_changed`, `asset_removed`, `transcript_added`, `transcript_changed`, `transcript_unavailable`

### Archive Completeness
Each job gets an `archive_completeness` rating:
- `complete` — all attachments and transcripts captured
- `partial_missing_assets` — some attachments failed to download
- `partial_failed_transcripts` — some YouTube transcripts unavailable (often due to cloud IP blocking)
- `no_auxiliary_assets` — job had no attachments or embeds to capture

## Reference Number Logic

- References are **globally sequential integers** assigned by CSJ (e.g. 412800 to 457524)
- They increment overall but have gaps — jobs expire, get filled, or aren't posted consecutively
- References are never reused
- Higher reference = generally more recently posted
- Current active jobs cluster in the highest reference ranges

## Current Dataset Size (as of April 2026)

| Metric | Count |
|--------|-------|
| Total job references captured | ~1,639 |
| Currently active | ~50% |
| Closed | ~50% |
| Unique departments | 138 |
| Unique raw grade strings | 150+ (mappable to ~10 canonical grades) |
| Lifecycle events | ~3,000+ |
| Jobs with attachments | ~244 |
| Jobs with supporting links | ~428 |
| Jobs with embedded video | ~43 |
