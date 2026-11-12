# Codebase Scan

## Initialization

If `specs/system/` does not exist:

- Create `specs/system/` directory
- Create `specs/system/modules/` directory
- Create `specs/system/system.md` with the `mode:` line and the section headings from [../../neat-discovery-solutioning/references/solutioning-artifacts.md](../../neat-discovery-solutioning/references/solutioning-artifacts.md) § system.md — no frontmatter (solutioning reads the `mode:` line as the first line)

## Outputs

```text
specs/system/
  system.md           # enriched: § Foundation (topology, Step 1; operational tooling, Step 9), § Contracts (Step 5), § Conventions / ### Patterns (Step 6), § Business Rules + § Domain Concepts (Step 7)
  modules/
    shared-modules.md                # shared libraries (no runtime)
    {container}-{component}.md       # one per component within containers
    index.md                         # lightweight lookup table
```

**Component granularity:** Modules are extracted at component level within each container (deployable unit). A container like `backend` is decomposed into domain modules (e.g., `backend-projects`, `backend-materials`, `backend-pipeline`) for progressive disclosure. Containers are detected first for topology, then components within each container are scanned.

**Step 1 — Topology Detection:**

Map the repo layout from directory structure and manifests. Classify the project:

- **Monorepo:** top-level `packages/`, `apps/`, `services/`, `libs/` directories containing multiple packages. Classify each package:
  - `product` — has its own entry point and can run/deploy independently
  - `shared` — library imported by products; no own runtime
  - `infra` — infrastructure-as-code (Terraform, CDK, Helm)
  - `devops` — CI/CD pipelines, scripts, tooling

- **Single-product:** single entry point at root or one top-level package. Classify as one `product` unit.

For `infra` and `devops`: flag their presence in `system.md § Foundation`; do not scan their internals.

Produce a list: `[{name, path, type}]` — used in all subsequent steps.

**Write topology to system.md:**

Update `system.md § Foundation` with topology information:

- **Monorepo:** `Topology: monorepo ({N} products, {M} shared libs)`
- **Single-product:** `Topology: single product`

For infra/devops presence, append on new lines in § Foundation:

- `Infra: {tool} at {path}` (e.g. "Infra: Terraform at infrastructure/")
- `DevOps: {tool} at {path}` (e.g. "DevOps: GitHub Actions at .github/workflows/")

**Step 2 — Contradiction Check:**

Before spawning any subagent, compare the PRD's requirements, tech stack, and constraints against the topology list and manifests. On an Iteration run, also compare the Active sections of the `requirementsNN.md` files being incorporated — the artifacts actually driving this run — against the same topology list and manifests:

- Does the required tech exist in the manifests?
- Does the required pattern exist in the codebase structure?
- Does the PRD (or, on Iteration, the new requirements) assume greenfield behavior for a clearly established codebase pattern?

Surface all contradictions to the user — the user must review and update the PRD or `requirementsNN.md` to resolve them before solutioning can continue. Stop here until resolved.

**Step 3 — Staleness Check:**

Determine which containers and components need re-scanning:

- `specs/system/modules/` absent → full scan (proceed to Step 4 for all containers).
- `specs/system/modules/` present, no `build-history.md` → skip scan entirely (nothing has changed). Proceed to Step 6; Steps 5, 7 and 8 are then no-ops (they depend on files read in Step 4, which did not run) — skip them too.
- `specs/system/modules/` present, build history check: if `specs/delivery/delivery.json` exists, derive the build-history path from its `planning_model` key — `phase-first`: `./specs/delivery/build-history.md`; `product-first`: `./specs/delivery/{product}/build-history.md`. If `delivery.json` is absent, check the phase-first default path (`./specs/delivery/build-history.md`). If the resolved path exists: read **all entries**; union all `touched_files` across every entry; for each component module file in `specs/system/modules/`: read its `path` field; if any file in the touched_files union starts with that component's path → mark the component's parent container for re-scan. Containers with no touched components → skip (reuse existing files). If zero containers end up marked for re-scan, Steps 4/5/7/8 are no-ops — same as the no-build-history branch above.

