# CLAUDE.md — ANMA Project Instructions

This project uses AI-Native Modular Architecture (ANMA).
Every module is defined by structured YAML contracts, not code.
Read contracts first. Never infer interfaces from source.

## Architecture

Modules: user-auth, todo-api, notifications

## Context Loading Order

On every task, read these files first (in order):

1. `CONVENTIONS.yaml` — universal rules all agents follow
2. `MANIFEST.yaml` — project modules and their status
3. `GRAPH.yaml` — dependency graph between modules
4. `modules/<module>/CONTRACT.yaml` — the interface spec
5. `modules/<module>/STATE.yaml` — current work and blockers
6. `modules/<module>/MEMORY.yaml` — accumulated knowledge

Do NOT skip this loading order. Do NOT read source code before contracts.

## Rules

### Naming

- modules: `kebab-case`
- interfaces: `snake_case`
- errors: `SCREAMING_SNAKE_CASE`

### Communication

- Never edit another module's files — use BUS/requests
- CONTRACT.yaml is truth — never infer interfaces from source
- Unmet needs -> file BUS/request, don't work around it

### Memory Management

- Max 20 entries in MEMORY.yaml
- Each entry under 100 characters
- Curate, don't append — delete stale entries before adding
- One line per entry, under 100 characters
- Decisions supersede discoveries on the same topic
- Remove warnings when resolved

### Error Format

- All errors: `{ code: STRING_CONSTANT, message: string, details: object | null }`

### Contract Status Values

Valid: draft, stable, frozen, breaking-change, deprecated

## Inter-Module Communication

- Never edit another module's files directly
- Use `BUS/requests/` to request changes from other modules
- Publish contract changes to `BUS/deltas/`
- Read BUS files relevant to your module on every task

## Linting

Run `python3 tools/lint_contracts.py` before committing any change.
Run `python3 tools/lint_contracts.py --strict` for zero-warning builds.

## Adding a Module

Use the scaffolding script:
```
python3 tools/new_module.py <name> --manager <manager> --consumes <deps>
```

## Key Principle

Design for replacement, not continuity. Any fresh agent with zero history
can take over any module by reading its 6 files. If knowledge exists only
in your context, write it to MEMORY.yaml or CHANGELOG.yaml.
