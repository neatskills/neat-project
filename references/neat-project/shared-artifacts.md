# Shared Artifacts

Canonical schemas for artifacts used across multiple neat-* skills.

**Skills referencing this file:**
- neat-delivery-planning
- neat-delivery-building
- neat-discovery-assessing
- neat-discovery-solutioning
- neat-discovery-modernizing

---

## issues.md

**File:** `specs/delivery/issues.md`

**Used by:** planning (writes), building (writes), assessing (writes), solutioning (writes)

**Purpose:** Append-only issue registry shared across solutioning, planning, and building.

### Base Schema

One JSON entry per issue, appended in order — not a markdown table.

```jsonc
{
  "ID": null,           // "ISS-NNN" — read existing entries, find highest number, increment by 1; "ISS-001" if absent
  "Type": null,         // "defect" (wrong or missing behavior) | "debt" (tech debt, coverage gap, pattern inconsistency) | "gap" (missing infrastructure, docs, convention)
  "Description": null,  // what's wrong
  "Blocker": null,      // true | false — see skill-specific rules for what constitutes a blocker
  "Status": null,       // "open" | "accepted" | "addressed"
  "Resolution": null,   // rationale text on accept (cite reference if Proven; explain reasoning if Inferred); "pending (REQ-NNN to be assigned)" then the assigned REQ-NNN on escalate (set by planning Phase 2 Step 2, then Phase 4 Step 1); "—" when open
  "Source": null        // provenance — "solution.md" | "modernization.md" | "codebase-scan" | "subagent-review" | "inline"; used by planning Phase 2 Step 1 for idempotency deduplication; omit for issues written by building
}
```

### Status Ownership

- **`open`**: planning (Phase 2 Step 1 seed; Phase 3 and Phase 5 Step 3 new `gap` entries), solutioning (codebase-analysis Step 8 — brownfield product track only), building (Phases 1–5)
- **`accepted`**: building (Phases 3, 4) only
- **`addressed`**: planning (Phase 4 Step 1) only

**Rules:**
- Never write `addressed` from building
- Never write `accepted` from planning
- Append-only: never delete entries
- Update `Status` and `Resolution` in place when decision is made

### Validation Rules

- `ID`: Must be unique, sequential ISS-NNN format
- `Type`: Must be one of: defect, debt, gap
- `Blocker`: Must be boolean (true | false)
- `Status`: Must be one of: open, accepted, addressed
- `Description`: Non-empty string
- `Resolution`: Required when Status=accepted or addressed; "—" when open

### Example

```jsonc
{
  "ID": "ISS-003",
  "Type": "debt",
  "Description": "Auth middleware stores session tokens in way that doesn't meet compliance requirements",
  "Blocker": true,
  "Status": "open",
  "Resolution": "—",
  "Source": "subagent-review"
}
```

---

## delivery.json

**File:** `specs/delivery/delivery.json`

**Used by:** planning (writes), building (reads + patches status)

**Purpose:** Workspace-level delivery state tracking.

### Schema

Format depends on `planning_model`:

```jsonc
{
  "planning_model": null,   // "phase-first" | "product-first" (naming kept for backwards compatibility, but now means shared vs. per-product paths)
  "product_col": null,      // true | false — true when wave tables carry a Product column (set in planning Phase 1 Step 1)
  "products": null,         // string array (shared paths, multi-product only) — omit for single-product
                             // OR object array (per-product paths) — each: { "name": null, "waves": [...] } — waves nested per product, "waves" below omitted at top level
  "waves": [
    {
      "wave": null,         // wave file slug (e.g. "wave01-booking", "wave02-payments") — zero-padded 2-digit
      "status": null        // "pending" | "planned" | "building" | "built" — "building" = claimed by a neat-delivery-building session
    }
  ]
}
```

For per-product paths, `waves` moves inside each entry of `products` (as shown above) — the top-level `waves` field is omitted in that case.

### Status Transitions

- **`pending`**: No plan exists yet (planning hasn't processed this wave)
- **`planned`**: Plan complete, ready to build (planning Phase 4 sets this)
- **`building`**: Claimed by building session, in progress (building Phase 1 Step 3 sets this)
- **`built`**: Implementation complete, tests pass (building Phase 8 sets this)

**Rules:**
- Planning writes: pending → planned
- Building writes: planned → building → built
- Building can revert building → planned if no features started (Phase 8 stop rule)

### Example (phase-first, single product)

```jsonc
{
  "planning_model": "phase-first",
  "product_col": false,
  "waves": [
    { "wave": "wave01-auth-ui", "status": "built" },
    { "wave": "wave02-payments", "status": "building" },
    { "wave": "wave03-admin", "status": "planned" }
  ]
}
```

---

## modules/{container-component}.md

**File:** `specs/system/modules/{container-component}.md`

**Used by:** solutioning (creates via codebase-analysis), building (updates source_files), planning (reads for context)

**Purpose:** Knowledge artifacts capturing module purpose, responsibilities, and integration patterns.

### Schema

One file per component within containers. Components are domain modules or architectural layers extracted from deployable units (containers) for progressive disclosure.

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

### Notes

- **External contracts** (HTTP endpoints, WebSocket, webhooks, external APIs) are NOT in module files — they belong in `system.md § Contracts` (Inbound/Outbound) as the single source of truth for system boundary integration.
- **Internal events** (`events_emitted`, `events_consumed`) stay in module files — these are module-to-module integration via internal event bus, not external contracts.
- **External libraries** (`external_imports`) are just package names/versions in modules — actual external API contracts (Stripe API, GitHub API) belong in `system.md § Contracts / Outbound Contracts`.

---

## modules/index.md

**File:** `specs/system/modules/index.md`

**Used by:** solutioning (creates), planning (reads for module loading), building (reads for module loading)

**Purpose:** Lightweight lookup table over all component and shared-package files.

### Schema

Rendered as a markdown table under a `# Module Index` header — one row per component or shared package (expect 10-50+ rows for multi-container projects):

```jsonc
{
  "Name": null,         // component or shared package name (e.g., "backend-projects")
  "Path": null,         // component's file or directory path
  "Type": null,         // api | ui | worker | cli | service | other (inherited from container)
  "Key Exports": null,  // 3–5 most significant export signatures, comma-separated — enough for the user to recognise relevance; format: "name(params): returnType" for functions, "ClassName" for classes
  "Internal Imports": null    // internal_imports module names, comma-separated
}
```

### Rules

- Written by codebase-analysis (Step 4) alongside the full component module files
- Updated incrementally on re-scan (only rows for re-scanned components are replaced)
- Solutioning writes it when expanding the modernization track Module Map into individual module files
- Building never creates module files or index rows

---

## modules/shared-modules.md

**File:** `specs/system/modules/shared-modules.md`

**Used by:** solutioning (creates), planning (reads), building (reads, never modifies)

**Purpose:** Multiple entries — one per shared package — using the same schema as `modules/{container-component}.md` with `type: shared`.

### Rules

- `internal_imports` may only reference other shared modules, never product modules

---

## See Also

- [recommendation-format.md](recommendation-format.md) - Resolution field guidance for accepting issues
