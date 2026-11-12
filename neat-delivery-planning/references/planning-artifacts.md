# Planning Artifacts

## Workspace Level

### delivery.json (`specs/delivery/delivery.json`)

**Base schema:** See [../../references/neat-project/shared-artifacts.md](../../references/neat-project/shared-artifacts.md) § delivery.json

**Planning usage:** Phase 4 writes this file and updates wave status `pending` → `planned`.

```jsonc
{
  "planning_model": null,   // "phase-first" | "product-first" — shared vs. per-product paths
  "product_col": null,      // true when wave tables carry Product column
  "products": null,         // string[] (shared, multi-product) OR object[] (per-product: { "name", "waves": [...] })
  "waves": [                // omit at top level for per-product paths (nested in products instead)
    { "wave": null, "status": null }  // slug (e.g. "wave01-booking"); status: "pending" | "planned" | "building" | "built"
  ]
}
```

JSON format — programmatic status tracking, not human-readable spec.

### system.md (`specs/system/system.md`)

Freeform markdown. Opens with `mode: greenfield | brownfield` — written by solutioning, updated by building; planning reads it (Phase 5 Step 2).

**Layout:** `## Foundation` (tech stack, dependencies, infra/CI, observability, secrets, testing) · `## Code Structure` (file tree, placement rules) · `## Conventions` (`### Test Framework` — owned by solutioning; `### Patterns` — owned by codebase-analysis, overwritten on rescan) · `## Architecture` (component relationships) · `## Contracts` (API surface) · `## Business Rules` · `## Domain Concepts` · `## Edge Cases`

### issues.md (`specs/delivery/issues.md`)

**Base schema:** See [../../references/neat-project/shared-artifacts.md](../../references/neat-project/shared-artifacts.md) § issues.md

**Planning-specific:** `Blocker: true` gates next planning run; `Resolution` field: "pending (REQ-NNN to be assigned)" → assigned REQ-NNN (set in Phase 2 Step 2, Phase 4 Step 1). Written in Phase 2 Step 1 (seed from solution.md), Phase 3 (requirements gaps), Phase 5 Step 3 (task gaps). Follow [recommendation-format.md](../../references/neat-project/recommendation-format.md) for Resolution guidance.

### decisions.md

Flat log grouped as `## {req-id}: {requirement title}` (or `## {wave-slug}: {wave description}` for cross-requirement decisions). Follow [recommendation-format.md](../../references/neat-project/recommendation-format.md) for "Why" field guidance.

```jsonc
{
  "### {title}": null,
  "**Decision:**": null,      // what was chosen (one sentence)
  "**Why:**": null,           // cite reference (Proven) or reasoning (Inferred)
  "**Implications:**": null,  // constraints/consequences for future work
  "**Source:**": null,        // "solution.md" (Phase 3 seed); omit otherwise
  "**ADR:**": null            // "adrs/adr-{NNN}-{slug}.md" (cross-scope only — names type/interface/config/module used across >1 wave)
}
```

### adrs/adr-{NNN}-{slug}.md (`{kb}/adrs/adr-{NNN}-{slug}.md`)

One file per cross-scope decision. NNN: highest existing + 1 (zero-padded 3 digits; 001 if none). Slug: title lowercased, spaces→hyphens, non-alphanumeric removed, ≤50 chars. Follow [recommendation-format.md](../../references/neat-project/recommendation-format.md) for "Why" field.

```jsonc
{
  "# ADR-{NNN}: {title}": null,
  "**Date:**": null,           // YYYY-MM-DD
  "**Status:**": null,         // accepted | deprecated | superseded-by ADR-{NNN}
  "**Context:**": null,        // situation/constraint prompting decision
  "**Decision:**": null,       // copy verbatim from decisions.md
  "**Alternatives:**": null,   // options considered, why rejected
  "**Why:**": null,            // copy verbatim from decisions.md (cite if Proven; explain if Inferred)
  "**Consequences:**": null,   // Positive / Negative / Risks
  "**Scope:**": null           // requirement IDs, wave slugs
}
```

**Rules:** Create only when `decisions.md` entry tagged `ADR:` — copy Decision/Why verbatim, expand with Date/Alternatives/Consequences/Scope. Architectural decisions only (tech commits, hard-to-reverse choices). Status starts `accepted`. Never replaces `decisions.md` entry. No ADRs for wave-local decisions.

## Phase Level

### wave{NN}-{name}.md (`{base}/wave{NN}-{name}.md`)

Consolidated wave file — requirements, features, tasks. Frontmatter: `status: draft` (→ `planned` in Phase 8 Step 3).

