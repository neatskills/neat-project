# Recommendation Format

Global standard for presenting recommendations, findings, and decisions across all neat-project skills.

## Core Principle

Every recommendation must include **justification** (why this specific recommendation) and **evidence type** (whether backed by proof or based on judgment).

## Standard Format

```
**[Category]** — Evidence | Issue | Recommendation | Why
```

### Fields

```jsonc
{
  "Category": null,       // classification of the finding — review dimension, decision topic, or issue type
  "Evidence": null,       // type of backing — "Proven" | "Inferred" (see criteria below)
  "Issue": null,          // what's wrong or unclear — the gap, flaw, ambiguity, or decision point
  "Recommendation": null, // what to do — specific action or choice
  "Why": null             // justification — Proven: cite the reference; Inferred: explain the reasoning
}
```

### Evidence Types

#### **Proven** — Backed by concrete reference

Can point to a specific artifact that proves this recommendation:

- Requirement explicitly states the behavior
- system.md contains the constraint/rule
- Existing code shows the pattern
- ADR mandates the approach
- solution.md specifies the design
- Test framework or quality gate requires it

**Why field format:** Cite the reference
> Why: system.md §Conventions mandates auth on all API endpoints; auth-middleware pattern exists in user-service module

---

#### **Inferred** — Based on judgment or trade-off analysis

Making a judgment call or interpreting intent:

- Choosing between valid alternatives (trade-offs)
- Applying best practices (not mandated)
- Interpreting ambiguous requirements
- Recommending patterns (not enforced)
- Making assumptions about performance/scale/complexity

**Why field format:** Explain the reasoning
> Why: API is called by mobile clients (session cookies harder to manage on mobile); system.md §Foundation shows existing services use JWT; short TTL balances security vs UX

---

## Usage Across Skills

### Reviews

All review prompts use this format:

- [Design Review](../../neat-delivery-fixing/references/design-review.md)
- [Task Review](../../neat-delivery-planning/references/task-review.md)
- [Solution Review](../../neat-discovery-solutioning/references/solution-review.md)
- [Implementation Review](../../neat-delivery-building/references/implementation-review.md)

### Decisions (decisions.md entries)

When recording decisions in `decisions.md`, use the schema in [../../neat-delivery-planning/references/planning-artifacts.md](../../neat-delivery-planning/references/planning-artifacts.md) § decisions.md. Evidence type is implicit:

- If "Why" cites system.md/requirements → Proven
- If "Why" explains trade-offs/reasoning → Inferred

### Issues (issues.md entries)

When creating issues in `specs/delivery/issues.md` (schema: [shared-artifacts.md](shared-artifacts.md) § issues.md), the "Why" principle applies to Resolution. When accepting an issue, Resolution should cite the reason (Proven: reference; Inferred: reasoning).

### Gap Resolution

When presenting unresolved gaps from gap-resolution loops, apply the same principle:

```markdown
{open: Which authentication method?}

**Context:** API will be called by mobile clients and web app
**Recommendation:** JWT with 1-hour TTL
**Why:** [Inferred] Mobile clients make session cookies harder to manage; system.md §Foundation shows existing services use JWT; short TTL balances security vs UX
```

User can then discuss alternatives if uncertain.

### Retro (Project Conventions)

When proposing project conventions in retro Step 2:

```markdown
> Project convention found: Money values must use BigDecimal, never float
> **Why:** [Proven] system.md §Business Rules mandates precision for financial calculations; float rounding caused ISS-042
> Add to CLAUDE.md?
```

---

## Alternatives Discussion

**Do not pre-list alternatives** in the initial finding/recommendation.

When the user is uncertain and asks about alternatives (e.g., "What else could we do?", "Why not X?"), then discuss alternatives in context with their specific concerns.

**Rationale:**

- Reduces verbosity for recommendations that will be accepted
- Alternatives emerge naturally in discussion
- User's questions guide which trade-offs are relevant
