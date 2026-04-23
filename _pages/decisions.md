---
title: Decisions
---

# Architectural decisions

Every significant decision across The Longhand Archive is recorded as an Architecture Decision Record (ADR). ADRs are immutable. When a decision is reversed, a new ADR supersedes the original.

## Governance

Cross-cutting decisions that apply to the whole project. Source: [`governance/decisions/`](https://github.com/longhandarchive/governance/tree/main/decisions)

| ADR | Title |
|-----|-------|
| 0001 | New repositories require three tests |
| 0002 | Repository naming convention |
| 0003 | Monorepo by default within components |
| 0004 | British English in public copy |
| 0005 | Brand posture: independent and non-endorsed |
| 0006 | OGL attribution on all public surfaces |
| 0007 | Personal data redacted at projection boundary |
| 0008 | ADR location: repo and org |
| 0009 | Repository licensing |
| 0010 | The project builds for its own purpose |

## Civil Service Jobs Collector

Decisions specific to the first collector. Source: [`civilservicejobs-collector/docs/decisions/`](https://github.com/longhandarchive/civilservicejobs-collector/tree/main/docs/decisions)

| ADR | Title |
|-----|-------|
| 0001 | Local-first, file-based archive |
| 0002 | Date-first closure policy |
| 0003 | Qualifying run semantics and evidence gating |
| 0004 | Two-layer lifecycle model |
| 0005 | Evidence-based lifecycle transitions |
| 0006 | Content-addressed asset storage |
| 0007 | Mutable projections and immutable evidence |

## Platform

Decisions for the public analytical surface. Source: [`platform/docs/decisions/`](https://github.com/longhandarchive/platform/tree/main/docs/decisions)

| ADR | Title |
|-----|-------|
| 0001 | Two-schema database model |
| 0002 | File tree authoritative, Postgres as projection |
| 0003 | Postgres full-text search, not dedicated service |
