# Workspace, Modules, and Task Types

Covers path variable derivation, module loading, and task type classification and execution.

## Workspace Path Variables

Path variables derived from `specs/delivery/delivery.json planning_model`. Used by planning and building.

| `planning_model` | `{base}` | `{kb}` |
| --- | --- | --- |
| `phase-first` (shared paths) | `specs/delivery/` | `specs/system/` |
| `product-first` (per-product paths) | `specs/delivery/{product}/` | `specs/system/{product}/` |

- Wave files live at `{base}/wave{NN}-{name}.md` (`{wave-file}`). Each feature is a `### {feature-slug}` section under its `## Features`.
- Building state sits next to the wave file: `{base}/wave{NN}-{name}-progress.json` (batch mode) or `{base}/wave{NN}-{name}-coordinator-state.json` (coordinator mode).
- If `delivery.json` is absent: skills that derive build-history paths proceed without build history (do not block).
- Assessing and solutioning derive the build-history path from `planning_model` inline — they do not use this reference.

## Module Loading

Standard procedure for loading module context from `specs/system/modules/`. Used by planning and building before breaking down or implementing a wave.

1. Load `specs/system/modules/index.md`. If absent: load all module files in full and skip step 2.
2. Match words ≥ 4 chars from requirement titles and risk descriptions against each index row's Name, Path, and Key Exports. Auto-load all matched modules and log what was loaded:

   > Loading modules for {wave}. Matched candidates (from keyword matches):
   > {list: name (path) — matched keywords}

3. Load full `modules/{name}.md` files for all matched modules, extended via `internal_imports` up to 2 levels deep.
4. Always load `specs/system/modules/shared-modules.md` read-only — must not be re-implemented.
5. Zero candidates (whether the index is absent, or present with no keyword matches) → warn:

   > No modules found — proceeding without module context.

## Task Types

Four task types are valid in delivery: `implementation`, `configuration`, `iac`, and `cicd`.

### Classification

Use the operational markers read in planning Phase 7 Step 1 to classify:

| Type | Description | Assign when |
| --- | --- | --- |
| `implementation` | writes/modifies source files; unit tests written and run during build | app logic, logging code, behavioral changes |
| `configuration` | manual steps only — CLI, cloud console, secrets; no source file output; deferred to runbook | secret creation/rotation (always), manual console actions |
| `iac` | writes IaC resource definition files (e.g. Terraform `.tf`, CDK `.ts`, SAM `template.yaml`, Pulumi `index.ts`, Bicep `*.bicep`); validated locally, execution deferred to runbook | resource provisioning task **and** `Infra: iac:*` |
| `cicd` | writes pipeline config files (`.github/workflows/*.yml`, `buildspec.yml`); no runbook entry — triggers on git push | pipeline config task **and** `Delivery: cicd:*` |

Classifier rules (in priority order):

- Secret creation/rotation → always `configuration`, regardless of other markers
- Task modifies application source code (app logic, structured logging) → always `implementation`
- Task provisions cloud resources + `Infra: iac:*` → `iac`
- Task writes pipeline config + `Delivery: cicd:*` → `cicd`
- Operational markers at default or non-matching values (`Infra` not `iac:*`, `Delivery` not `cicd:*`) → use `implementation` or `configuration` only (no regression)
- If the `Delivery` marker names a specific CI/CD vendor (e.g. `cicd:github-actions`), the vendor choice is a hard-to-reverse architectural decision — note it in the `cicd` task spec's `acceptance_criteria` as vendor-committed. (ADR evaluation for the vendor choice happens in planning Phase 6 Step 3.)

### Execution

**Module map update** (`implementation` and `iac` tasks), for each file written or modified:

- Find the module whose `path` it falls under (from the `modules/*.md` files loaded in building Phase 1 Step 4).
- Not yet listed in that module's `source_files` → append it (union merge — never remove existing entries).
- No existing module's `path` matches → leave the module map unchanged. Note the unmapped file for Phase 6's `build-history.md` Deviations entry. Log a `gap` in `issues.md`: "no module found for `{path}` — verify solutioning completed for this track."

**`implementation` tasks:**

Sub-steps 1, 2, and 3 below are read-only and safe to re-run on resume.

