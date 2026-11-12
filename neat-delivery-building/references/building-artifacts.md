# Building Artifacts

Files written to disk by neat-delivery-building.

## Discovery Level

### issues.md (`specs/delivery/issues.md`)

**Base schema:** See [../../references/neat-project/shared-artifacts.md](../../references/neat-project/shared-artifacts.md) § issues.md

**Building-specific additions:**

`Blocker` field semantics:
- Always `true`: dependency conflicts, contract violations, data migration safety, security vulnerabilities
- Usually `true`: missing infrastructure, auth/z failures, integration test failures
- Usually `false`: style, performance, refactoring, non-blocking tech debt

**Usage:** Written during Phase 2/3/4. Append only; update `Status` and `Resolution` in place.

## Workspace Level

### build-history.md

Append-only chronological log of all waves built.

```jsonc
{
  "### {wave}: {req-id-1} · {req-id-2} · ...": null,      // one entry per wave; req-ids sorted ascending
  "**Delivered:**": null,               // summary of what was built for this wave
  "**Deviations:**": null,              // omit if none
  "**Issues:**": null,                  // Phase 4 findings logged to issues.md with accept/open outcomes; omit if none
  "**Quality:**": null,                 // CRAP results — format per Quality field rules below; omit if no features had CRAP checks
  "**Architecture Changes:**": null    // system.md sections updated — "system.md §{Section}: {summary}"; omit if none
}
```

**Quality field:** One line per feature: `{feature-name}: {N} methods in {M} files`, then method issues (CRAP ≥ 15): `{method-name} ({score} → {decision})` and file issues (LOC ≥ 200 or CRAP Sum ≥ 50): `{file-name} (file CRAP {score} → {decision})`. Decision = "accepted" | "refactored" | "tests-added". Link ISS-NNN if debt created.

Example: `**Quality:** auth: 18 methods in 2 files — validateClaims (22 → accepted as ISS-042), service.ts (file CRAP 175 → gap ISS-043); payment: 2 methods in 1 file`

## Wave Level

### progress.json

Feature-level state and cross-session resume tracking — supports parallel building (multiple sessions, different features, same wave).

**Location:** `{base}/wave{NN}-{name}-progress.json` — e.g., `specs/delivery/wave01-booking-progress.json` or `specs/delivery/api/wave01-booking-progress.json`

```jsonc
{
  "wave": {                                   // wave state — shared across all sessions
    "skill": "neat-delivery-building",        // always "neat-delivery-building"
    "wave": "wave01-booking",                 // wave slug
    "status": "building",                     // building | closing | built
    "pre_implementation": {                   // answered once, reused by every session
      "decisions_reviewed": "2026-11-07 14:00",   // timestamp when decisions.md was reviewed
      "callouts": [                           // open callouts resolved during pre-implementation checks
        { "callout": "{SERVICE_NAME} placeholder in task auth-config", "resolution": "auth-service" }
      ],
      "values": {                             // naming alignment values, agreed before Phase 7 writes the runbook
        "SERVICE_NAME": "auth-service"
      }
    },
    "last_updated": "2026-11-07 14:05"        // YYYY-MM-DD HH:MM — last wave-level change
  },
  "features": {                               // one entry per started feature
    "auth": {
      "status": "done",                       // in-progress | done
      "task": null,                           // current/next task slug; null when done
      "completed": ["auth-001", "auth-002", "deferred:config-001", "iac:infra-001"],   // completed task slugs — prefix deferred: (configuration), iac: (iac)
      "touched_files": ["src/auth/service.ts", "src/auth/controller.ts", "tests/test_auth.py"],   // files modified by this feature — read by the parallel safety check
      "quality": {
        "methods": [                          // method CRAP results (Phase 2 Step 4)
          {
            "file": "src/auth/service.ts",
            "method": "validateToken",
            "score": 12,
            "complexity": 8,
            "coverage": 0.85,
            "decision": "accepted"            // accepted | refactored | tests-added | pending (coordinator mode — user decides between rounds)
          }
        ],
        "files": [                            // file-level results (Phase 2 Step 5)
          {
            "file": "src/auth/service.ts",
            "loc": 280,                       // lines of code in the file
            "crap_sum": 85.5,                 // Σ CRAP of all methods in the file
            "methods": 9,                     // method count
            "diagnosis": "big",               // clean | complex | big | both — from the 2D thresholds (LOC 200, CRAP Sum 50)
            "decision": "accepted"            // accepted | refactored | split | split-and-refactor | pending (coordinator mode)
          }
        ]
      },
      "deviations": ["Used bcrypt instead of argon2 (argon2 not in stack per system.md)"],   // deviations from plan — feed Phase 6 build-history.md
      "last_updated": "2026-11-07 14:05"      // YYYY-MM-DD HH:MM — last feature update
    }
  }
}
```

