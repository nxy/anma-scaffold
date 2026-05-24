# Cross-Cutting Changes

How to handle changes that affect multiple modules.

## 1. Measure the blast radius

```bash
python3 tools/impact.py user-auth
```

This shows every module that consumes `user-auth` and would be affected by a contract change. Read the output before touching anything.

## 2. Publish a delta

Create a file in `BUS/deltas/` describing the change:

```yaml
source: user-auth
type: breaking-change
description: "verify_token output adds 'roles' field"
action_required: migrate
affected: [todo-api, notifications]
```

Set `action_required: migrate` so consuming modules know they need to update.

## 3. Plan the migration order

```bash
python3 tools/plan_migration.py user-auth
```

This reads GRAPH.yaml and outputs a safe update order — leaf modules first, working up the dependency tree. Follow the order exactly.

## 4. Update each module

For each affected module (in the order from step 3):

1. Update its `consumes` entry to reference the new contract version
2. Update TESTS.yaml to cover the changed interface
3. Run the linter: `python3 tools/lint_contracts.py`
4. Fix any errors before moving to the next module

## 5. Validate the full project

After all modules are updated:

```bash
python3 tools/lint_contracts.py --strict
```

Zero errors, zero warnings = migration complete. Archive the delta:

```bash
python3 tools/bus_archive.py
```