User can request a full rescan by saying "rescan" — mark all containers for re-scan regardless of staleness.

**Step 4 — Module Scan (Component-Level):**

For each container (deployable unit) marked for re-scan, detect internal components and spawn at most 5 subagents per container. Process containers sequentially.

**Sub-step 1 — Universal Component Detection:**

Detect components using language-agnostic patterns based on directory structure, file size, and naming conventions.

**Discover structure:**
1. List all subdirectories at depth 1-2: `find {container-path} -maxdepth 2 -type d`
2. List all source files (exclude build artifacts: `.git/`, `node_modules/`, `__pycache__/`, `target/`, `build/`, `dist/`, `.mypy_cache/`)
3. Detect source file extensions automatically: `.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`, `.rb`, `.java`, `.cs`, `.rs`, `.php`, `.kt`, `.scala`, `.swift`, `.c`, `.cpp`, `.h`

**Identify component boundaries:**

1. **Directory components:**
   - Any subdirectory with 3+ source files → component
   - Any subdirectory containing a manifest file → component
     - Manifests: `package.json`, `__init__.py`, `go.mod`, `Cargo.toml`, `Gemfile`, `pom.xml`, `*.csproj`, `composer.json`
   
2. **File components:**
   - Top-level source files > 300 lines → standalone component
   
3. **Prefix groups:**
   - Top-level files with common naming prefix (2+ files) → group into component
   - Example: `user_service.rb` + `user_repository.rb` → `{container}-user`

**Recognize architectural patterns:**

Map common directory names to component types (universal across languages):
- `api/`, `routes/`, `controllers/`, `handlers/` → `{container}-api`
- `models/`, `entities/`, `domain/` → `{container}-domain`
- `services/`, `business/`, `core/`, `internal/` → `{container}-core`
- `adapters/`, `clients/`, `external/`, `pkg/` → `{container}-adapters`
- `infrastructure/`, `config/`, `db/` → `{container}-infrastructure`

**Apply merge rules:**

- `test/`, `tests/`, `spec/`, `__tests__/`, `*_test/` → include in parent component's source_files, don't create separate component
- `utils/`, `helpers/`, `common/` with < 5 files → merge into nearest component
- Files < 100 lines with generic names (`utils.*`, `helpers.*`, `constants.*`, `types.*`) → merge into parent
- `config/`, `constants/` → merge into infrastructure component

**Sub-step 2 — Intelligent Grouping:**

If detected components > 15 for a container:

1. Use reasoning to cluster into 8-12 logical groups based on:
   - **Domain cohesion**: Files sharing domain concepts (e.g., `projects` + `plan_page` → `{container}-project-management`)
   - **Layer/architectural role**: Same abstraction level (e.g., all API routes → `{container}-api`)
   - **Import cohesion**: Files that frequently import each other
   - **Naming patterns**: Common prefixes/suffixes (e.g., `user_service` + `user_repository` → `{container}-user`)
   - **Directory boundaries**: Respect package/module directories

