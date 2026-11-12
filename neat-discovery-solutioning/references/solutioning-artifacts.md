# Solution Artifacts

## solution.md

Frontmatter:

```yaml
---
status: draft  # Set to "final" when quality checks complete
reflects_requirements: []  # List of NN values from incorporated requirementsNN.md files (e.g., [01, 02, 03]); empty for Initial mode (only prd.md), populated incrementally in Iteration mode
---
```

Content schema:

```jsonc
{
  "## Architecture": null,          // 1–2 sentence section intro; contains the subsections below. Ref architecture/app-design.md; also architecture/infrastructure.md (Path A) if platform ecosystem (governs all later sections); also architecture/agent-design.md (Agent Design, orchestration patterns) if AI/agents.
  "### System Landscape": null,     // (optional) Supplementary — Mermaid diagram: all owned systems in context. Produce when PRD Requirements section references 2+ distinct owned systems each with their own deployment boundary.
  "### System Context": null,       // (optional) C4 L1 — Mermaid diagram: system as one box, external actors and systems around it. Container Diagram implicitly covers L1 for most projects; add this subsection only when the audience is non-technical stakeholders or the system has so many external dependencies that L2 is too dense as an entry point. Precedes Container Diagram when present.
  "### Container Diagram": null,    // C4 L2 — data-flow view: Mermaid diagram showing deployable units (containers), external actors, and communication paths between them. Risk-annotated per Technical Risks table. Followed by inline "Owned containers:" list identifying which containers receive L3 Component Diagrams.
  "### Deployment Diagram": null,   // C4 Deployment Diagram — infrastructure-layer view: cloud-native (infrastructure.md Path B) only. graph LR Mermaid diagram — named managed services grouped by layer (Networking/Edge, Compute, Data, Messaging, Security/Auth); directional arrows labeled with protocol; risk-annotated per Technical Risks table; ref architecture/infrastructure.md for diagram conventions. For PaaS, local-first, and managed platform (Path A): insert <!-- Deployment Diagram: not applicable — {reason} --> instead.
  "### Component Diagrams": null,   // C4 L3 grouping header — one subsection per owned container (those listed in the "Owned containers:" list below Container Diagram). If no components can be determined for a container, insert <!-- Component Diagram: not enough information to determine components for {Container Name} --> and skip that subsection.
  "#### {Container Name}": null,    // C4 L3 — internal components of this container: Mermaid diagram showing components and their relationships. Risk-annotated per Technical Risks table.
  "### Code Diagrams": null,        // C4 L4 grouping header — only when codebase_analysis is not skipped AND at least one module has scan_status: ok. Omit entirely otherwise.
  "#### {Component Name}": null,    // C4 L4 — key classes/modules (2–3) for this architecturally significant component (cross-cutting concerns, integration boundaries, core domain logic). Not risk-annotated.
  "## Code Structure": null,        // comprehensive code structure design; ref architecture/code-structure.md; contains subsections below
  "### Repository Strategy": null,  // Decision (monorepo|multi-repo|single-repo), Rationale (cite team structure, product count, deployment model, scale); IF multi-repo: Repository table (Repository | Purpose | Owner Team | Tech Stack) + Shared Code Strategy; IF monorepo: Workspace Structure list (workspaces/packages with purpose) + Monorepo Tool + Rationale
  "### Organization Pattern": null, // Decision (layer-based|feature-based|hybrid), Rationale (cite domain complexity, scale, tech stack conventions); IF hybrid: Hybrid Strategy (which parts use layers, which use features, and why)
  "### Module Boundaries": null,    // IF monorepo: table Package/Module | Purpose | Depends On; IF feature-based: table Feature Area | Requirements | Components; IF layer-based: table Layer | Responsibility | Key Components
  "### Folder Structure": null,     // Concrete directory tree (top 3-4 levels); IF multi-repo: subsection per repo (#### Repository: {name}); variables replaced with actual names
  "### Conventions": null,          // Tests, Configs, Shared code, Generated code (if applicable), Public API (libraries only)
  "### Component-to-Folder Mapping": null, // Table per owned container: Component | Folder Path | Notes
  "### Migration Path": null,       // (brownfield only, if codebase_analysis not skipped) Current Structure (from codebase-analysis findings), Target Structure, Migration Strategy (gradual|restructure-upfront), Rationale, Phases (numbered steps)
  "## Tech Stack": null,            // table: Layer | Technology | Rationale; highlight build-vs-adopt decisions; ref architecture/app-design.md, architecture/agent-design.md (if AI/agents), architecture/infrastructure.md (hosting/observability tooling), architecture/integration.md (if present, e.g. event bus/queue choice)
  "## Flows": null,                 // one subsection per Phase 2 flow: Mermaid diagram + key decision points and branches
  "## Data Model": null,            // Mermaid erDiagram — entities, fields, and relationships — regardless of storage type (applies to non-relational and non-DB storage too); + table: Entity | Key fields | Storage | Notes; ref architecture/app-design.md (Data section); also architecture/agent-design.md (Knowledge Design section) if agent present
  "## Integrations": null,          // (if present) table: Integration | Protocol | Auth | Data contract; ref architecture/integration.md
  "## Operational Concerns": null,  // Deployment, Observability, Security, Performance & Scalability, Rollout subsections; ref architecture/infrastructure.md (all subtopics); also architecture/app-design.md (Security & Compliance section) and architecture/agent-design.md (Security & Compliance section, if agent present) for Security
  "## Technical Risks": null,       // top risk (1 sentence) + table: Risk | Likelihood | Impact | Mitigation; rows sorted High → Med → Low
  "## UX Design": null,             // (if present) ref architecture/app-design.md (UX section); include only when UX complexity warrants
  // Open Questions: embed inline within the section they belong to (e.g., under Tech Stack for an unresolved language choice) — not as a trailing section. Format as a clearly labelled callout block (plain bullets: question | owner | target date) immediately after the section's content. Omit if none remain unresolved.
}
```