**Initial feature entry:** `status: "in-progress"`, `task` = first task slug, `completed` / `touched_files` / `deviations` = `[]`, `quality` = `{ "methods": [], "files": [] }`, `last_updated` = timestamp. Created on first claim (Phase 1 Step 3), auto-updated after tasks/verification, kept permanently.

### coordinator-state.json

Coordinator mode execution state — round-based tracking for parallel subagent builds, cross-round resume, question/failure aggregation. Created when user picks Coordinator mode. Deleted on wave completion.

**Location:** `{base}/wave{NN}-{name}-coordinator-state.json`

```jsonc
{
  "wave": "wave01-booking",         // wave slug
  "pre_implementation": {           // same shape as progress.json wave.pre_implementation
    "decisions_reviewed": "2026-11-07 14:00",
    "callouts": [{ "callout": "{open: auth-provider}", "resolution": "AWS Cognito" }],
    "values": { "SERVICE_NAME": "auth-service" }
  },
  "decision_log": [                 // answers to blocked questions; passed to every later subagent manifest
    { "feature": "auth", "question": "Extend BaseService or implement IUserService?", "answer": "Implement IUserService" }
  ],
  "features": {
    "auth": {
      "status": "pending",          // pending | building | blocked | failed | escalated | done
      "round": 1,                    // round this feature last ran in; null if not yet run
      "resume_from": null,          // task slug to resume from; null = start at the first task
      "completed": [],              // same prefixes as progress.json: deferred:{slug}, iac:{slug}
      "touched_files": [],
      "quality": { "methods": [], "files": [] },   // progress.json quality schema
      "deviations": [],
      "blocker": null,              // from the subagent report while status = blocked
      "failure": null,              // from the subagent report OR file merge failure: { "source": "subagent" | "merge", "task": "slug", "error": "...", "details": "..." }
      "worktree_path": null         // worktree path for this batch (if building); null when done/failed/blocked
    }
  },
  "last_updated": "2026-11-07 14:05"
}
```

**Lifecycle:**
- Created Phase 1 Step 2
- Updated after each round
- Read on resume
- Deleted Phase 8 on completion
- Mutually exclusive with progress.json: batch mode uses progress.json, coordinator mode uses coordinator-state.json
- After each write, confirm all required keys present

### runbook.md (`{base}/runbook.md`)

Phase-level operational guide — aggregates all manual configuration steps across every wave. Written on first wave with config/iac tasks, appended by subsequent waves. Frontmatter: `status: draft` (→ `final` on last wave completion).

````markdown
# Runbook

## Values
| Key | Value | Notes |
|-----|-------|-------|
| Environment | {env-name} | target environment (add when Environments: multi) |
| {var-name} | {agreed-value} | short description / where it was provisioned |

## {wave-slug} — {short title}           ← infrastructure sections, one per wave

### {step-intent}                          ← configuration tasks: one step per CLI/console action
**Why:** one sentence — what this step sets up and why it is needed.
1. actual CLI command or console action with values substituted
2. ...
**Done when:** verifiable criterion (e.g. CLI query output matches, console state confirmed)

### Provision infrastructure               ← iac tasks: one block per iac tool used this wave
1. Initialize: `{tool} init`
2. Preview: `{tool} plan`
3. Apply: `{tool} apply`
4. Capture outputs:
   ```

   {tool} output

   ```
   Record each output value in the Values table above.
**Done when:** `{tool} output` returns all expected keys with non-empty values.

## Build & Deploy                          ← written once, when the last wave is built
### Pre-deployment
### Infrastructure                         ← omit if no iac tasks in phase
### Build
### Deploy
### Smoke check

## Verification                            ← written once, when the last wave is built
All manual verification steps from each wave's feature acceptance_criteria across the phase,
grouped by boundary (CLI → Network → Browser → Observability). Automated tests are excluded.
Each check: steps with real commands substituted, expected output, done-when.
````

**Rules:** `## Values` table at top — updated in place each wave (new key → add row; same value → no-op; different value → conflict; resolved reference → update). When `Environments: multi`, add `Environment` row. Wave sections appended in build order — `configuration` tasks as numbered steps under intent heading, `iac` tasks as "Provision infrastructure" block. All placeholders replaced with agreed values. `## Build & Deploy` and `## Verification` appended once on last wave only.

**Build & Deploy subsections:** Pre-deployment (env vars from Values table, provider CLI auth check, IaC tool installed if `iac` tasks exist); Infrastructure (omit if no `iac` — `{tool} init && plan && apply`, capture outputs to Values table); Build (project build with env vars); Deploy (if `cicd` tasks: "Push to `{branch}`", else deploy script); Smoke check (service responds).

**Verification:** Pull manual steps from every wave's feature `acceptance_criteria`. Group by boundary (CLI → Network → Browser → Observability). Substitute real URLs/names from Values table. Exclude automated test coverage. Include Observability check (logs, no errors, alarms OK) only if tooling present per `system.md § Foundation`.
