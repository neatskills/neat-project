# Modernizing Methodology

## Constraint Checks

Evaluate first — their findings constrain pattern selection in Phase 5.

| Constraint | Evidence | Finding to record |
| --- | --- | --- |
| Availability / SLA | SLA contracts, incident history | Contractual RTO/uptime — eliminates big-bang cutovers if tight |
| Compliance / regulatory | Compliance certs, legal docs | Active GDPR/PCI/HIPAA scope — mandates a compliance phase regardless of other scores |

## Dimension Scoring

Score each dimension Red / Amber / Green. Amber adds one non-blocking improvement task to the first phase touching the same system area — not its own phase.

| # | Dimension | Evidence sources | Red | Amber | Green |
| --- | --- | --- | --- | --- | --- |
| 1 | Structural complexity | `internal_imports` graph from modules scan; API consumer list | >3 circular deps or blast radius >50% of modules, OR >5 external consumers with undocumented contracts | Blast radius 20–50% OR 2–5 consumers partially documented | Blast radius <20% AND ≤2 consumers with documented contracts |
| 2 | Changeability | Test files from codebase scan; issues.md quality findings | <20% structural test coverage (test files relative to source files) or no harness | 20–60% | >60% and harness exists |
| 3 | Operational readiness | Operational tooling scan (Phase 2); DORA metrics; runbooks, ADRs | None of IaC/CI/CD/observability present; manual deploys; no runbooks | 1–2 operational capabilities present; partial docs | All three present + rollback mechanism; daily deploys; complete runbooks + ADRs |
| 4 | Technology health | `external_imports` from modules scan; `### System Summary § Foundation` secrets findings | EOL runtime, critical CVE, OR hardcoded secrets / missing auth | Deprecated (not EOL) libs OR overly permissive cloud IAM/RBAC | Current stack, no CVEs, no hardcoded secrets, least-privilege IAM/RBAC |
| 5 | Data complexity | Schema files and DB access patterns from codebase scan; docs | Distributed data, no migration tooling, >10M rows per largest table | Centralised data, partial tooling | Single store, migration tooling exists |

**Amber handling:** note each Amber dimension and its improvement task as session state — assignment to a transformation phase happens in Phase 5 once phase system areas are known. If no phase touches the area, record as an unscheduled improvement row in `## Legacy System Analysis / ### Improvements` (see SKILL.md Phase 5 Step 3) — not in Constraint Checks.

**Missing data fallback:** If no API consumer list was provided, score the consumer sub-criterion of Dimension 1 as Amber (unknown) and note the assumption.

## Phase Derivation Rules

Rules 1–7 are mutually exclusive — first match wins. Rules 8–10 are additive markers, evaluated independently of each other and of whichever rule 1–7 matched.

| Priority | Condition | Phase(s) inserted |
| --- | --- | --- |
| 1 | Compliance constraint present | Compliance remediation — mandatory first |
| 2 | Technology health Red **and** Operational readiness Red | Operational uplift first, then technology health |
| 3 | Technology health Red only | Security / runtime hardening |
| 4 | Operational readiness Red only | Operational uplift |
| 5 | Structural complexity Red + Changeability Red | Test coverage first, then decomposition |
| 6 | Structural complexity Red + Changeability Green or Amber | Decomposition |
| 7 | Changeability Red only (not already covered by rule 5) | Test coverage phase |
| 8 | Data complexity Red | Data migration — isolated and late |
| 9 | Structural complexity Red (any module where ≥3 other modules depend on it via `internal_imports`, or any module with documented external consumers) | Record `strangler_fig: mandated` for Phase 5 — no big-bang decomposition permitted |
| 10 | Availability/SLA constraint present | Record `sla_constraint: true` for Phase 5 — blue/green or canary mandated |

**Depends on:** record each inserted phase's `Depends on` (TP-IDs):

- Rules 2, 5 insert multiple phases in stated sequence ("X first, then Y") → Y's `Depends on` = X.
- Rule 8 (Data migration, "isolated and late") → depends on whichever phase(s) Rules 1–7 inserted.
- Rules 1, 3, 4, 6, 7 (single-phase insertions) and Rules 9–10 (markers only) → `Depends on = —`, unless a playbook's `## Derivation` states otherwise.

**Sizing:** Size each phase independently (S / M / L / XL / XXL). If any phase estimates XL or XXL, decompose it into two or more dependent phases. The final phase list must contain only S / M / L phases.

## Pattern Selection

| Pattern | Use when |
| --- | --- |
| Strangler fig | `strangler_fig: mandated`; or incremental replacement needed without downtime |
| Lift-and-shift | Platform or infra change only — no logic rewrite |
| Replatform | Runtime or framework upgrade with partial rewrites |
| Decompose | Breaking apart a monolith — structural complexity phase |
| Rewrite | Small system with Changeability Red and no viable migration path |
| Test coverage | Test coverage phase inserted by Rules 5 or 7 — harness and coverage uplift; no deployment change |

If `sla_constraint: true`: mandate blue/green or canary deployment in every phase involving deployed component changes.
If `strangler_fig: mandated`: no big-bang decomposition in any phase.

## Complexity & Risk Criteria

| Field | Low | Medium | High |
| --- | --- | --- | --- |
| Complexity | Single component/module affected; pattern is Lift-and-shift or Test coverage; no data migration | 2–5 components affected, or partial rewrite (Replatform) | >5 components affected, cross-system coordination, or Decompose/Rewrite with data migration |
| Risk | Isolated, easily reversible per this phase's Rollback; no external consumers affected | Moderate blast radius or first application of this pattern in the codebase; some external consumers | `strangler_fig: mandated` or `sla_constraint: true` with tight RTO; data migration involved; broad blast radius or many external consumers |