```markdown
## Requirements
| ID | Type | Requirement | Requires | Risks | [Product] |
|----|------|-------------|----------|-------|-----------|
| REQ-001 | feature | User auth | | Med: session | [product-name] |

## Features
### feature-auth
**Description:** User authentication with login, signup, password reset.
**Acceptance Criteria:** [bulleted list]
**Depends On:** none
**Tasks:**
1. (step 1, implementation) Create User model and auth endpoints
   - Files: src/models/user.ts, src/routes/auth.ts
   - Inputs: Database schema from solution.md
   - Outputs: User model, /auth/login endpoint
   - Conventions: Follow REST API patterns from system.md
   - Acceptance Criteria: Endpoints respond with 200 on valid credentials
```

**Rules:** NN: 2-digit wave number (01-99). Name: slug from requirements (e.g. "booking"). Requirements table: compress titles ("User authentication: implement login, signup, password reset, and session management with JWT tokens" → "User auth (login, signup, reset)"); compress risks ("Medium risk due to session management complexity" → "Med: session mgmt"). Product column only if `product_col: true` in delivery.json. Issue-derived rows: `Requirement` = issue description + `[ISS-NNN]`; `Type` = `bug`/`chore`; `Risks` = description + `[blocker]` if blocker; ID continues from highest discovery REQ (or REQ-001); in `issues.md` set `Status: addressed`, `Resolution: REQ-NNN`. Features: one ### per feature (Description, Acceptance Criteria, Depends On, Tasks). Tasks appended in Phase 7.

### plan.md (`{base}/plan.md`)

Human-readable overview — written after Phase 6 (features), finalized after Phase 7 (tasks). Frontmatter: `status: draft` (→ `final` after Phase 7). One section per wave:

```markdown
## Wave: {wave-slug}
**Requirements:** REQ-XXX {Title}, REQ-YYY {Title}
### Features
- **{feature-name}** ({slug}) — {purpose}
  - Depends on: {depends_on or "none"}
### Decisions
- {title from decisions.md} (link)
### Risks
- {risk from requirements/features}
### Open Issues
- {open issue or {open:} callout}
```

~30-60 lines per run — overview only, not feature file copy. **Factory contract:** triggers plan review gate (Gate 3) when `status: final`.

## Feature Level

### feature.md

```jsonc
{
  "name": null,                 // feature slug (matches wave file heading)
  "description": null,          // what/why
  "acceptance_criteria": null,  // conditions for acceptance
  "depends_on": null,           // feature slugs that must complete first
  "tasks": [
    {
      "slug": null,                 // kebab-case, unique within feature
      "step": null,                 // execution order (≥1); same step → parallel
      "type": null,                 // "implementation" (write/modify source) | "configuration" (manual infra/CLI/console/env) | "iac" (write IaC files — .tf/.ts/template.yaml; execution deferred to runbook) | "cicd" (write pipeline config — .github/workflows/*.yml, buildspec.yml; no runbook, triggers on push)
      "files": null,                // prose: path, functions/classes, R/W intent
      "inputs": null,               // what this task needs
      "outputs": null,              // what this task produces
      "conventions": null,          // relevant subset of global conventions
      "acceptance_criteria": null   // completion conditions (verifiable without reading implementation)
    }
  ]
}
```

**Rules:** Step numbers scoped to feature (no cross-feature deps via step). Mixed-type tasks must split — valid: `implementation`+`configuration`, `implementation`+`iac`, `iac`+`configuration`, `cicd`+`configuration`; invalid: `implementation`+`cicd`, `iac`+`cicd`. `iac` execution deferred to runbook; `cicd` execution on git push (no runbook). `tasks` omitted in Phase 6, appended in Phase 7. **Tracking:** Building tracks task completion and feature status in `{base}/wave{NN}-{name}-progress.json`.

## Phase Level (continued)

### review.md (`specs/delivery/review.md` or `specs/delivery/{product}/review.md`)

Project-level review policy for building's subagent review — created by planning Phase 1 if absent, read by building Phase 4. All sections optional (empty tables valid).

```jsonc
{
  "# Review": null,
  "## Additional passes": [
    { "Pass": null, "Instruction": null }  // e.g. "Security" — "what to check (one sentence)"
  ],
  "## Severity overrides": [
    { "Finding type": null, "Override": null }  // e.g. "Security" — "Always Important regardless of scope"
  ],
  "## Additional exclusions": [
    { "Pattern": null, "Reason": null }  // e.g. "src/gen/**" — "Generated by build pipeline"
  ]
}
```

**Rules:** All sections optional (absent = empty → building applies standard passes only). Standard passes always applied: Delivery correctness (implementation satisfies acceptance criteria), Architectural fit (no system.md conflicts), Convention compliance (no system.md §Conventions/§Business Rules violations). Additional passes extend standard; Severity overrides override standard severity; Additional exclusions supplement standard. No frontmatter — project-level, all phases; edit file directly for policy changes (git history records).
