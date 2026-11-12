# Scoping Method

## Estimation Method

### T-Shirt Size Scale

| Size | Level | Indicators |
| --- | --- | --- |
| XS | Trivial | Config, copy, single field |
| S | Small | Simple CRUD, basic form |
| M | Medium | Multi-component, standard integration |
| L | Large | Complex feature, multiple subsystems — one team can own it |
| XL | Very Large | Major cross-system, high risk — requires cross-team coordination |
| XXL | Epic | Multi-team, architectural changes |

Sizes are relative, not time-based.

### The Estimation Process

**Step 1 — Parse Requirement:**

Extract: What the system must do, implied technical work, integration points, edge cases.

**Step 2 — Assess Technical Complexity:**

**Low:** Established patterns, single system, minimal logic, standard libraries

**High:** Novel algorithms, cross-system coordination, complex rules, migrations, performance-critical

Complexity = what needs to be built, independent of team experience.

**Step 3 — Assess Risk:**

How likely is this requirement to explode in scope (3x)? Low = well-bounded, familiar pattern, clear "done".
High = vague requirements, new tech, external dependencies, undefined edge cases. Feeds into the size decision,
the Rationale column, and is itself persisted as the Risk field.

**Step 4 — Synthesize Size & Document:**

| Complexity \ Risk | Low | High |
| --- | --- | --- |
| High | M or L | L, XL, or XXL |
| Low | XS, S, or M | M or L |

Within a range, use the Indicators column as the tiebreaker — if the requirement's work matches the XS indicator, size XS, not S.

**High Risk + XL or XXL:** scope is too unbounded to commit to. Do not classify the item as Active — Phase 4 Step 2 (Classify) will reclassify it as Deferred, pending a focused Iteration-mode assessing run to bound the scope.

Output fills one row per requirement — the Delivery Estimate table for Initial, or the Active/Deferred table (per its Phase 4 Step 2 classification outcome) for Iteration; see the table schemas in `assessing-artifacts.md`. Size, Complexity, Risk, and Rationale are set now; each requirement's Active/Deferred/Drop status (and, where applicable, Reason/Promote-when) is finalized once Phase 4 Step 2 (Classify) runs:

- **REQ-ID:** carry forward from the Requirements section
- **Title** (Initial only): 3–6 word label extracted from the requirement text. Iteration has no Title field — its Active/Deferred tables use the full requirement text under "Requirement" instead.
- **Scope** (Initial only): Active or Deferred — set from the Phase 4 Step 2 classification outcome (Drop items are excluded from this table). Iteration has no Scope column — status is implied by which table (Active or Deferred) the row appears in.
- **Size:** T-shirt size from the synthesis matrix
- **Complexity:** 1–2 sentences on what makes it hard (or not)
- **Risk:** Low or High, from Step 3 — persisted so a later Iteration's Phase 4 Step 2 reclassification rule (Active + Risk=High + XL/XXL → Deferred) can evaluate promoted items without re-estimating them
- **Rationale:** key assumption or risk from the estimate — same meaning whether the item ends up Active or Deferred
- **Reason** (Deferred rows only): why the item was deferred — set once Phase 4 Step 2 classifies it
- **Promote-when** (Deferred rows only): the condition under which the item should be promoted to Active in a future phase — set once Phase 4 Step 2 classifies it; a later Iteration's Phase 3 Step 2 re-evaluation checks against this

If info is missing, state what you assumed and what would change the estimate.

**Do not size NFRs individually.** NFR effort is embedded in the functional requirements that must satisfy them. Add a buffer note in the Effort Summary instead.

### Effort Summary and Dependency Map

Effort Summary buckets rows by Active/Deferred status, which Phase 4 Step 2 (Classify) determines — so populate both of the following only after Classify has run, not immediately after sizing:

- **Effort Summary** — count and size distribution per scope bucket (Active / Deferred / Full Scope, where Full Scope = Active + Deferred), plus NFR buffer note
- **Dependency Map** — prerequisites (REQ-XXX before REQ-YYY), parallelizable requirements, critical path, and phasing order

Both sections are populated after all requirement rows are estimated and classified (Phase 5).

## Classification Method

Applies to functional requirements newly drafted in Phase 3 Step 1 only.

- NFRs are not classified — their effort is embedded in the functional requirements that satisfy them.
- Re-evaluated Deferred items are resolved in Phase 3 Step 2, not here.

### Two-Pass Classification

Per requirement, stop at first match:

1. **Rules 1–2** — establish the Active set; union with any items promoted in Phase 3 Step 2 (rule 3 must check against the full current-phase Active set).
2. **Rules 3–6** — apply rule 3 iteratively to a fixpoint: after each requirement newly becomes Active via rule 3, re-scan the remaining unclassified requirements against the now-larger Active set, repeating until a full pass adds no new Active items. Only once this fixpoint is reached — a full pass finds no requirement newly qualifying under rule 3 — evaluate rules 4–6 once, in order, for each requirement still unclassified.

| # | Condition | Outcome |
| --- | --- | --- |
| 1 | Compliance, legal, or regulatory | **Active** |
| 2 | Removing this would make the product useless or valueless to its target users | **Active** |
| 3 | Hard prerequisite for an Active item | **Active** |
| 4 | Partially meets rules 1–3 | **Split**, then classify each part |
| 5 | Still valid but not needed for the product to be usable and valuable | **Deferred** |
| 6 | No longer relevant or explicitly out of scope | **Drop** |

### Post-Classification Rules

- Rules override user markings — flag any conflict before continuing.
- If nothing was deferred, write "Deferred: None."
- **Active + Risk=High + XL/XXL** → reclassify as Deferred (applies to all current-phase Active rows, including promoted items):
  - Reason: "scope unbounded — re-run assessing in Iteration mode with investigation findings before this can proceed"
  - Promote-when: "scope bounded to L or smaller"
  - **Demotion pass:** for each Active item that depended on the reclassified item via rule 3 only — demote to Deferred (rule 5) unless rules 1–2 or another Active prerequisite independently justify it. Repeat until stable.
- **Active + Risk=High (any size)** → add to the Risks section.
- **Deferred (rule 5):** record Reason and Promote-when.
- **Split (rule 4):** create sub-requirements (e.g. REQ-005a, REQ-005b); size each part per the estimation step before classifying each starting from rule 1.
- **Drop:** omit from all tables and the dependency map.
- Record outcomes in the Delivery Estimate table (Initial) or Active/Deferred tables (Iteration).

## Domain Complexity Method

Analyze the drafted requirements to classify domain complexity using concrete criteria:

1. **Identify distinct domains:** Count business domains/bounded contexts (e.g., User Management, Billing, Analytics are 3 domains)
2. **Count cross-domain dependencies:** Identify bidirectional dependencies where Domain A requires Domain B AND Domain B requires Domain A
3. **Assess aggregate complexity:** Identify aggregate roots (DDD entities that own consistency boundaries)

**Classify:**

- **Simple:**
  - 1 domain, <30 requirements
  - OR 2 domains with unidirectional dependencies only
  
- **Moderate:**
  - 2-3 domains, 30-100 requirements
  - <3 bidirectional dependencies between domains
  - Clear boundaries (can draw clean context map)
  
- **Complex:**
  - 3+ distinct domains, >100 requirements
  - OR ≥3 bidirectional dependencies between domains
  - OR event sourcing/CQRS patterns required
  - OR >5 aggregate roots across domains

## Deferred Re-evaluation Method

Used by assessing Phase 3 Step 2 (Iteration only).

- Dedupe Deferred items by REQ-ID — keep only the most recent phase's row. Then assess each against build history and its Promote-when condition, in this order, stopping at first match: no longer needed? partially addressed? still valid with Promote-when condition now met (scope changed or not)? still valid with Promote-when condition not yet met?
- **No longer needed** → Drop. Omit from this phase's tables and the dependency map.
- **Partially addressed** → drop the addressed part. Carry the remaining part forward as Deferred this pass (not eligible for the Promote-when check below in this same pass):
  - Requirement text: updated to reflect only the remaining scope.
  - Reason: `partially addressed: {what was resolved}`.
  - Promote-when: carried forward unchanged, unless the remaining scope makes it inapplicable — then set a new Promote-when that reflects only the remaining scope.
- **Still valid, Promote-when condition now met** → move to this phase's Active table. Carry over its prior Size/Complexity/Risk unless scope changed, in which case re-estimate in Phase 4 Step 1. Promoted items skip Phase 4 Step 2's Classify rules 1–2 and 4–6 — those apply only to requirements newly drafted in Phase 3 Step 1 — but remain subject to the Risk=High + XL/XXL reclassification rule.
- **Still valid, Promote-when condition not yet met** → remains Deferred. Carry forward into this phase's Deferred table, updating Rationale/Reason/Promote-when if scope changed.
