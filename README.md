# Neat Project

Structured AI-assisted software development: from discovery through implementation.

## Overview

Neat Project provides six Claude Code skills that guide software projects from initial requirements through tested implementation. Skills enforce quality gates, maintain knowledge artifacts, and produce verifiable deliverables at each phase.

**What it does:**
- Scopes and sizes requirements (assessing)
- Designs architecture and validates feasibility (solutioning)
- Plans implementation in dependency order (planning)
- Implements with TDD and quality gates (building)
- Debugs with root-cause analysis (fixing)
- Evaluates legacy system health (modernizing)

**What you get:**
- Structured artifacts (`specs/discovery/`, `specs/delivery/`, `specs/system/`)
- Progressive disclosure (load context on-demand, not upfront)
- Quality enforcement (CRAP scores, mutation adequacy, test coverage)
- Traceability (requirements → features → tasks → code)

## Installation

```bash
# From neat-workspace repository
./skills-link.sh neat-project install
```

This creates symlinks in `~/.claude/skills/` for all six skills.

## Skills

### Discovery

| Skill | Use when | Produces |
|-------|----------|----------|
| **neat-discovery-assessing** | Starting or extending discovery | `prd.md` (new) or `requirements{NN}.md` (iteration) |
| **neat-discovery-solutioning** | Requirements are scoped | `solution.md` (architecture, tech stack, risks) |
| **neat-discovery-modernizing** | Evaluating legacy codebase | `modernization.md` (health assessment, roadmap) |

### Delivery

| Skill | Use when | Produces |
|-------|----------|----------|
| **neat-delivery-planning** | Solution design is complete | `wave{NN}-{name}.md` (tasks in dependency order) |
| **neat-delivery-building** | Wave is planned | Verified code + `build-history.md` |
| **neat-delivery-fixing** | Debugging issues | Verified fix with tests |

## Quick Start

**New greenfield project:**
```
/neat-discovery-assessing   # → prd.md
/neat-discovery-solutioning # → solution.md
/neat-delivery-planning     # → wave files
/neat-delivery-building     # → code
```

**Add features to existing project:**
```
/neat-discovery-assessing   # → requirements01.md
/neat-discovery-solutioning # → solution.md (updated)
/neat-delivery-planning     # → wave files
/neat-delivery-building     # → code
```

**Modernize legacy system:**
```
/neat-discovery-modernizing # → modernization.md
/neat-discovery-solutioning # → solution.md
/neat-delivery-planning     # → wave files
/neat-delivery-building     # → code
```

## Artifact Structure

```
specs/
  discovery/
    prd.md                  # Product requirements (greenfield)
    requirements{NN}.md     # Iteration requirements (brownfield)
    solution.md             # Architecture and design
    modernization.md        # Modernization roadmap (legacy track)
  
  delivery/
    delivery.json           # Workspace state and wave tracking
    wave{NN}-{name}.md      # Features and tasks per wave
    plan.md                 # Cross-wave summary
    build-history.md        # Append-only build log
  
  system/
    system.md               # System knowledge base
    modules/{unit}.md       # Module-level details
    modules/index.md        # Module lookup table
```

## Documentation

Each skill has:
- `SKILL.md` — Phase-by-phase execution protocol
- `references/` — Domain-specific details loaded on-demand
- `references/neat-project/` — Global references shared across skills

## License

Apache License 2.0
