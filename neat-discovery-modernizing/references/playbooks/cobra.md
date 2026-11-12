# Cobra Playbook

Tech stack: Go CLI applications using the Cobra framework (`github.com/spf13/cobra`).

## Evidence

Additional scan targets in `legacy_path`:

- `go.mod` — Go version and Cobra version (`github.com/spf13/cobra vX.Y.Z`)
- `cmd/` directory — command tree structure and depth
- `main.go` — entry point and root command wiring
- Package-level `var` declarations in `cmd/*.go` — global flag state pattern
- `init()` functions in `cmd/*.go` — flag registration via `init()`
- `*_test.go` distribution: files in `cmd/` vs `internal/` or `pkg/`

Additional documents to request after standard document intake:
> "Share any available: shell completion requirements, man page generation config, plugin or extension architecture docs."

## Evaluate

Dimension overrides — apply these in place of the generic signals for the specified dimensions.

**Dimension 4 — Technology health:**

| Signal | Red | Amber | Green |
| --- | --- | --- | --- |
| Go version in `go.mod` | < 1.21 | 1.21 | ≥ 1.22 |
| Cobra version | < 1.7 | 1.7 | ≥ 1.8 |
| Context propagation | No `context.Context` parameter in any `RunE` function | Partial — some commands accept and propagate ctx | All `RunE` functions accept and propagate ctx |

**Dimension 2 — Changeability:**
Apply additional signal: ratio of `_test.go` files in `cmd/` vs `internal/`/`pkg/`. Tests in `cmd/` test Cobra plumbing, not business logic — high ratio indicates low effective test coverage.

- Red: > 80% of test files located in `cmd/`
- Amber: 50–80% of test files located in `cmd/`
- Green: < 50% of test files located in `cmd/`

**Dimension 1 — Structural complexity:**
Apply additional signal: count package-level `var` declarations holding flag values in `cmd/*.go`. Each is shared mutable state that couples commands.

- Red: > 10 global flag vars, or any `init()` function registering flags
- Amber: 5–10 global flag vars
- Green: < 5 global flag vars; flags bound to per-command option structs

## Derivation

Stack-specific rules — evaluated before standard Rules 1–10. Unlike Rules 1–10 (first-match-wins), P1–P3 are evaluated additively — each matching condition inserts its phase.

| Priority | Condition | Phase inserted |
| --- | --- | --- |
| P1 | > 10 global flag vars or `init()` flag registration present | Flag state refactor — extract global flag vars into per-command option structs; remove `init()` flag registrations |
| P2 | Business logic found directly in `RunE` closures (not delegating to a separate function) | Logic extraction — move `RunE` bodies into testable service or handler functions; `RunE` becomes a thin adapter |
| P3 | No `context.Context` in any `RunE` | Context propagation — thread `ctx` through command tree and all downstream calls |

**Note:** P1 and P2 often co-occur — apply both; if both fire and the combined estimate is ≤ L, merge into a single refactor phase.

## Design

Additional patterns — extend the Pattern Selection table with these before determining the pattern for each phase.

| Pattern | Use when |
| --- | --- |
| Option struct | Global flag vars present — define a `type Options struct` per command, bind flags to struct fields with `cmd.Flags().StringVar(&opts.Field, ...)`, pass struct to the run function |
| RunE delegation | Business logic in `RunE` — `RunE` calls an injected handler or service; enables unit testing without Cobra setup overhead |
| Context threading | Missing context propagation — add `ctx context.Context` as first parameter to `RunE` and all downstream functions; wire from `cobra.Command.ExecuteContext` at the root command |