2. Anti-patterns (don't group):
   - Infrastructure + domain logic
   - API routes + business logic
   - Core types + adapters

3. Target: 8-12 component modules per container

**Sub-step 3 — Subagent Batching:**

1. Divide components into at most 5 batches
2. Spawn one subagent per batch (max 5 per container)
3. Each subagent receives 1-N components to scan
4. Subagent iterates sequentially through its batch
5. Returns array of component module data

**Subagent prompt (batched component scan):**

```markdown
You are scanning {N} components in container "{container-name}".

**Container type**: {container-type}
**Base path**: {container-path}
**Components to scan**: {component-list}

For each component, extract knowledge (not just inventory):

1. Read component file(s) + one level of imports
2. Extract metadata:
   - name, container, type, path, entry_point, scan_status
   - **source_files**: List of files read (component file + imports within same container)
3. Extract knowledge:
   - **purpose**: Why this exists (1-2 sentences, infer from code/exports)
   - **responsibilities**: What it does (3-5 key behaviors)
   - **exports**: Structured with purpose/side-effects/returns (5-10 max)
   - **internal_imports**: Modules from this system with "why" for each
   - **external_imports**: Third-party packages with "why" (what we use them for)
   - **events_emitted**: Internal events published to event bus (module-to-module only, not external WebSocket/HTTP)
   - **events_consumed**: Internal events subscribed from event bus
   - **constraints**: Business rules, invariants
   - **extension_points**: HOW to extend (specific guidance)
   - **usage_patterns**: How to use (initialization, workflows)

Do NOT extract external contracts (HTTP endpoints, WebSocket, external APIs) - those go to system.md § Contracts / Inbound or Outbound separately.

Return as JSON array: one object per component, matching the component module schema in [shared-artifacts.md](shared-artifacts.md) § modules/{container-component}.md.

Iterate through components sequentially. If one component scan fails, record scan_status: "failed" and continue to the next.
```

**Shared layer scan:**

If any shared package is marked for re-scan or `modules/shared-modules.md` is absent, scan shared packages (no component-level decomposition for shared libraries). Spawn one subagent for all shared packages.

**On subagent failure:** write `scan_status: failed` to component module file, continue. Surface all failures at end of Step 4.

**Shared→product violation:** surface to user and ask: annotate and continue, or stop to fix. Do not proceed until resolved.

**Write results to `specs/system/modules/`:**

- One `{container}-{component}.md` per component
- One `shared-modules.md` for all shared packages
- Follow schema in [shared-artifacts.md](shared-artifacts.md) § modules/{container-component}.md (with added `container` field)

**Write/update `specs/system/modules/index.md`:**

- One row per component (expect 10-50+ rows for multi-container projects)
- Key Exports: pick 3-5 exports most likely to appear in requirement keywords
- If `index.md` exists: replace only rows for re-scanned components
- If absent: create fresh from all component files
- Follow schema in [shared-artifacts.md](shared-artifacts.md) § modules/index.md

Compare each module file and `index.md` against its schema — confirm all required keys are present.

Retain all `source_files` in-session for Steps 5, 7 and 8.

**Step 5 — Contract Extraction:**

Extract external-facing contracts from component modules and populate `system.md § Contracts`.

**What are contracts:**
- **API endpoints**: REST routes, GraphQL operations, gRPC methods (from `type=api` modules' in-session `source_files`)
- **WebSocket events**: Event handlers, message types
- **Message queue topics**: Pub/sub topics, queue names (from code inspection)
- **External integrations**: Outbound API clients, webhook receivers

**What are NOT contracts** (exclude):
- Internal interfaces/types (TypeScript interfaces, Go interfaces, Java interfaces)
- Internal function signatures between components
- Private methods or helper functions

**Extraction process:**

1. **Aggregate API endpoints from modules:**
   - Read all `{container}-*.md` files
   - For each module where `type=api`: grep its in-session `source_files` (Step 4) for route definitions
     - Extract endpoint list
     - Group by container

2. **Detect event-based contracts** (optional - if detectible from code):
   - WebSocket event registrations: grep source_files for `on(`, `socket.on`, `ws.on`
   - Message queue topics: grep for `publish(`, `subscribe(`, `.topic(`, `.queue(`
   - If detection is unreliable or no patterns found, skip (building will add these)

3. **Write to `system.md § Contracts`:**
   - Read existing `system.md`
   - **If § Contracts is empty:** Populate from scratch
   - **If § Contracts has content:** Append new contracts not already listed
   - Format: `{Method} {Path} — {Purpose from module context}` for REST
   - Format: `{Type}.{field} — {Purpose}` for GraphQL
   - Format: `{event-name} — {Purpose}` for WebSocket/events

4. **Example output:**
   ```markdown
   ## Contracts
   
   **Backend API:**
   - POST /api/auth/login — User authentication
   - GET /api/projects — List all projects
   - POST /api/projects/{slug}/steps/{step}/start — Start pipeline step
   - GET /api/projects/{slug}/artifact — Fetch project artifact
   - WebSocket /api/projects/{slug}/terminal — Terminal I/O stream
   
   **Events:**
   - project.created — New project registered
   - step.completed — Pipeline step finished
   ```

**Note**: Building (knowledge-sync) will add delta contracts from new touched_files. This initial extraction provides the baseline.

**Step 6 — Convention Extraction:**

From `modules/shared-modules.md`: identify patterns that act as project-wide conventions — auth middleware, DB client wrappers, logger interface, error types, test helpers, config injection patterns.

Write to `system.md ## Conventions / ### Patterns` only — overwrite the content of this subsection each rescan. Do not read or modify `### Test Framework`; that subsection is owned by `neat-discovery-solutioning` Phase 4 Step 4 and must never be touched by the codebase scan.

**Step 7 — Domain Extraction:**

From entity types, schemas, service names, and workflow names found in each scanned module's `source_files` (retained from Step 4) and `exports`:

- Identify new roles, concepts, business rules, and workflows not yet present in `system.md` § Business Rules or § Domain Concepts.
- **Append only** — never remove or overwrite existing content in those sections. If a concept already exists, skip it.

Write additions to `system.md` § Business Rules (rules) and § Domain Concepts (roles, concepts, workflows).

**Step 8 — Quality Audit:**

Spawn a subagent with the prompt below. Fill in `{scan-files}` (the full set of files read in Step 4) and `{knowledge-path}` (`specs/system/system.md`). The test framework may not be recorded yet — Step 6 may not detect it, and solutioning Phase 4 Step 4, which guarantees it, runs after this step.

```markdown
You are a software quality auditor assessing an existing codebase. Read {scan-files} and § Conventions in {knowledge-path}, then identify issues across:

1. Technical debt — pattern inconsistencies, poor abstractions, deprecated dependencies
2. Test coverage — missing tests, untested paths, coverage gaps relative to the test framework in § Conventions if recorded; otherwise assess structurally (presence/absence and placement of test files)
3. Risk areas — fragile code, high-complexity modules, missing error handling

Return findings as: **[Dimension]** — Evidence | Location | Issue | Recommendation | Why

**Evidence:**
- Proven — Backed by concrete reference (file content, manifest entry, system.md convention)
- Inferred — Based on judgment or best practice

Report only — do not fix. If no issues found, state "No findings" explicitly.
```

Read existing `specs/delivery/issues.md` entries (absent = none); determine the current highest ISS-NNN for ID continuity. Write each finding to `specs/delivery/issues.md` (create if absent), incrementing from the highest existing ID — follow the `issues.md` schema in [shared-artifacts.md](shared-artifacts.md) § issues.md. Set `Blocker: true` only if the finding makes continuing delivery work on this codebase unsafe or would make the next planning run materially wrong (e.g. a broken build, a security hole in a shared path); `false` otherwise. Compare your output against the `issues.md` schema — confirm all required keys are present.

**Step 9 — Operational Tooling Scan:**

Detects existing operational tooling. For each category, search the filesystem:

- **IaC**: `*.tf` (Terraform), `cdk.json`/`cdk.ts` (CDK), `template.yaml` (SAM/CloudFormation), `serverless.yml`, `Pulumi.yaml`, `*.bicep` (Bicep), `*.json` ARM templates
- **CI/CD**: `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`, `buildspec.yml`, `.circleci/config.yml`, `azure-pipelines.yml`, `cloudbuild.yaml`
- **Secrets**: provider secret service references in source (e.g. SSM ARN patterns, Key Vault references, GCP Secret Manager calls), `.env.example` with non-obvious placeholder values
- **Observability**: structured logging setup (Winston, Pino, `console.log` patterns with level), cloud monitoring SDK imports (e.g. CloudWatch, Azure Monitor, GCP Cloud Monitoring, Datadog, Sentry)
- **Environments**: multiple `.env.{env}` files, `terraform/environments/`, `config/{env}.json`, environment-specific config structure
- **Rollback**: function/service versioning config in IaC or deploy scripts, object storage versioning in IaC, blue/green or canary deployment patterns

Record each finding as one line in `system.md § Foundation`. For Iteration runs, overwrite the existing per-category line in place (each category has exactly one line at all times):

- Detected → `{category}: {tool/pattern} detected at {path}`
- Not found → `{category}: none detected`

These findings drive operational decisions in the consuming skill — never propose a new tool when an existing one is detected.
