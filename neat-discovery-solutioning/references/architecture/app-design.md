# Application Architecture Design

**Include when:** standard features, workflows, or CRUD requirements exist.

**Inputs:** prd.md constraints (Current State — existing systems, access constraints); agent architecture output (if present, for UX section).

## Decisions

For XL/XXL stories, note the architectural implication for each. State pattern/category (decided) vs library/version (deferred). Flag lock-in risks.

| Layer | Decide | Defer to planning |
| --- | --- | --- |
| **Frontend** | SPA vs server-rendered; component organization; state management; responsive/accessibility | Framework, UI library |
| **Backend** | API-first; monolith vs microservices; protocol (REST/GraphQL/gRPC); background jobs; caching strategy | Language/framework |
| **Database** | Storage type with rationale; entities/relationships; migration strategy | Table definitions, full schema |
| **Auth** | SSO integration; role-based access (roles→permissions); session vs token | Protocols (SAML vs OIDC), token formats |

**Deferred requirements:** when a deferred requirement has a zero-cost substrate decision at current scope (e.g. SSR vs SPA, event hooks, string externalization), lock in the substrate now — keeps later promotion additive rather than a rearchitecture.

## Data

**Storage type** — this section owns the authoritative decision, unless [infrastructure.md](infrastructure.md) Path A (managed platform ecosystem) applies — there, storage is already constrained by the platform (see infrastructure.md § Cross-Cutting Constraints for Other Documents); refine within those constraints instead. The Database row above states the category; refine it here.

| Type | Use when |
| --- | --- |
| **File-based** | Small, stable, human-readable state |
| **Relational DB** | Structured queries, ACID guarantees, relational data |
| **Document store** | Flexible schemas, hierarchical data |
| **Vector store** | Embedding storage and similarity search; required when agent uses RAG |
| **Object storage** | Binary assets, large files, archival |

State decision with rationale. Document migration path if likely to change.

**Data model:** core entities and relationships; schema approach; entity-to-storage mapping; read/write patterns per entity.

**Data pipeline (if needed):** sources, sinks, transformation steps, trigger (event vs schedule), error handling. If none, state why.

**Data lifecycle:** retention and deletion policy (right to deletion, cascading deletes); data leaving the system (flag for disclosure and compliance); residency if mandated (region + whether each storage layer satisfies it); encryption at rest (flag storage types that don't encrypt by default); backup approach and integrity guarantees.

## UX

**Note:** User journeys and layout wireframes are produced by `neat-discovery-assessing` and appear in PRD § User Journeys and § Layout Design when frontend work is identified. This section provides **architectural reference** for UX decisions that affect system design: streaming vs polling, caching strategy, offline support, AI interaction patterns, accessibility compliance level, and design system choice. Reference the PRD UX sections when making these architectural decisions.

*Include when ANY apply: multi-turn conversational interface; custom design system; offline-first or PWA; device-specific interaction constraints; accessibility compliance; complex multi-step guided workflow; AI agent present (for Human-AI Interaction Pattern, even if the rest of the UI is standard CRUD). Skip for standard CRUD/admin UI with no agent.*

**Interaction Model:**

| Model | Use when | Key concerns |
| --- | --- | --- |
| **Conversational** | Natural-language intent | Turn management, context persistence, clarification |
| **Form-driven** | Structured data entry, deterministic outcomes | Field grouping, validation UX, progressive disclosure |
| **Dashboard/read** | Viewing data primary; actions secondary | Information hierarchy, filtering, data density |
| **Workflow/wizard** | Multi-step, branching, persistent state | Step navigation, partial save, branching logic |
| **Hybrid** | Multiple models across features | Dominant model; mode transitions |

State the model with one-line rationale. List 3–5 critical user journeys as text flows (*Actor → action → system response → outcome*).

**Human-AI Interaction Pattern** *(only if agent present):*

| Pattern | Description | Key concerns |
| --- | --- | --- |
| **Chat** | User messages; AI responds in thread | Streaming or polling (sync request/response — simpler; no typing-indicator UX), history, context window |
| **Inline suggestion** | AI proposes content; user accepts/rejects/edits | Latency, confidence signalling, undo |
| **Batch review** | AI processes bulk; user reviews results | Progress feedback, batch size, error rate |
| **Autonomous background** | AI acts without prompt; surfaces results async | Notification, explainability, override |

State pattern and orchestration implication. How does the system signal AI confidence? What can users override? Are AI outputs clearly labelled?

**Design System:**

| Option | Choose when | Risk |
| --- | --- | --- |
| **Adopt existing** | Speed matters; no brand differentiation | Component constraints may require workarounds |
| **Extend existing** | Brand customisation; core components reused | Upgrade complexity; upstream divergence |
| **Build custom** | Strong brand identity; unique patterns; library is a product | High cost; maintenance burden |

State decision with rationale. Defer component library version to planning.

**Device & Platform Targets:** state targets and minimum versions explicitly — silence is not "all platforms." Web: desktop-only / desktop-first / mobile-first / PWA. Mobile: native iOS / Android / cross-platform / web-only. Offline: required or not — if required, state what works offline and what syncs. Localization: state languages/regions; flag RTL, date/number formatting, translated AI outputs.

**Accessibility:** state compliance level (WCAG 2.1 AA/AAA, Section 508, EN 301 549) and whether contractual or aspirational. Flag interaction patterns requiring specific treatment.

**Defer to planning:** screen layouts · navigation structure · error states · component library · ARIA patterns · color palette and contrast

## Performance & Scalability

- **Caching strategy:** decide what gets cached and at which layer (in-memory, distributed, CDN boundary) — defer cache technology to planning
- **Rate limiting:** decide whether API endpoints need rate limiting and the enforcement strategy (per-user, per-IP, per-plan)
- **Asset optimization:** decide bundling, lazy loading, and compression approach — defer tooling to planning
- **Performance budget:** define target thresholds for load, interaction, and layout stability; flag high-risk patterns (real-time updates, large tables, file uploads)
- **Load ceiling:** state what the current design supports and the first scaling trigger — infra owns the authoritative answer; flag here if a requirement drives an unusual ceiling

**Defer to planning:** cache technology · CDN config · horizontal scaling config · specific threshold values

## Observability

- **Application logging:** identify key events to log at the application layer (errors, state transitions, user actions); define structured log format
- **Error boundaries:** where errors are caught, how they surface to users, and what gets reported upstream
- **Client-side signals:** what user-facing performance and error signals need capturing — defer tooling to infra

**Defer to planning:** log aggregation platform · alerting thresholds · monitoring dashboards (owned by infra)

## Security & Compliance

- **Encryption in transit:** HTTPS everywhere; flag internal service-to-service channels needing mTLS
- **Input validation:** identify boundaries requiring validation (API endpoints, file uploads)
- **Compliance obligations:** identify applicable regulations (GDPR, HIPAA, SOC 2, etc.); flag data collection, consent, and right-to-deletion requirements; state audit trail needs

Secrets management and encryption at rest are owned by infra.
