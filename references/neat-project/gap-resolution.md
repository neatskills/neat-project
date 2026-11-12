# Gap Resolution

Governs how unknowns are identified, classified, resolved, and written to artifact fields across the skill pipeline. Applies within one existing Step of the calling skill — the caller supplies the node graph; this file governs node types, loop mechanics, artifact output, and cross-skill continuity.

## Node Types

Three types, classified by resolution mechanism:

- **Fact node** — resolvable by lookup (a loaded file, a grep, a schema field already in context). AI resolves synchronously. Never presented to the user.
- **Assumption node** — resolved by lookup but with low confidence (ambiguous result, stale source, or conflicting signals). Surfaced as a check: state what was found and ask for confirmation — no recommendation, the user is confirming not choosing. If confirmed wrong but the correct answer is unknown, demoted to a decision node.
- **Decision node** — resolvable only by user judgment. Always presented; paired with a recommendation unless purely informational (something only the user knows, with no tradeoff to weigh) — ask those plainly.

## Node Schema

Each node the calling skill seeds into the graph carries:

| Field | Description |
| --- | --- |
| `topic` | The gap it covers (e.g. `"backend language"`, `"auth model"`) |
| `section` | The artifact field or section the resolved value writes to (e.g. `prd.md § Constraints`). Specified by the calling skill — the loop treats it as opaque. |
| `type` | `fact` \| `assumption` \| `decision` |
| `prerequisites` | Other `topic` values that must resolve before this node unblocks |
| `unresolved-outcome` | What the calling skill does if this node stays unresolved: `open` (embed `{open: topic}` callout in `section`) \| `risk` (write a risk entry to the artifact's risk section) |

## Resolution Outcomes

| Outcome | Condition | Artifact output |
| --- | --- | --- |
| **Resolved** | Answer obtained (by lookup or user) | Written to `section` |
| **Skipped** | User can't or won't answer; `unresolved-outcome: open` | `{open: topic}` callout embedded inline in `section` |
| **Risk** | User can't or won't answer; `unresolved-outcome: risk` | Risk entry written to the artifact's risk section |
| **Disconfirmed** | Assumption confirmed wrong; correct answer unknown | Node demoted to decision node; re-enters loop unblocked |
| **Contradicted** | A later answer conflicts with a previously resolved node | Node reopened → re-resolved → `section` updated |
| **New sub-decision** | An answer reveals a decision that didn't exist before | Added to graph with `prerequisites` naming the revealing node; inherits the revealing node's `section` and `unresolved-outcome` unless the new topic clearly belongs elsewhere; `type` classified per Node Types |
| **Pruned** | Calling-skill's seeded conditional pruning rule fires | Node and its branch removed from the graph; announced in the next round's lead-in |

## The Loop

Step 1 runs once. Steps 2–4 repeat until the frontier is empty. Step 5 confirms and exits, but loops back to Step 2 if the user supplies a correction.

Seed → Resolve → Gate → Answer → Confirm

**Step 1 — Seed:** Build the graph from the caller's already-loaded context. There is no persisted checkpoint — if the session was interrupted, re-derive the graph fresh from whatever is now durably recorded. Skip any topic already answered in a loaded artifact. Only add a node when it's genuinely blocking.

**Step 2 — Resolve:** Resolve all reachable fact nodes to a fixpoint. For each unblocked fact node, attempt lookup synchronously (grep, read — never a background subagent). If found, write to `section` and mark resolved; this may unblock further nodes — continue until no unblocked fact nodes remain. If a lookup finds nothing at all (not ambiguous — simply absent), escalate: reclassify as assumption if partial evidence exists in context, as decision if none. Never drop a fact node silently. Return to Step 3.

**Step 3 — Gate:** Compute the frontier: all unblocked, unresolved decision and assumption nodes. If empty, proceed to Step 5. Otherwise proceed to Step 4.

**Step 4 — Answer:** Present up to 3 frontier nodes ordered by blocking-depth (most dependents first). Wait for answers. For each answer, apply the matching Resolution Outcome above. Two notes beyond the table: a **new sub-decision** doesn't unblock anything on its own — it enters the frontier once its own prerequisites (typically the revealing node, already resolved) are met; conditional pruning ("if topic X resolves to value Y, drop topic Z") is calling-skill logic declared at seeding time, not a node schema field, but a **pruned** branch must still be announced in the next round's lead-in — never dropped silently.

Return to Step 2.

**Step 5 — Confirm:** Summarize each settled decision and its answer, then ask the user to confirm it is complete and correct. Confirmed → session ends, calling skill may act on the graph. Correction or new information → reopen affected nodes and return to Step 2. An empty frontier alone is not session end — do not act without this confirmation.

## Recommendation Rule

Every decision node ships with a recommended answer, unless purely informational — ask those plainly. Ground the recommendation in already-loaded context (a stated convention, a prior recorded decision, a scan finding). When context is thin, fall back to a best-practice recommendation labelled **(convention)**. Never omit a recommendation solely because context is sparse.

## Round Format

Number each question in the round (Q1, Q2, Q3 — presentation order only; the node stays keyed by `topic` internally) so the user can answer by digit. Decision nodes use the standard format:

```text
Q1 — <topic>: <question body>

➡ Recommended: <answer + one-line why>
```

Assumption nodes omit the recommendation — the user is confirming, not choosing:

```text
Q2 — <topic>: Found <what was found>. Is this correct?
```

Order the round by blocking-depth (Step 4 above), not by a running count. A total question count above ~15 signals the calling skill seeded too many nodes — prune the graph before starting, not by running more rounds.

**`{open:}` callout format.** When a node is skipped or unanswerable (`unresolved-outcome: open`), embed inline in `section`:

```text
{open: <topic>} <one-line description of what remains unresolved>
```

Example: `{open: auth-model} Authentication model undecided — no constraint found in loaded context.`

## Cross-Skill Continuity

Nodes resolved in one skill become facts in the next because they are written to artifacts that downstream skills load. The inheritance chain:

```text
assessing   → prd.md / requirements{NN}.md
solutioning → solution.md
planning    → decisions.md / wave{NN}-{name}.md
fixing      → issues.md (escalations and patterns) — no frontier loop
building    → (inline escalation only — no frontier loop)
```

**Seeding rule:** when a skill loads an upstream artifact, any topic already answered in that artifact is pre-resolved — seed it as a fact node (self-resolving) rather than re-asking.

**Propagating unresolved items:** `{open: topic}` callouts in a loaded artifact are unresolved nodes from the prior skill. The loading skill should grep for `{open:` in loaded artifacts and re-seed those topics as decision nodes in its own graph. When re-seeding, assign a new `section` appropriate to the loading skill's own artifact — `topic` is the continuity link, but `section` and `unresolved-outcome` are always skill-specific and must be re-declared. An item can chain through multiple skills: it stays `open` as long as it remains unresolvable at each stage. It terminates when a skill resolves it, or when a skill applies `unresolved-outcome: risk` — writing a risk entry and not propagating the `{open:}` callout further.
