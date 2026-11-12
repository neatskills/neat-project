# Parallel Safety

Coordination rules for building one wave across parallel sessions.

## Parallel Safety Check

For each building wave:

1. **Planning independence**: Read this wave's `{base}/wave{NN}-{name}.md § Requirements` table. Check `Requires` edges between them. If any `code` edge exists between requirements in the wave → not safe (features depend on each other). Only `contract` edges allowed.

2. **File overlap check**: Read this wave's `{base}/wave{NN}-{name}-progress.json`:
   - Collect `touched_files` from all features with `status: "in-progress"`
   - Read the wave file's `## Features` section
   - Identify free features: `depends_on` features are all `done` (check progress.json), and feature itself is not `in-progress`
   - For each free feature, collect `files` from all its tasks
   - Check for overlaps: any file path appears in both in-progress `touched_files` AND free feature task `files` → not safe

3. **Result**: If both checks pass → safe for parallel. Otherwise → not safe.

## Finisher Check

Run after a feature is marked complete.

1. Re-read `progress.json`
2. Check `wave.status`:
   - **"closing"** → Another session is finishing → stop:

     > Feature {f} complete. Wave is being finalized by another session.

   - **"built"** → Wave already complete → stop:

     > Feature {f} complete. Wave already built.

   - **"building"** → Continue to step 3
3. Check if ALL features are done:
   - List every feature in `{wave-file} § Features`
   - For each feature: check progress.json has entry with status: "done"
   - If any feature has NO entry in progress.json → not all done
   - If any entry has status != "done" → not all done
   - **All done** → This session is the finisher → Set `wave.status = "closing"`, proceed to Phase 3
   - **Some in-progress or not started** → Not finisher → continue below

**If not finisher:**

- Re-read `progress.json` to get current state
- Identify free features: `depends_on` features all done, not currently `in-progress`
- For each free feature, count its tasks in `{wave-file} § Features`
- **If no other feature in-progress:**
  - **Free features available** → Continue to next free feature (loop back to Phase 2 Step 2 with new feature)
  - **No free features** (all blocked by depends_on) → stop:

    > Feature {f} complete. Remaining features blocked by dependencies.

- **If another feature in-progress (parallel session active):**
  - **Free features available** → ask:

    > Feature {f} complete. What next?
    > [1] Build next feature: {list free features with task counts}
    > [2] Stop (let other session continue)

  - **No free features** → stop:

    > Feature {f} complete. Other features in progress or blocked.
