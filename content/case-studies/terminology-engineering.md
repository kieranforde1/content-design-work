---
title: "Terminology — Engineering Rebuild on HubSpot Infrastructure"
breadcrumb: "Terminology · Strand 3"
parent: terminology.html
organisation: HubSpot
role: Content Designer
period: "2025–2026"
status: "Active · In engineering build"
engineering_partners: 3
stats:
  - number: "~77%"
    description: Target automated checking coverage, up from ~47% baseline
  - number: "+30pp"
    description: Projected increase in automated terminology coverage (percentage points)
  - number: "3"
    description: Engineering partners scoped in
---

## Tool architecture: three layers of capability

The engineering rebuild translates the proof-of-concept into a rules-based, in-house checker that catches terminology issues directly in the engineering code review workflow. Three layers: a term database, a detection layer, and a workflow integration that surfaces issues in pull request reviews.

## Key decisions: what I chose and why

Chosen to prioritise catching issues at the code review stage — the earliest point in the development workflow where a content issue can be caught and fixed without requiring a separate content review cycle.

## Outcomes so far

Engineering scoped with three partners. Target: automated checking coverage from ~47% baseline to ~77% — a 30 percentage point increase.
