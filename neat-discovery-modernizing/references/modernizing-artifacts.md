# Modernizing Artifacts

## modernization.md

Frontmatter:

```yaml
---
project: null  # project name, inferred from codebase
date: null  # ISO date (YYYY-MM-DD)
codebase_analysis: null  # "ok" | "skipped"
legacy_path: null  # path to the legacy system, set in Phase 1 Step 2
status: draft  # Set to "final" when quality checks complete
---
```

Content schema:

```jsonc
{
  "## Executive Summary": null, // 2–3 paragraphs: what the system is, why it needs transformation (key evaluation findings), and the transformation plan at a high level (TP count and sequencing). Written last.
  "## Legacy System Analysis": null, // Contains five sub-sections: System Summary, Domain Summary, Module Map, Quality Findings, Improvements — collected across Phases 2, 3, 5 and written in Phase 7
  "## Evaluation": null, // table: Dimension | Score (Red/Amber/Green) | Evidence | Key findings — one row per dimension: Structural complexity, Changeability, Operational readiness, Technology health, Data complexity
  "## Constraint Checks": null, // table: Constraint | Finding | Impact on transformation | Improvement tasks — one row per constraint evaluated (Availability/SLA, Compliance/regulatory); omit section if no constraints apply
  "## Transformation Phases": null, // table: ID | Name | Triggered by | Depends on | Pattern | Size | Complexity | Risk | Architectural changes | Improvement tasks | Go/no-go criteria | Rollback — one row per phase (TP-1, TP-2, …); Size must be S/M/L after decomposition
  "## Requirements": null // table: ID | Requirement | Requires | Source — one row per derived requirement (from Phase 6); ordered by TP (all TP-1 rows, then TP-2, etc.); ID = REQ-NNN sequential across all TPs, starting REQ-001; Requirement = improvement task text (or drafted from TP metadata); Requires = comma-separated REQ-IDs derived from the upstream TP (resolved from this TP's `Depends on` by exact ID match) or "—"; Source = TP-N (traceability — internal to modernization.md; not propagated downstream)
}
```

### Legacy System Analysis Sub-sections

The `## Legacy System Analysis` section contains five required sub-sections:

#### System Summary

Contains: Foundation, Architecture, Contracts, Flows, Conventions/Patterns.

#### Domain Summary

Domain entities and their relationships extracted from codebase scan.

#### Module Map

Component breakdown with a `#### Shared Packages` sub-section.

#### Quality Findings

Issues and technical debt discovered during codebase scan.

#### Improvements

Unscheduled Amber dimension improvements that were not assigned to any transformation phase.

**Schema:**

Markdown table with two columns:

| Dimension | Improvement task |
| --- | --- |
| {Dimension name} | {improvement task description} |

**Rules:**
- One row per unassigned Amber dimension from Phase 3
- Only includes Amber dimensions whose system area is not touched by any transformation phase
- Omit section if all Amber improvements were assigned to phases
- Must stay machine-distinguishable from `## Constraint Checks` (different column structure)
