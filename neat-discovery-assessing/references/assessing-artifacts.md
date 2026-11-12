# Discovery Artifacts

## Initial Mode

### prd.md

**Path:** `specs/discovery/prd.md`

Frontmatter:

```yaml
---
status: draft  # Set to "final" when quality checks complete
---
```

Content schema:

```jsonc
{
  "## Executive Summary": null,                  // 2–3 paragraphs: problem, goal, scope. Written last.
  "## Problem Statement & Current State": null,  // what's broken, current baseline, constraints
  "## Goals & Success Metrics": null,            // table: Goal | Metric | Target | How measured; leave targets as {TBD} if baseline unknown
  "## Constraints": null,                        // plain bullets: budget, timeline, regulatory, technology
  "## Project Context": null,                    // **Team Structure:** {solo | small-team | multi-team}; **Repository Preference:** {monorepo | multi-repo | {inferred}} (multi-product only, omit otherwise); **Domain Complexity:** {simple | moderate | complex}; **Rationale:** {brief explanation — domain count, dependency count, key factors}; **Scale Indicators:** Active count, Deferred count, size distribution
  "## Products": null,                           // multi-product only: list of product name + description pairs
  "## Requirements": null,                       // ### Delivery Objectives sub-table first (Initial snapshot — counts reflect discovery-time classification): # | Objective | Requirements (comma-separated REQ-ID list) | Active | Deferred; then ### Requirements table: ID | Requirement | Priority [| Product — multi-product only]; includes integration requirements
  "## User Journeys": null,                      // Mermaid flowcharts: one per primary user path (typically 3–5); graph TD format; show decisions, branches, happy path, key error states; include auth flows if relevant; omit section if no UX artifacts confirmed in Phase 2 Step 2
  "## Layout Design": null,                      // ASCII wireframes: one per unique page/screen type; use Unicode box-drawing (┌─┐│├┤└─┘); label regions (Header, Nav, Content, Sidebar, Footer, Modal); note responsive breakpoints if desktop/mobile differ; omit section if no UX artifacts confirmed
  "## Non-Functional Requirements": null,        // table: ID | NFR | Target
  "## Delivery Estimate": null,                  // table: REQ-ID | Title | Scope | Size | Complexity | Risk | Rationale | Reason | Promote-when (Reason & Promote-when: Deferred rows only); effort summary (Active / Deferred / Full Scope + NFR buffer note); Mermaid dependency map
  "## Risks & Assumptions": null,               // ### subheading per risk level (High → Med → Low) with colored callout blocks; omit a level heading if no risks exist at that tier; final ### Assumptions subsection: plain bullets: assumption | impact if wrong
  // Open Questions: embed inline within the PRD section they belong to, not as a trailing section.
  // Format: one `{open: <topic>} <description>` callout per question (../../references/neat-project/gap-resolution.md § Node Schema) — downstream skills grep for `{open:`.
}
```

## Iteration Mode

### requirements{NN}.md

**Path:** `specs/discovery/requirements{NN}.md` (NN = zero-padded 2-digit iteration number)

Frontmatter:

```yaml
---
status: draft  # Set to "final" when quality checks complete
---
```

Content schema:

```jsonc
{
  "## Active": null,         // table: REQ-ID | Requirement | Size | Complexity | Risk | Rationale [| Product — multi-product only]; risks (plain bullets: risk | likelihood (Low/Med/High) | mitigation); effort summary (Active / Deferred / Full Scope + NFR buffer note); Mermaid dependency map
  "## Deferred": null,       // table: REQ-ID | Requirement | Size | Complexity | Risk | Rationale | Reason | Promote-when [| Product — multi-product only]
  "## Non-Functional Requirements": null, // omit section if no new NFRs this phase; table: ID | NFR | Target, continuing the NFR-ID sequence from prior files
  // Open Questions: embed inline within the Active, Deferred, or Non-Functional Requirements section they belong to, not as a trailing section. One `{open: <topic>} <description>` callout per question (../../references/neat-project/gap-resolution.md § Node Schema).
}
```

Do **not** update `prd.md`.
