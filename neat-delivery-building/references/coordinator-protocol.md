# Coordinator Protocol

Builds one wave with parallel subagents. Replaces building Phase 1 Step 5 and Phases 2–8 for the wave.

**Execution model:** features run in dependency rounds. Each subagent works autonomously and returns one report when it finishes — subagents cannot message the coordinator or the user mid-run. Questions a subagent cannot settle come back as `blocked` features and are answered between rounds.

## Phase 1: Setup

Loads the wave, settles every wave-level question, and plans the first round.

**Step 1 — Load:**

- Already loaded by building Phase 1 Step 4: `{wave-file} § Requirements` rows, `system.md`, module files, `delivery-rules.md`, `CLAUDE.md`, and feature slugs with task counts from `{wave-file} § Features`.
- Load every feature's full section from `{wave-file} § Features` — round planning needs each task's `files`.
- `{coordinator-file}` exists → resume: 
  - Load coordinator-state.json
  - Features with `status: building` revert to `pending` with `resume_from` = their first task not in `completed`
  - **Clean stale worktrees:** Run `git worktree list | grep ".claude/worktrees/agent-"`. For each match, run `git worktree remove {path} --force`. Cleanup failure → warn user "Stale worktree at {path} — remove manually if needed", continue.
  - Skip to Step 3
- Otherwise write `{coordinator-file}` — Follow § coordinator-state.json schema below, every feature `status: pending`.

**Step 2 — Pre-implementation:**

- **Decisions and codebase inputs:** Follow building SKILL.md Phase 1 Step 5 (a) and (b). Record `pre_implementation.decisions_reviewed` in coordinator state instead of progress.json.
- **Open callouts:** Follow building SKILL.md Phase 2 Step 3 § Open callouts across all feature sections in `{wave-file}`. Cluster callouts by topic prefix (auth-*, payment-*, …) when presenting. Record resolutions in `pre_implementation.callouts`.
- **Naming values:** Follow building SKILL.md Phase 2 Step 3 § Naming alignment across all features' configuration, iac, and cicd tasks. Record in `pre_implementation.values`.
- **Preserve validation (compensating control for subagents):** For each feature that will be delegated to subagents, validate that task decomposition preserves requirement intent. Read feature requirements from `{wave-file} § Requirements`, compare against task specs. If drift detected (tasks elaborate beyond, invent features, or miss requirements) → revise tasks before spawning subagents.
- Write `coordinator-state.json`.

**Step 3 — Plan the round:**

- **Ready features:** `status: pending` and every `depends_on` feature is `done` (or outside this wave).
- **File overlap:** compare task `files` across ready features. Features sharing any file go in the same batch — a subagent builds its batch sequentially, so they never write concurrently.
- **Batch:** a feature with 4+ tasks gets its own batch. Features with 1–3 tasks share batches of up to 3. Overlap batches from above stay together regardless of size.
- Present the plan:

  > Round {N}: [auth] (8 tasks) · [notifications, admin] (5 tasks)
  > Waiting on dependencies: payment (after auth)

## Phase 2: Rounds

Runs each round, merges the reports, and settles anything the subagents could not.

**Step 1 — Run:**

- One batch in the round → build it directly, no subagent: Follow [execution-protocol.md](execution-protocol.md) in main session mode. Go to Step 3.
- Otherwise, per batch: build a subagent manifest — Follow [subagent-manifest.md](subagent-manifest.md). Set each feature to `status: building`, `round: {N}`, and write `coordinator-state.json`.
- Spawn one Agent per batch with `isolation: "worktree"`, all in a single message so they run in parallel and their reports return together. At most 5 per message — more batches go in a second message once the first returns. Prompt per Agent:

  ```text
  Read {skill_dir}/references/execution-protocol.md and follow it in subagent mode.

  Subagent manifest:
  {manifest JSON}
  ```

**Step 2 — Merge worktrees and reports:**

For each Agent result (task-notification):

1. **Extract worktree metadata:**
   - Parse task-notification for `<worktree>` element
   - Extract `<worktreePath>` (e.g., `.claude/worktrees/agent-a02287682db6e831c`)
   - Parse `<result>` to get subagent's JSON report
   - **If `<worktree>` absent OR report unparseable:**
     - Every feature in batch → `status: "failed"`, `failure: { "source": "merge", "error": "no worktree or invalid report", "details": "{error details}" }`
     - Skip steps 2-3, proceed to step 4

