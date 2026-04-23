---
title: Architecture
---

# The system at a glance

The Longhand Archive separates two concerns: **capture** of data from public sources, and **derivation** of queryable and presentational views from that captured data.

This separation is the foundational architectural commitment. Capture and derivation have different lifecycles, different failure profiles, different audiences, and different authority. Mixing them produces systems that are fragile in all of those dimensions at once.

## Seven layers

The system is organised into seven layers of responsibility. Not seven services. Not seven repositories. Seven layers, each with a defined role and defined boundaries.

1. **Collectors** — capture data from public sources. Each collector owns one source, handles its specifics, and produces a local archive. Collectors do not know about the platform or any consumer of their output.

2. **Archive contract** — the versioned specification of what collectors produce and what consumers can rely on. The only interface between the two sides.

3. **Sync / projection layer** — reads from collector archives, projects into the platform's queryable form (Postgres). One-directional. Applies compliance transformations, including personal data redaction, at this boundary.

4. **Data layer** — domain models, queries, database access. Single Postgres instance with two schemas: `archive` (projection, derived, rebuildable) and `accounts` (user data, authoritative in its own right).

5. **Application layer** — HTTP routing, views, authentication, attribution.

6. **Presentation layer** — HTML, CSS, client-side behaviour.

7. **Public API** — versioned JSON interface for external consumers. Does not yet exist.

## Authority hierarchy

Authority flows from the original public source through the system:

1. **Original public source** — ultimate authority
2. **Collector's on-disk archive** — authoritative local copy, must not be lost
3. **Postgres database** — derived projection, rebuildable from archive
4. **Website / API** — derived from database, rebuildable from code
5. **User data** — authoritative in its own right, outside the derivation chain

If the archive and a derived system disagree, the archive is correct.

## Current state

The first collector — against the UK Civil Service Jobs site — is operational. It captures job listings, their full content, lifecycle transitions, and supporting assets (attachments, transcripts) to a local file-based archive.

The platform (layers 3–6) has architectural decisions and documentation but no application code yet. The archive contract is in early drafting, drawn from the collector's actual output shape.

---

The authoritative architecture document is maintained in the [governance repository](https://github.com/longhandarchive/governance).
