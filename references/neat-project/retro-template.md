# Retro

Templates for each skill's terminal `## Phase N: Retro` section. The calling skill fills in `{skill-name}` (its `name:` frontmatter value) and any noted overrides.

## Simple

Used by skills with a single-pass retro (no project-convention capture).

Emits a retro report covering the full run.

Scan the conversation for mid-run corrections and tool call failures — then emit:

````retro
Retro: {skill-name}

- {execution-overhead}
- Mid-run corrections: list each (what happened, what was expected; note if user-initiated)
- Tool call failures: list each (command + what failed)
````

Done.

**Default `{execution-overhead}`:** `Execution overhead: unnecessary sequential steps, wasted subagent round-trips, rework from ambiguous specs`

## With Conventions Capture

Used by skills that also distill user-initiated corrections into durable CLAUDE.md conventions (multi-wave or multi-run skills, where the same convention could otherwise be re-taught every session).

Emits a retro report covering the full run.

For each user-initiated mid-run correction, classify before emitting: "Would a different project using this skill have the same problem?" Yes → **skill-gap**. No → **project-convention** (carry to Step 2).

**Step 1 — Skill-gap:**

Scan the conversation for mid-run corrections and tool call failures — then emit:

````retro
Retro: {skill-name}{title-suffix}

- Execution overhead: unnecessary sequential steps, wasted subagent round-trips, rework from ambiguous specs
- Mid-run corrections: list each (what happened, what was expected; tag user-initiated as skill-gap or project-convention)
- Tool call failures: list each (command + what failed)
````

**Step 2 — Project conventions** *(conditional{step2-conditional-suffix} — only if project-convention items were identified above):*

For each project-convention, distill to a reusable rule (e.g., "Money values must use BigDecimal, never float"). Present:

> Project convention found: `{rule}`. Add to CLAUDE.md?
> [1] Yes
> [2] No
> [3] Not sure — I'll check

For each approved rule: append to `CLAUDE.md § Project conventions` (create file/section if absent; skip if equivalent rule already present; never modify other sections).

Done.

**`{title-suffix}`:** optional text appended to the retro title (e.g. planning: `· wave {wave}`); omit if not applicable.
**`{step2-conditional-suffix}`:** optional additional clause (e.g. planning: `— run once at end of the full session, after all waves, not per wave`); omit if not applicable.
