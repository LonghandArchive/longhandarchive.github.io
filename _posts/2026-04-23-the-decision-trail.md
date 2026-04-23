---
layout: default
title: "The decision trail"
date: 2026-04-23
---

Every non-trivial project accumulates decisions. Most of them disappear into commit messages, Slack threads, or the heads of the people who made them. When someone later asks "why does it work like this?" the answer is usually a shrug or a guess.

The Longhand Archive records its decisions as Architecture Decision Records — short documents that capture what was decided, why, and what follows from it. The format is not original; Michael Nygard proposed it in 2011 and it has been widely adopted since. What matters is not the format but the discipline: every load-bearing decision gets written down, with enough context that a future reader can judge whether the reasoning still holds.

The project now carries twenty decisions across three repositories: ten at the governance level covering how the project operates, seven for the Civil Service Jobs collector covering how the archive works, and three for the platform covering how the derived analytical surface will be built.

The decisions range from the structural — the archive is a local-first file tree, not a database — to the operational — a job is closed when its advertised close date passes, regardless of whether the site still shows it. Some formalise choices that were already implicit in the code. Others resolve questions that were genuinely open.

The most useful decisions are the ones that prevent re-litigation. When someone — including a future version of the author — considers changing something fundamental, the first step is finding the ADR and reading the context. If the context is convincing and the circumstances have not changed, the decision stands. If the circumstances have changed, a new ADR supersedes the old one and says why.

The [full decision trail](/decisions/) is on this site. The source documents live in the repositories where they apply.
