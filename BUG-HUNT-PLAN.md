# ANMA Scaffold Bug Hunt — Dynamic Workflow Plan

## Codebase: 34 Python files, 12,175 lines, 111 unit tests, 83 smoke tests

---

## WAVE 1: Per-Group Static Analysis (7 parallel subagents)

Each subagent reads its assigned files and reports:
- Logic errors, wrong conditions, off-by-one
- Missing error handling or swallowed exceptions
- Hardcoded paths that should use discover_modules .get() pattern
- Functions that accept root but don't convert with Path(root)
- Arguments that don't match the function signature
- Dead code or unreachable branches

### Agent 1: Core Linter (3 files, ~2800 lines)
- tools/lint_contracts.py
- checks/check_principles.py
- checks/check_conventions_pin.py

### Agent 2: Module Lifecycle (4 files)
- tools/new_module.py
- tools/remove_module.py
- tools/import_contracts.py
- tools/init_project.py

### Agent 3: Sync & Generation (6 files)
- tools/sync_all.py (incremental hash logic, --force, --regenerate-only)
- tools/gen_graph.py
- tools/gen_tests.py
- tools/gen_contract.py
- tools/gen_claude_md.py
- tools/gen_product_spec.py

### Agent 4: Discovery & Claims (2 files)
- tools/discover.py (edge cases: symlinks, empty dirs, duplicate names)
- tools/claims.py (Path(root) conversion, concurrent access, YAML round-trip)

### Agent 5: Analysis Tools (7 files)
- tools/dashboard.py
- tools/compat_matrix.py
- tools/contract_diff.py
- tools/impact.py
- tools/verify_contract.py
- tools/graph_viz.py
- tools/plan_migration.py

### Agent 6: Infrastructure (8 files)
- tools/anma.py (dispatch correctness for all commands)
- tools/yaml_editor.py
- tools/smoke_test.py
- tools/test_linter.py
- tools/session_log.py
- tools/rename_project.py
- tools/bus_archive.py
- tools/new_manager.py

### Agent 7: Benchmark (4 files)
- tools/benchmark/bench_scaling.py
- tools/benchmark/generate_archetypes.py
- tools/benchmark/measure_tokens.py
- tools/benchmark/eval_degradation.py

---

## WAVE 2: Cross-Cutting Analysis (5 parallel subagents)

Each subagent checks ONE bug category across ALL files.

### Agent 8: Path Safety Audit
Scan every file for:
- `root / 'modules' / mod_name` without .get() fallback
- Functions that accept root parameter but don't do Path(root)
- Hardcoded `'modules/'` strings that miss domain modules
- Paths that assume flat layout (no domains/ support)

### Agent 9: Argument & CLI Audit
For every tool with argparse:
- Does it have a main() function? (anma.py requires it)
- Do --path, --force, --confirm flags work as documented?
- Does anma.py dispatch pass the right args to each tool?
- Are help strings accurate (check counts, file counts)?

### Agent 10: YAML Parsing Audit
For every call to parse_yaml_file():
- Is the return value checked for None?
- Is '_parse_error' handled?
- Does the code crash on empty YAML files?
- Does the code handle missing keys with .get() not []?

### Agent 11: Cross-Tool Consistency Audit
Check that tools agree on:
- Module names in example contracts vs MANIFEST vs GRAPH
- conventions_version: 3 everywhere
- Tool counts in anma.py help, ARCHITECTURE.md, README
- Claims.yaml tracked, sync-state.yaml ignored
- All tools that import discover_modules use .get() pattern

### Agent 12: Edge Case Audit
Test each tool mentally with:
- Empty project (no modules, no domains)
- 0 interfaces in a contract
- Module with no consumes
- Domain with no GATEWAY.yaml
- .anma/ directory missing
- MANIFEST.yaml or GRAPH.yaml missing
- Module name with unusual characters

---

## WAVE 3: Runtime Verification (4 parallel subagents)

Each subagent creates a temp project and RUNS the tools.

### Agent 13: Happy Path
- Create 10-module project with 2 domains
- Run every tool: init, new_module, sync_all, lint, gen_graph, dashboard,
  compat_matrix, contract_diff, impact, verify_contract, graph_viz
- All should exit 0 with no errors

### Agent 14: Edge Cases
- Empty project: init, then lint (should exit 1 cleanly, not crash)
- Module with 0 interfaces: sync_all, lint (should warn, not crash)
- Missing GATEWAY.yaml with cross-domain dep (lint should error, not crash)
- Delete MANIFEST.yaml, run sync_all (should recreate or error cleanly)
- Delete GRAPH.yaml, run gen_graph (should recreate)

### Agent 15: Claims Integration
- Claim a module as user-a
- Try remove_module as user-b (should warn)
- Try remove_module with --force as user-b (should succeed)
- Claim, release, status, clear cycle
- Claims with no .anma/ directory

### Agent 16: Incremental Sync Verification
- sync_all first run (no state) — all regenerate
- sync_all second run — all skip
- Change 1 contract — only that regenerates
- Change gen_tests.py — all regenerate with message
- --force — all regenerate
- --regenerate-only — skip TESTS entirely

---

## WAVE 4: Verification & Report (1 agent)

### Agent 17: Deduplicate and Verify
- Collect all findings from Waves 1-3
- Remove duplicates
- For each finding: can it be reproduced?
- Classify: BUG (confirmed), SMELL (code quality), FALSE POSITIVE (discard)
- Run python3 -m unittest tools.test_linter — still 111/111?
- Run python3 tools/smoke_test.py — still 83/83?
- Produce final report with severity and suggested fix for each BUG

---

## Expected Output Format

For each confirmed bug:
```
BUG-001: [severity: critical|high|medium|low]
File: tools/example.py, line 42
Description: function foo() crashes when passed empty dict
Reproduction: python3 -c "from example import foo; foo({})"
Suggested fix: Add `if not data: return` guard
```