2. **Merge files to main:**
   - Collect all `touched_files` from all features in the batch report
   - For each file in `touched_files`:
     1. Verify exists: `[[ -f "{worktreePath}/{file}" ]]` → if missing, set `merge_error = "file missing: {file}"` and break
     2. Create parent dirs: `mkdir -p "$(dirname "{file}")"`
     3. Copy with metadata: `cp -p {worktreePath}/{file} {file}` → if fails, set `merge_error = "copy failed: {file}"` and break
     4. Verify copied: `[[ -f "{file}" ]]` → if missing, set `merge_error = "verify failed: {file}"` and break
   - **If merge_error set:**
     - Every feature in batch → `status: "failed"`, `failure: { "source": "merge", "error": "file merge failed", "details": "{merge_error}" }`
     - Skip cleanup (leave worktree for debugging)
     - Proceed to step 4
   - **All files copied successfully:** proceed to cleanup

3. **Cleanup worktree:**
   - `git worktree remove {worktreePath} --force`
   - Cleanup fails → warn user: "Worktree cleanup failed: {worktreePath} — may have stale worktrees", continue (non-blocking)

4. **Merge feature state:**
   - **If file merge failed (from step 2):**
     - For each feature in batch: update coordinator state with `status: "failed"` and `failure` from step 1 or 2
     - Skip module map, skip issues.md (no files were merged)
   - **Otherwise (file merge succeeded):**
     - Per feature: copy `status`, `completed`, `touched_files`, `quality`, `deviations`, `resume_from`, and `blocker` or `failure` from subagent report into coordinator state
     - Module map: for each touched file, patch `modules/{unit}.md` `source_files` per [delivery-rules.md](../../references/neat-project/delivery-rules.md) § Task Types § Execution (Module map update)
     - Issues: write each reported `issues` entry to `specs/delivery/issues.md` per building SKILL.md **Escalate to planning**
   - Set `worktree_path: null` for all features in batch (cleanup done or debugging complete)
   - Write `coordinator-state.json`

**Step 3 — Resolve:**

- **Pending quality decisions** (`decision: "pending"` in `quality`): present the prompt — method entries per [quality-gates.md § Refactoring Strategies](quality-gates.md § Refactoring Strategies) § Tier 1 user prompt, file entries per [quality-gates.md § File-Level Quality Check](quality-gates.md § File-Level Quality Check) § Analysis & Prompt. Apply the choice inline and set `decision`.
- **Blocked features:**

  > {feature} blocked on task {task}: {question}
  > Context: {context}
  > [1] Answer inline
  > [2] Escalate to planning

  - [1] → append `{question, answer, feature}` to `decision_log` (also to `pre_implementation.callouts` if the blocker is an `{open:}` callout). Set `status: pending` — the feature resumes from `resume_from` in the next round, and every later subagent manifest carries the answer.
  - [2] → Escalate to planning. Set `status: escalated`.
- **Failed features:**

  > {feature} failed on task {task}: {error}
  > {details}
  > [1] Fix inline, then resume
  > [2] Escalate to planning
  > [3] Accept partial — record unbuilt tasks as a deviation

  - [1] → fix the failing task here and run feature-scoped tests. Append the task slug to `completed`, set `resume_from` to the next task (feature done if none), `status: pending` (or `done`).
  - [2] → Escalate to planning. Set `status: escalated`.
  - [3] → set `status: done`. Add the unbuilt task slugs to `deviations` and log a `gap` per **Escalate to planning**.
- Write `coordinator-state.json`.

**Step 4 — Next round:**

- Any ready features → Phase 1 Step 3.
- All features `done` → Phase 3.
- Otherwise (remaining features held behind `escalated` ones) → stop. Tell the user which features are held and by what. The wave stays `status=building` in `delivery.json` and `coordinator-state.json` is kept for resume.

## Phase 3: Finisher

Runs the wave-level phases once every feature is done.

Run building SKILL.md Phases 3–8 for this wave, with one substitution: read wave-wide context (`touched_files`, `quality`, `deviations`, `completed`) from coordinator state instead of `progress.json`, and skip every `progress.json` write. Phase 8 § Mark wave complete runs items 5–6 only (`delivery.json`).

After Phase 8 § Mark wave complete, delete `{coordinator-file}`. Then exit per Phase 8 § Continue or exit.

## Interruption

- State is written after every Phase 1 step and every Phase 2 Step 2 and Step 3, so a stopped session loses at most the running round's bookkeeping. Files the subagents wrote stay on disk.
- On restart, building Phase 1 Step 2 offers "Resume coordinator build" for the wave — Phase 1 Step 1 resumes from `coordinator-state.json`.
- `delivery.json` status follows building SKILL.md Phase 8 § Stop rule.

## coordinator-state.json

Follow [artifacts.md](artifacts.md) § coordinator-state.json for schema and lifecycle.

After each write, compare against the schema — confirm all required keys are present.
