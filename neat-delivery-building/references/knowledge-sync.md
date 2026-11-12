# Knowledge Synchronization

Updates system.md after wave completion, and rescans module files when the user asks.

## Step 1 — Module Rescan (On Request)

- User asked for a module rescan and `specs/system/modules/` exists → run a full rescan: load and follow `../../references/neat-project/codebase-analysis.md` Steps 1–9, marking all modules for re-scan.
- Otherwise → skip to Step 2. Without a modules directory (greenfield), Iteration-mode solutioning creates the modules.

**Rescan scope (delta updates):**

- Updates module files with new knowledge: `purpose`, `responsibilities`, `exports`, `internal_imports`, `external_imports`, `events_emitted`, `events_consumed`, `constraints`, `extension_points`, `usage_patterns`
- Merges new data into existing components (does not replace entire files)
- Updates `modules/index.md` (Key Exports and Internal Imports columns)
- Updates `system.md § Conventions / § Patterns` (codebase-analysis Step 6)
- Writes issues.md entries for technical debt and risks (codebase-analysis Step 8)
- Appends new contracts to `system.md § Contracts / Inbound Contracts` and `§ Contracts / Outbound Contracts` (from codebase-analysis Step 5)

---

## Step 2 — system.md Updates

Re-read `specs/system/system.md` only if Phase 5 already ran this session (mid-session resume). Otherwise use Phase 1 loaded version.

**Analysis inputs:**

- `touched_files` (all features, from Phase 3 union)
- `specs/discovery/solution.md` (on-demand load per Phase 1 Step 4)
- Module files (refreshed in Step 1 if rescan ran)

**Update rules per section:**

### § Foundation

**When to update:** New runtime component added (database client, cache, queue, observability sink, secrets manager).

**How to detect:**

- New package.json dependency matching category (e.g., `redis`, `pg`, `@aws-sdk/client-dynamodb`)
- New configuration file for observability/logging (`logger.ts`, `telemetry.config.js`)
- New infrastructure marker in code (grep for `Infra:`, `Delivery:`, `Environments:` in comments or config)

**How to update:**

- **Component added (new category):** Append one line: `{Category}: {tool/library}` (e.g., `Cache: Redis`)
- **Component replaced (same category, different tool):** Update existing line in place (e.g., `Database: PostgreSQL` → `Database: PostgreSQL + TimescaleDB`)
- **Contradiction (both tools present for same category):** Write `gap` to issues.md; keep existing line unchanged

### § Architecture

**When to update:** Architectural pattern introduced, evolved, or changed.

**How to detect:**

- New middleware/plugin architecture file
- New event bus/queue integration pattern
- State management pattern file
- Authentication/authorization strategy file

**How to update:**

- **Pattern added (new architectural concern):** Append one sentence describing the pattern
- **Pattern evolved (same intent, refined implementation):** Update existing line in place with refined description
- **Pattern contradicts solution.md or existing pattern:** Write `gap` to issues.md with old vs. new; do not edit in place

**Distinction:**

- **Evolution**: JWT auth using `jsonwebtoken` → JWT auth using `jose` (same pattern, different library)
- **Contradiction**: JWT auth → OAuth 2.0 (different pattern, changes integration contracts)

### § Contracts

**When to update:** New HTTP route, GraphQL field, WebSocket event, or external integration added.

**How to detect:**

- Grep `touched_files` for route definitions (`app.get`, `@Get`, `router.post`, `@route`)
- GraphQL schema file changes (`*.graphql`, `schema.ts`)
- WebSocket event handler registrations
- External API client instantiations

**How to update:**

- **New contract:** Append one line: `{Method} {Path} — {purpose}` for REST; `{Type}.{field} — {purpose}` for GraphQL
- **Removed contract (route/handler deleted):** Write `gap` to issues.md ("Endpoint {method} {path} removed — verify intentional deprecation"); do not remove from system.md

**Rationale:** Removals might be intentional API deprecation (should be documented) or accidental (should be caught). Either way, planning needs to see the gap.

### § Business Rules

**When to update:** Validation logic, invariant checks, or domain constraints implemented.

**How to detect:**

- New validator functions (`validate*`, `check*`, `ensure*`)
- Invariant assertions in domain logic
- Guard clauses with business conditions

**How to update:** Append one sentence per rule: `{Entity} must {constraint}`

**Example:** `Order total must equal sum of line items + tax`

### § Domain Concepts

**When to update:** New entity type or significant entity field added.

**How to detect:**

- New database migration file adding table
- New GraphQL type or OpenAPI schema entity
- New domain model class/interface

**How to update:** Append one sentence per concept: `{Entity}: {definition}`

**Example:** `Subscription: recurring billing arrangement with renewal date and cancellation policy`

### § Edge Cases

**When to update:** Edge case handling added (boundary conditions, error states).

**How to detect:**

- Test files in `touched_files` with edge case descriptions (test names like "handles empty cart", "rejects negative quantity")
- Error handling for unusual states (grep for `if (!`, `throw new`, error constructors)

**How to update:** Append one sentence per edge case handled

**Example:** `Empty cart checkout blocked with user-friendly message`

---

## Step 3 — Divergence and Gap Reporting

**Check for divergences:**

- New component contradicts existing system.md content → write `gap` to `specs/delivery/issues.md`
- Architectural pattern contradicts solution.md or existing pattern (not evolution) → write `gap`: "system.md § Architecture diverged from solution.md — verify {pattern} change aligns with architecture decisions"
- New shared service created → write `gap`: "New shared service `{name}` created in `{path}` — run solutioning to update `modules/shared-modules.md`"

**Gap format:** Follow [../../references/neat-project/shared-artifacts.md](../../references/neat-project/shared-artifacts.md) § issues.md schema. Set `Blocker: false`, `Type: gap`.

---

## Step 4 — Greenfield Transition

If `system.md` contains `mode: greenfield`:

- Update to `mode: brownfield` after all writes complete
- First wave marks the transition — subsequent waves proceed as brownfield

**Note:** Modules are NOT created during building. Iteration-mode solutioning detects `mode: brownfield` + no modules and runs full codebase-analysis to create them.

**Rationale:** Greenfield projects have no codebase at Initial-mode solutioning (skip codebase-analysis). First building wave produces initial code, transitioning to brownfield. Iteration-mode solutioning creates modules from the now-existing codebase.