## modules/{container-component}.md (`specs/system/modules/{container-component}.md`)

One file per component within containers. Components are domain modules or architectural layers extracted from deployable units (containers) for progressive disclosure. Written by codebase analysis as **knowledge artifacts** capturing purpose, responsibilities, and integration patterns.

```jsonc
{
  "name": "",              // component name: "{container}-{component}" (e.g., "backend-projects")
  "container": "",         // parent container name (e.g., "backend") — links component to deployment unit
  "type": "",              // api | ui | worker | cli | service | other (inherited from container)
  "path": "",              // component's file or directory path (e.g., "app/backend/projects.py")
  "entry_point": "",       // primary file for this component
  "scan_status": "",       // ok | failed
  
  "source_files": [],      // All source files belonging to this module (initialized by codebase-analysis, updated by building when new files touched)
  
  "purpose": "",           // WHY this module exists — 1-2 sentences describing core responsibility and business value
  
  "responsibilities": [],  // WHAT it does — 3-5 key behaviors/capabilities in plain language
  
  "exports": [             // Key exported functions/classes with semantic information (5-10 max)
    {
      "signature": "name(params): returnType",  // Function/method signature or "ClassName" for classes
      "purpose": "",        // What this export does (one sentence)
      "side_effects": "",   // State changes, I/O, spawned tasks (omit if pure function)
      "returns": ""         // What it returns (omit if obvious from signature)
    }
  ],
  
  "internal_imports": [    // Modules from this system
    {
      "module": "{container}-{component}",  // Module name
      "why": ""             // What we use from this module
    }
  ],
  
  "external_imports": [    // Third-party packages
    {
      "package": "{name}@{version}",  // Package name and version
      "why": ""             // What we use this library for
    }
  ],
  
  "events_emitted": [],    // Internal events published to event bus (e.g., "step.completed", "user.created")
  "events_consumed": [],   // Internal events subscribed from event bus (e.g., "payment.processed")
  
  "constraints": [],       // Invariants, business rules, limitations (e.g., "Only one active step per project")
  
  "extension_points": [],  // WHERE/HOW to add new functionality (specific guidance for common changes)
  
  "usage_patterns": ""     // HOW to use this module — typical usage patterns, initialization, common workflows
}
```

