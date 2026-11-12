# Wave Selection Decisions

**Trigger:** Phase 1 Step 2 — after reading delivery.json and determining routing.

## Input Context

1. **Planned waves:** waves with `status=planned` — read `§ Requirements` table to build dependency edges
2. **Building waves:** waves with `status=building` — run parallel safety check per [wave-rules.md](wave-rules.md)
3. **Coordinator resumption:** building wave with `{base}/wave{NN}-{name}-coordinator-state.json` present

## Decision Matrix

| Scenario | Conditions | Options Presented |
|----------|-----------|-------------------|
| **Coordinator resume** | Building wave + coordinator-state.json exists | "Resume coordinator build: {wave}" |
| **Both building & planned** | Safe building waves with free features + ready planned waves | [1] Next feature in {wave}: {free features}<br>[2] Start new wave: {ready planned waves}<br>[3] Refactor |
| **Building only** | Safe building waves with free features, no planned waves | [1] Next feature in {wave}: {free features}<br>[2] Refactor |
| **Planned only** | No safe building waves, 1+ planned waves | [1] Start new wave: {ready planned waves}<br>[2] Refactor |
| **No options** | No safe building waves + no planned waves | Stop — explain why no waves available |
| **No free features** | Building waves exist but all features claimed/blocked | Explain — no features available |

## Parallel Safety Check

Building wave is **safe for parallel** if:
- **No coordinator state** (`coordinator-state.json` absent)
- **Passes wave-rules.md § Parallel Safety Check**

Not safe → do not offer parallel option (exclude from [1] option above).

## Wave Selection Format

**Wave list:**
> [1] wave01-req-020-027 (Projects UI + Code Review)  [2] wave02-req-019-021-022 (Auth + Settings)

**Feature list:** For each feature, read from wave file's `## Features / ### {feature-slug}` and count tasks:
> [1] auth (5 tasks)  [2] payment (8 tasks)  [3] notifications (2 tasks)

**Why show task counts:** Helps users decide whether to build multiple small features sequentially (2-3 tasks) vs. one large feature per session (8+ tasks) vs. parallel sessions.

## User Selection Routing

**Feature (option 1 in building scenarios):**
- If multiple → user picks one
- Set `{wave}`, `{wave-file}`, `{feature}` → skip to Step 3 (claim feature)

**New wave (option 1 in planned-only, 2 in both):**
- If multiple → user picks one
- Set `{wave}`, `{wave-file}` → proceed to **Build Mode Decision** below

**Refactor (option 2 in single-type, 3 in both):**
- Follow [refactoring-protocol.md](refactoring-protocol.md) → when complete, return to Step 2

**Coordinator resume:**
- Set `{wave}`, `{wave-file}` → run Step 4 (load plan) → Follow [coordinator-protocol.md](coordinator-protocol.md)

## Build Mode Decision (New Waves Only)

**After wave selection, before claiming:**

1. Read `{wave-file} § Features` for selected wave
2. Count features and tasks per feature
3. Classify: Large (4+ tasks), Small (1-3 tasks)

**Decision matrix:**
- **1–3 features, all small** → recommend Batch mode (coordinator overhead not worth it)
- **Otherwise** (4+ features OR any large feature) → recommend Coordinator mode

**Present:**
> Wave {wave}: {N} features ({X} large, {Y} small). How do you want to build this wave?
> [1] Coordinator mode — parallel subagents where possible, sequential where dependencies require. One session, respects `depends_on`.
> [2] Batch mode — sequential implementation, one feature at a time. No subagents.
> Recommended: [{recommended}]

**User selects [1] Coordinator:**
- Run Step 3 § Claiming a wave items 1–3 (claim in delivery.json; no progress.json)
- Run Step 4 (load plan)
- Follow [coordinator-protocol.md](coordinator-protocol.md) — replaces Step 5 and Phases 2–8
- Coordinator exits through Phase 8 § Continue or exit → Phase 9 (retro)

**User selects [2] Batch:** continue with Step 3 (claim wave and first feature).

## Topological Sort Rules

**Planned waves:** Sort by inter-wave dependencies (ignore contract edges). Exclude any wave with edge to a `building` wave (not safe for parallel).

**Free features (within building wave):** Feature is free if:
- No entry in progress.json OR `status ≠ "in-progress"`
- All `depends_on` features have `status = "done"` in progress.json