1. Read the task spec.
2. **Read relevant source:**

   **Sub-step 1 — Files being modified (no cap):** For each file path in the `files` field, check if it exists on disk. Exists → read content, regardless of stated intent. Does not exist → skip (net-new file; nothing to read). _Anchoring note: content read here describes current state only — where the spec indicates replacement or rewrite, implement per the spec, not the existing code. For files being fully replaced this read adds context overhead with no correctness benefit; this is a known accepted tradeoff._

   **Sub-step 2 — Codebase inputs:** For each entry in `inputs`:
   - Is it a plan output? Check whether any prior step in this feature or a prior feature declared it as an `output`. Use path-based matching where possible; if not matchable by path, treat as a codebase input.
   - Is a plan output → read from disk (may not be in-context in coordinator/parallel/resume scenarios).
   - Is not a plan output → check Phase 1 Step 5b session state first: already surfaced and resolved there → follow the same resolution without asking again. Not yet surfaced → grep/find to locate in codebase: found → read content; not found → ask user inline before proceeding.
   - **Gate: do not proceed to Step 3 until all inputs are resolved.**
   - _Known inefficiency: parallel tasks at the same step number that share a codebase input each read it independently — no deduplication across parallel tasks._

   **Sub-step 3 — Bounded pre-pass for import gaps:** Scan the `source_files` field for import paths referencing modules not already loaded in Phase 1. Run before Step 3. Skip entirely if no unresolved import paths exist.

   For each unresolved import path, in declaration order from the `source_files` field:
   1. Check exports in already-loaded `modules/{name}.md` — signature present → done, no file read.
   2. Not in exports → identify owning module via longest prefix match against each loaded module's `path` field:
      - One match → grep for the symbol within that module's `source_files` list: found → read that file; not found → fall back to project-wide grep.
      - No match → project-wide grep directly.
      - Tie (multiple equal-length prefix matches) → grep across all tied modules' `source_files` combined.
      - `scan_status: failed` → skip `source_files`; warn: `"Module scan failed for {module} — interface context may be incomplete"`; fall back to project-wide grep.
   3. Project-wide grep returns multiple hits → select the file whose path most closely matches the import path; if still ambiguous → take the first result.

   **Cap: at most 3 additional files total across all import paths in this task**, resolved in declaration order. Stop when cap is reached — remaining unresolved gaps proceed with available context.

3. Write unit tests from `acceptance_criteria` — all must fail. Skip for purely structural criteria (file/dir exists, config key present, export declared). Confirm all fail; if any pass, surface to the user before implementing.
4. Implement to make unit tests pass. **Implementation choice order** — take the first option that fully meets the spec: (a) a helper, component, or pattern already in the codebase (check exports in the loaded `modules/*.md`), used the way the surrounding code uses it; (b) the standard library or a platform feature; (c) a dependency already in the package manifest; (d) new code, the minimum that passes the tests. Never add a dependency that is not in the manifest or `solution.md` — that is an architecture decision; surface it to the user. Add every file written or modified to `touched_files` and apply § Module map update. **WCAG check:** if the task's `acceptance_criteria` specify a WCAG AA or AAA target and the implementation writes CSS, verify every foreground/background pair before the structural gate — normal text requires ≥4.5:1, large text (18 pt / 14 pt bold) and UI components ≥3:1. Adjust color values; if a pair still fails after 2 attempts, surface the conflict to the user (e.g., palette vs. contrast target) before proceeding.
5. Structural gate: verify each declared output exists, no unintended files modified, all outputs non-empty. On failure: discard implementation (keep tests), re-implement once; second failure → ask user inline.
6. Gate passes → task complete. Building records `{slug}` in `features.{feature}.completed` in `progress.json` (building Phase 2 Step 4) — never write to `{wave-file}`.

**`configuration` tasks:**

1. Read the task spec.
2. Record the task in the runbook queue: using the agreed values from building Phase 2 Step 3 (naming alignment), translate the task's `acceptance_criteria`, `outputs`, and `conventions` into clean operational instructions with all placeholders substituted. Store this for Phase 7 runbook generation — do not present to the user now and do not wait for confirmation.
3. Task complete — building records `deferred:{slug}` in `progress.json` (building Phase 2 Step 4). Do not add files to `touched_files`. Continue immediately to the next task.

Configuration tasks are never executed during the wave build. All manual provisioning steps are consolidated into the phase-level runbook and executed once after all waves in the phase are built.

**`iac` tasks:**

1. Read the task spec.
2. Write IaC resource definition files (`infra/main.tf`, `lib/stack.ts`, `template.yaml`, or tool-conventional path) per the task spec. Add every file written to `touched_files` and apply § Module map update.
3. Run the IaC validation command for the tool named in the `Infra:` marker: Terraform → `terraform validate` (in the `infra/` directory); CDK → `cdk synth --no-rollback`; SAM → `sam validate`; Pulumi → `pulumi preview`; Bicep → `az bicep build --file {file}`; other tool → skip validation with a note. If the tool is not installed locally → surface to user: "`{tool}` not installed — IaC files written but not validated; validate manually before running `{tool} apply`." Do not proceed silently.
4. Structural gate: verify each declared output file exists and is non-empty. On failure: discard implementation (keep spec), re-implement once; second failure → ask user inline.
5. Queue runbook entry: `{tool} init && {tool} plan && {tool} apply`, plus the list of output values to capture as forward references in the Values table.
6. Task complete — building records `iac:{slug}` in `progress.json` (building Phase 2 Step 4).

**`cicd` tasks:**

1. Read the task spec.
2. Write pipeline config files (`.github/workflows/*.yml`, `buildspec.yml`, `.gitlab-ci.yml`, or equivalent) per the task spec. Add every file written to `touched_files`.
3. Run `yamllint {file}` if installed; if not installed, skip with note.
4. Structural gate: verify each declared output file exists and is non-empty. On failure: re-implement once; second failure → ask user inline.
5. No runbook entry — pipeline execution triggers on git push.
6. Task complete — building records `{slug}` (no prefix) in `progress.json` (building Phase 2 Step 4).