**Note**: 
- **External contracts** (HTTP endpoints, WebSocket, webhooks, external APIs) are NOT in module files — they belong in `system.md § Contracts` (Inbound/Outbound) as the single source of truth for system boundary integration.
- **Internal events** (`events_emitted`, `events_consumed`) stay in module files — these are module-to-module integration via internal event bus, not external contracts.
- **External libraries** (`external_imports`) are just package names/versions in modules — actual external API contracts (Stripe API, GitHub API) belong in `system.md § Contracts / Outbound Contracts`.

## modules/index.md (`specs/system/modules/index.md`)

Lightweight lookup table over all component and shared-package files. Written by codebase-analysis (Step 4) alongside the full component module files; updated incrementally on re-scan (only rows for re-scanned components are replaced). Solutioning writes it when expanding the modernization track Module Map into individual module files. Building never creates module files or index rows.

Skills load this file to surface candidate components to the user before loading full component module files. Rendered as a markdown table under a `# Module Index` header — one row per component or shared package (expect 10-50+ rows for multi-container projects):

```jsonc
{
  "Name": null,         // component or shared package name (e.g., "backend-projects")
  "Path": null,         // component's file or directory path
  "Type": null,         // api | ui | worker | cli | service | other (inherited from container)
  "Key Exports": null,  // 3–5 most significant export signatures, comma-separated — enough for the user to recognise relevance; format: "name(params): returnType" for functions, "ClassName" for classes
  "Internal Imports": null    // internal_imports module names, comma-separated
}
```

## modules/shared-modules.md (`specs/system/modules/shared-modules.md`)

Multiple entries — one per shared package — using the same schema above with `type: shared`. `internal_imports` may only reference other shared modules, never product modules.

## system.md (`specs/system/system.md`)

Freeform markdown. Initialized by solutioning; updated in place by building. Planning reads only — never writes.

**Mode line:** Opens with `mode: greenfield | brownfield`.

**Sections (in order):**

1. `## Foundation` — Runtime, frameworks, dependencies, infrastructure markers (Infra, Delivery, Environments), observability, secrets, testing
2. `## Code Structure` — File tree + placement rules (extracted from solution.md § Code Structure)
3. `## Conventions` — Contains subsections:
   - `### Test Framework` — Testing setup and requirements
   - `### Patterns` — Code patterns and conventions
4. `## Architecture` — Component relationships and patterns
5. `## Contracts` — External system boundary. Contains subsections:
   - `### Inbound Contracts` — HTTP API endpoints, WebSocket streams, webhooks we receive (what external clients can call)
   - `### Outbound Contracts` — External APIs we consume, webhooks we send (what external services we integrate with)
6. `## Business Rules` — Invariants, constraints, domain policies
7. `## Domain Concepts` — Entities and their definitions
8. `## Edge Cases` — Boundary conditions and exceptional scenarios

**Section ownership:**

| Section | Initializer | May update | Read-only for |
| --- | --- | --- | --- |
| `## Foundation` | solutioning | codebase-analysis Step 1 (topology on initialization); building (append new components; update in place when component replaced); solutioning (Iteration operational tooling rescan — overwrite per-category lines) | planning |
| `## Code Structure` | solutioning | never | all |
| `## Conventions / ### Test Framework` | solutioning Phase 4 Step 4 | never | all |
| `## Conventions / ### Patterns` | codebase-analysis Step 6 | codebase rescan (overwrite this subsection only) | all |
| `## Architecture` | solutioning | building (append new patterns; update in place when pattern evolved; gap when contradicts solution.md) | planning |
| `## Contracts` | solutioning | building (append new endpoints/events; gap when removed) | planning |
| `## Business Rules` | solutioning | building (append) | planning |
| `## Domain Concepts` | solutioning | building (append) | planning |
| `## Edge Cases` | solutioning | building (append) | planning |

**Writing rules for building (Phase 5):**

- Always read before editing.
- Per update: one sentence per new component, interface, or concept. No explanatory prose.
- Never write to `## Code Structure`, `## Conventions`, or any subsection under Conventions.
- If a touched component contradicts existing content → write to `specs/delivery/issues.md` as a `gap`; do not edit the file in place.

## Shared Artifacts

### issues.md (`specs/delivery/issues.md`)

Written by solutioning (codebase analysis) and building (subagent review). Schema defined in `../../references/neat-project/shared-artifacts.md`. Append new entries; never delete entries. Update `Status` and `Resolution` in place when a decision is made.
