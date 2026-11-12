# Artifact Status Marker

Shared semantics for the `status` frontmatter field in human-review artifacts.

## Valid Status Values

### Discovery Artifacts (prd.md, requirements{NN}.md, solution.md)

| Status | Meaning | Transitions To |
|--------|---------|----------------|
| `draft` | Work in progress, not ready for review | `final` |
| `final` | Reviewed and approved, frozen (immutable for prd.md and requirements; updated in place for solution.md) | - |

**Lifecycle:** `draft` → `final`

**Freeze-after-final:**

- `prd.md` with `status: final` is immutable
- `requirements{NN}.md` with `status: final` is immutable
- `solution.md` with `status: final` can be updated in place during iterations (exception to freeze rule)

### Delivery Artifacts (wave files, delivery.json entries)

| Status | Meaning | Transitions To |
|--------|---------|----------------|
| `pending` | Wave identified but not yet planned | `planned` |
| `planned` | Feature breakdown complete, ready to build | `building` |
| `building` | Currently being built by neat-delivery-building | `built` |
| `built` | Build complete, wave delivered | - |

**Lifecycle:** `pending` → `planned` → `building` → `built`

## Format

Every human-review artifact starts with YAML frontmatter containing a `status` field:

```yaml
---
status: draft
---
```

Valid values: `draft`, `final`

## Semantics

- **`status: draft`** — artifact is being worked on, or in a review/iteration loop. The skill has not yet completed its quality checks.
- **`status: final`** — skill has completed its quality and review steps for this artifact. The artifact is ready for human review or consumption by downstream skills.

## When to Set

- **On first write:** Set `status: draft` when the artifact file is first created.
- **At completion:** Set `status: final` once the skill has completed its quality and review steps for this artifact.
- **During iteration:** If a Review/iteration loop re-opens the artifact (user requests changes, skill revises), reset to `status: draft`. Set back to `status: final` when the iteration loop completes.

## Covered Artifacts

Human-review artifacts that carry the `status` marker:

| Artifact | Path | Skill |
|---|---|---|
| PRD | `specs/discovery/prd.md` | neat-discovery-assessing |
| Requirements (Iteration) | `specs/discovery/requirements{NN}.md` | neat-discovery-assessing |
| Solution Design | `specs/discovery/solution.md` | neat-discovery-solutioning |
| Modernization | `specs/discovery/modernization.md` | neat-discovery-modernizing |
| Runbook | `{base}/runbook.md` | neat-delivery-building |
| Plan | `{base}/plan.md` | neat-delivery-planning |

The `{base}` path variable follows [delivery-rules.md](delivery-rules.md).

## Usage in Skills

### Mode Detection

Skills use the `status` marker to detect whether an artifact is complete:

```
1. Check if prd.md exists
2. If exists, read frontmatter
3. If status=final → artifact is complete (triggers Iteration or Review mode)
4. If status=draft → artifact is in progress (Resume mode)
5. If status absent → legacy artifact (treat as final when no run is active)
```

Writing the marker follows § When to Set. To read it, the first 10 lines of the artifact are enough for the frontmatter.

## Contract with Software Factory

The software factory renderer triggers quality gates when it detects `status: final`:

- `prd.md` → `status: final` → Gate 1 (Review PRD)
- `solution.md` → `status: final` → Gate 2 (Review Solution)
- `plan.md` → `status: final` → Gate 3 (Review Plan)

The factory renders Markdown to HTML for human review, using the section headings and table structures defined in each skill's `references/artifacts.md` schemas.

## Legacy Artifacts

Artifacts without a `status` line are legacy. The factory treats them as final when no run is active. **Skills must not rely on this behavior** — always write the marker explicitly.
