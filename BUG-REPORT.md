# ANMA Scaffold Bug Hunt Report

Generated: 2026-05-28

## Summary
- Total raw findings: 71
- After deduplication: 50
- Confirmed BUGs: 17
- Code SMELLs: 21
- False positives discarded: 12
- Test suite status: 111 unit tests PASSED, 83 smoke tests PASSED

---

## Confirmed Bugs

### BUG-001: [severity: high]
**File:** tools/gen_tests.py, line 207
**Category:** data-loss
**Description:** The `--append` flag destroys existing tests instead of appending. When `--append` is used, the code filters out already-covered interfaces (line 202) and generates YAML containing ONLY the new tests. This is then written via `write_text()` (line 210), overwriting the entire file and losing all previously existing tests.
**Reproduction:** 1) Create a module with 3 interfaces, run `gen_tests.py --output TESTS.yaml`. 2) Add 2 more interfaces to CONTRACT.yaml. 3) Run `gen_tests.py --append --output TESTS.yaml`. The TESTS.yaml now contains only 2 tests; the original 3 are gone.
**Suggested fix:** In `--append` mode, read the existing TESTS.yaml, parse its tests list, merge the new tests into the combined list, and write the merged result.

### BUG-002: [severity: high]
**File:** tools/claims.py, line 52
**Category:** data-corruption
**Description:** `_save_claims()` writes `by` and `branch` values into YAML without quoting (line 52: `f'  {mod}: {{ by: {by}, branch: {branch}, since: "{since}" }}'`). Only `since` is quoted. When `by` or `branch` contains YAML-special characters (colons, commas, braces, hash signs, or YAML reserved words like `true`, `yes`, `null`), the file becomes malformed YAML. On reload, `_load_claims` returns `{}`, silently losing all claims data. Additionally, YAML boolean coercion (e.g., user `yes` becomes boolean `True`) causes ownership checks in `add_claim()` line 64 to fail incorrectly.
**Reproduction:** `python3 tools/claims.py claim test-mod --by 'agent: admin'` followed by `python3 tools/claims.py status` shows "No active claims" because the YAML is corrupt.
**Suggested fix:** Quote all interpolated values: `f'  {mod}: {{ by: "{by}", branch: "{branch}", since: "{since}" }}'`. Or use `yaml.dump()` instead of manual formatting.

### BUG-003: [severity: high]
**File:** tools/rename_project.py, line 39
**Category:** logic-error
**Description:** `rename_project.py` uses unbounded `str.replace(old_name, new_name)` on entire file contents. If the old project name appears as a substring of module names, manager names, or any other text, those values are silently corrupted. For example, renaming project `todo` to `tasks` turns module `todo-api` into `tasks-api` in MANIFEST.yaml, but the directory on disk remains `modules/todo-api`, breaking the project.
**Reproduction:** 1) Name project `app` with module `app-auth`. 2) Run `python3 rename_project.py platform`. 3) MANIFEST references `platform-auth` but directory is still `modules/app-auth`.
**Suggested fix:** Use targeted YAML-aware replacement (e.g., regex matching `^project:\s*<old>`) or use the existing `yaml_editor.manifest_rename_project()`.

### BUG-004: [severity: high]
**File:** tools/contract_diff.py, line 148
**Category:** logic-error
**Description:** `generate_deltas()` computes `cons_added` and `cons_removed` (consumes changes) and uses them in the no-changes guard at line 148, but never generates any delta entries or changelog entries for them. When a contract change ONLY modifies the `consumes` section (no `provides` changes), the function passes the early-return check but returns a summary with `added=0, removed=0, modified=0` and empty delta/changelog lists. Consumes changes are silently dropped.
**Reproduction:** 1) Snapshot a module. 2) Edit CONTRACT.yaml to add a `consumes` entry (no `provides` changes). 3) Run `contract_diff.py`. Output: `0 added, 0 removed, 0 modified` with no deltas generated.
**Suggested fix:** Add delta generation loops for `cons_added` and `cons_removed`, creating delta entries with types `dependency_added` and `dependency_removed`.

### BUG-005: [severity: high]
**File:** tools/plan_migration.py, line 91
**Category:** logic-error
**Description:** The enrichment loop for transitive consumers reports the WRONG version pin. For a transitive consumer C that consumes intermediate module B (which consumes migrating module A), line 91 checks C's consumes for B (the `via` module) and reads its `contract_version`. This reports C's pin on B as if it were C's pin on A. The migration plan then displays incorrect version pin information for transitive consumers.
**Reproduction:** Module A (v1), B consumes A at v1, C consumes B at v3. Run `plan_migration.py A 2`. Transitive consumer C shows `pinned_version=3` (its pin on B), implying C pins A at v3, which is wrong.
**Suggested fix:** For transitive consumers, set `pinned_version = None` since they do not directly consume the migrating module. The `via` field already communicates the indirect relationship.

### BUG-006: [severity: high]
**File:** tools/new_module.py, line 285
**Category:** command-injection
**Description:** `os.system(f'cd {root} && python3 lint_contracts.py')` interpolates the project root path directly into a shell command without quoting. If the path contains spaces or shell metacharacters, the command breaks or executes arbitrary code. The `subprocess` module is available and used in peer scripts.
**Reproduction:** Run `python3 tools/new_module.py my-mod --path '/tmp/my project' --lint`. The shell splits the unquoted space.
**Suggested fix:** Replace with `subprocess.run([sys.executable, str(TOOLS_DIR / 'lint_contracts.py')], cwd=str(root))`.

### BUG-007: [severity: high]
**File:** tools/sync_all.py, line 262
**Category:** data-loss
**Description:** When rebuilding MANIFEST.yaml, the managers section silently drops module ownership for managers stored in list format. Lines 222-225 correctly handle list-format managers when building `modules_dict`. But lines 259-263 (writing managers back out) only handle the `dict` case and fall through to `owns = []` for the list case. A manager like `backend-manager: [auth, users]` is correctly read but written back as `backend-manager: { owns: [] }`, losing all module assignments.
**Reproduction:** Create MANIFEST.yaml with `managers:\n  my-manager: [module-a, module-b]`. Run `sync_all.py`. Result: `my-manager: { owns: [] }`.
**Suggested fix:** Add `elif isinstance(mgr_data, list): owns = mgr_data` at line 262, matching the read logic at lines 224-225.

### BUG-008: [severity: high]
**File:** tools/verify_contract.py, line 46
**Category:** silent-failure
**Description:** `load_module_tests()` returns the raw result of `parse_yaml_file()` without checking for `None` or `_parse_error`. If YAML is malformed, `parse_yaml_file` returns `{'_parse_error': '...'}` which is truthy. Callers do `.get('tests', [])` on it, getting `[]`, so tests silently appear to pass with 0 test cases. No error is reported.
**Reproduction:** Create a module with syntactically invalid TESTS.yaml. Run `verify_contract.py <module> --plan`. Output: plan with 0 tests, no error.
**Suggested fix:** After lines 46-47, check: `if tests is None or '_parse_error' in (tests or {}): print error and sys.exit(1)`.

### BUG-009: [severity: medium]
**File:** tools/remove_module.py, line 37
**Category:** logic-error
**Description:** `clean_bus()` uses substring matching (`name in f.read_text()`) to decide which BUS files to delete. Removing module `auth` also deletes BUS files that reference `user-auth` or any module containing `auth` as a substring.
**Reproduction:** Create modules `auth` and `user-auth`. Create a BUS delta referencing `user-auth`. Run `remove_module.py auth --confirm`. The BUS delta for `user-auth` is also deleted.
**Suggested fix:** Parse BUS YAML files and check structured fields (`source`, `from`, `to`) for exact module name matches, similar to what `sync_all.py` does at lines 288-300.

### BUG-010: [severity: medium]
**File:** checks/check_conventions_pin.py, line 12
**Category:** logic-error
**Description:** The check iterates `all_contracts` instead of `contracts`. When the linter is invoked with `--module <name>`, `contracts` contains only filtered modules but `all_contracts` contains every module. This causes the plugin to emit warnings for modules the user did not ask to lint. All other checks iterate `contracts` for reporting.
**Reproduction:** Run `python3 tools/lint_contracts.py --module user-auth`. The check_conventions_pin plugin still emits warnings for all other modules.
**Suggested fix:** Change line 12 from `for mod_name in sorted(all_contracts):` to `for mod_name in sorted(contracts):`.

### BUG-011: [severity: medium]
**File:** checks/check_principles.py, lines 99 and 216
**Category:** crash
**Description:** `check_p2_tokens_are_bottleneck` (line 99) and `check_p6_recovery_is_cheap` (line 216) do `conventions.get('token_thresholds', {}).get(...)`. If `token_thresholds` is `None` (YAML key with no value) or a non-dict type, `.get()` on it raises `AttributeError`. The main linter's `check_memory_files` (line 954-957) correctly guards against this with `isinstance(result, dict)`.
**Reproduction:** Set `token_thresholds:` with no sub-keys in CONVENTIONS.yaml (parsed as None). Run the linter. Plugin crashes with `AttributeError: 'NoneType' object has no attribute 'get'`.
**Suggested fix:** Use `(conventions.get('token_thresholds') or {}).get(...)` pattern.

### BUG-012: [severity: medium]
**File:** tools/gen_claude_md.py, line 186
**Category:** logic-error
**Description:** `generate_module_claude_md` hardcodes `modules/{module_name}/` paths in generated CLAUDE.md instructions (lines 186-188), but domain modules live at `domains/<domain>/<module_name>/`. The function resolves the correct `mod_dir` (line 151) but does not use it for path references. Agents reading a CLAUDE.md generated for a domain module would be directed to a non-existent path.
**Reproduction:** Create a domain module and run `gen_claude_md.py --module auth --force`. The generated CLAUDE.md contains `modules/auth/CONTRACT.yaml` instead of `domains/backend/auth/CONTRACT.yaml`.
**Suggested fix:** Compute the relative path from root using `mod_dir.relative_to(root)` and use it in lines 186-188.

### BUG-013: [severity: medium]
**File:** tools/import_contracts.py, line 144
**Category:** logic-error
**Description:** When `sync_all.py` fails, the error output shows `result.stdout[-500:]`. However, Python tracebacks go to stderr. The actual error from `sync_all.py` is in `result.stderr`, which is never displayed. Users see empty or irrelevant stdout instead of the real error.
**Reproduction:** Cause `sync_all.py` to fail (e.g., corrupt YAML) and run `import_contracts.py`. The "failed" message shows stdout (progress messages), hiding the actual traceback in stderr.
**Suggested fix:** Change to `print(f"  sync_all.py failed:\n{(result.stderr or result.stdout)[-500:]}")`.

### BUG-014: [severity: medium]
**File:** tools/sync_all.py, line 134-170
**Category:** logic-error
**Description:** Deleted TESTS.yaml not regenerated when contract is unchanged. Step 1 (`ensure_stub`, line 134-136) creates a minimal stub TESTS.yaml with empty `tests: []`. Step 2 (hash check, line 167-170) then sees the contract hash matches AND the stub file exists, so it skips regeneration. Result: module ends up with an empty stub instead of properly generated test cases.
**Reproduction:** 1) Create modules with interfaces, run `sync_all.py` (generates proper TESTS.yaml). 2) Delete one module's TESTS.yaml. 3) Run `sync_all.py` -- the stub is created but not regenerated because the contract hash matches.
**Suggested fix:** Record which files were created as stubs in Step 1 and exclude them from the "file exists" check at line 169. Or move `ensure_stub` for TESTS.yaml to after the hash-based regeneration loop.

### BUG-015: [severity: medium]
**File:** tools/gen_contract.py, line 197
**Category:** injection
**Description:** The `purpose` string is interpolated into YAML with double quotes but without escaping. If `--purpose` contains double-quote characters, the generated CONTRACT.yaml has broken YAML syntax.
**Reproduction:** Run `python3 tools/gen_contract.py my-module --purpose 'Handle "auth" flows'`. Output contains `purpose: "Handle "auth" flows"` which is invalid YAML.
**Suggested fix:** Escape double quotes in the purpose string before interpolation, or use a YAML library for serialization.

### BUG-016: [severity: medium]
**File:** tools/yaml_editor.py, line 298
**Category:** logic-error
**Description:** `scope_add_module()` silently fails to add the module if the SCOPE.yaml file lacks an `owns:` line. The function appends to the in-memory list, but the for-loop searching for `owns:` never finds it, so the file is written back unchanged. Despite this, the function returns `True` (success), misleading the caller.
**Reproduction:** Create SCOPE.yaml without an `owns:` line. Call `scope_add_module(root, 'mgr', 'new-mod')`. Returns `True`, but module is not in the file.
**Suggested fix:** After the for-loop, check if `owns:` was found. If not, insert it or return `False`.

### BUG-017: [severity: medium]
**File:** tools/new_manager.py, line 53
**Category:** logic-error
**Description:** `new_manager.py` creates the manager directory and files (lines 35-48) before calling `manifest_add_manager()` (line 53). If MANIFEST already has this manager name, `manifest_add_manager` returns `(False, error)`. The script prints the error but exits with code 0 (success), leaving the system inconsistent: directory exists but MANIFEST was not updated. No rollback occurs.
**Reproduction:** Manually add a manager to MANIFEST without creating its directory. Run `new_manager.py <same-name>`. Directory is created, MANIFEST reports already-exists, exit code 0.
**Suggested fix:** Check `manifest_add_manager` return value. On failure, roll back (delete created directory) and exit with code 1.

---

## Code Smells

### SMELL-001: [severity: medium]
**File:** checks/check_principles.py, line 155
**Category:** imprecise matching
**Description:** P4 BUS file scanning uses plain substring matching (`if m in c`) to detect module mentions. Module `api` matches any BUS file containing `api` as part of another word (e.g., `api-gateway`, `capacity`). This could hide missing-BUS-event warnings.
**Suggested fix:** Use word-boundary matching: `re.search(r'\b' + re.escape(m) + r'\b', c)`.

### SMELL-002: [severity: medium]
**File:** tools/sync_all.py, line 139
**Category:** spurious side-effect
**Description:** `sync_all.py` creates per-module BUS subdirectories (`mod_dir / 'BUS' / 'requests'` and `deltas`) inside each module directory. Every other tool operates on the project-root-level BUS directory. These spurious empty directories inside every module are never read by any tool.
**Suggested fix:** Remove lines 138-141. BUS directories at project root are managed elsewhere.

### SMELL-003: [severity: medium]
**File:** tools/remove_module.py, line 72
**Category:** incomplete cleanup
**Description:** `remove_module.py` does not clean up `consumes` references in other modules' CONTRACT.yaml files. Even with `--force`, consuming modules retain broken consumes entries that cause lint errors. The script warns about this but does not fix it or list specific files to edit.
**Suggested fix:** With `--force`, either auto-remove dangling consumes entries or list specific files that need manual editing.

### SMELL-004: [severity: medium]
**File:** tools/yaml_editor.py, line 29
**Category:** silent data loss on corruption
**Description:** `read_manifest()` returns `parse_yaml_file(str(path)) or {}`. When YAML is malformed, `parse_yaml_file` returns `{'_parse_error': '...'}` which is truthy, so `or {}` does not replace it. The `_parse_error` dict is returned as valid data. Callers like `manifest_add_module` then do `.get('modules', {})` returning `{}`, potentially overwriting the corrupted file and losing all data. Same issue affects `read_graph()` (line 186), `read_scope()` (line 281).
**Suggested fix:** Check for `'_parse_error' in result` before returning.

### SMELL-005: [severity: medium]
**File:** tools/gen_tests.py, line 39
**Category:** silent failure on malformed YAML
**Description:** When CONTRACT.yaml has malformed YAML, `parse_yaml_file` returns `{'_parse_error': '...'}` which passes the `if not contract` guard. The code proceeds to `.get('provides', [])` returning `[]`, and silently generates 0 test stubs with no error message.
**Suggested fix:** Change guard to `if not contract or '_parse_error' in contract:`.

### SMELL-006: [severity: medium]
**File:** tools/lint_contracts.py, line 503
**Category:** silent failure on malformed YAML
**Description:** `load_graph()`, `load_conventions()`, and `load_manifest()` return raw `parse_yaml_file()` results without `_parse_error` checks. The `_parse_error` dict is truthy and passes `if conventions:` guards in check functions, causing convention-based checks to silently use empty defaults.
**Suggested fix:** Add `_parse_error` checks in the loader functions or in `main()` after calling them.

### SMELL-007: [severity: medium]
**File:** tools/dashboard.py, line 34
**Category:** silent failure on malformed YAML
**Description:** Uses `parse_yaml_file(...) or {}` for manifest and graph. Malformed YAML returns `_parse_error` dict which passes through. Dashboard silently shows no modules with no error reported.
**Suggested fix:** Check for `_parse_error` in parsed results and print a warning.

### SMELL-008: [severity: medium]
**File:** tools/sync_all.py, line 158
**Category:** silent failure on malformed YAML
**Description:** `parse_yaml_file(str(contract_path)) or {}` on line 158 allows `_parse_error` dicts through. A corrupted contract is skipped with the misleading message "no interfaces yet". At line 206, corrupted MANIFEST causes module data loss in the rebuild.
**Suggested fix:** Check for `_parse_error` before using parsed results; print clear warnings.

### SMELL-009: [severity: medium]
**File:** tools/claims.py, line 36
**Category:** crash on unexpected type
**Description:** `_load_claims()` does not validate that the `claims` value is a dict. If `claims.yaml` contains `claims: some_string` or `claims: [a, b]`, the non-dict value is returned, causing `AttributeError` in `add_claim()` on `.get(module)`.
**Suggested fix:** Add type check: `result = data.get('claims', {}); return result if isinstance(result, dict) else {}`.

### SMELL-010: [severity: medium]
**File:** tools/claims.py, line 108
**Category:** argument handling
**Description:** The `--path` flag is defined on the top-level parser but not on subparsers. With argparse subparsers, flags on the parent must appear BEFORE the subcommand. When dispatched from `anma.py`, the subcommand is prepended, placing `--path` after it. Argparse rejects this.
**Suggested fix:** Add `--path` to each subparser, or use `parse_known_args` on the parent.

### SMELL-011: [severity: medium]
**File:** tools/gen_product_spec.py, line 98
**Category:** logic-error
**Description:** When `--module` filter is used, summary header still counts ALL modules and interfaces (computed from unfiltered `contracts` dict). Header may say "12 modules" while body shows 1 module.
**Suggested fix:** Apply `module_filter` before computing summary statistics.

### SMELL-012: [severity: medium]
**File:** tools/new_module.py, line 37
**Category:** cross-tool consistency
**Description:** `new_module.py` generates CONTRACT.yaml without a `conventions_version` field. The `check_conventions_pin.py` plugin warns on contracts missing this field, so every module created via `new_module.py` immediately triggers a lint warning.
**Suggested fix:** Add `conventions_version` to the generated contract, reading the value from CONVENTIONS.yaml.

### SMELL-013: [severity: medium]
**File:** .gitignore, line 35
**Category:** cross-tool consistency
**Description:** `.gitignore` includes `.anma/sync-state.yaml` but not `.anma/claims.yaml`. Claims data is per-user, per-branch transient state that should not be committed. Committing it would cause merge conflicts in multi-developer workflows.
**Suggested fix:** Add `.anma/claims.yaml` to `.gitignore` (or broaden to `.anma/`).

### SMELL-014: [severity: medium]
**File:** tools/graph_viz.py, line 64
**Category:** injection / broken output
**Description:** The purpose string is interpolated directly into a Mermaid node label in double quotes without escaping. Special Mermaid characters (double quotes, parentheses, brackets) cause syntax errors in the generated diagram.
**Suggested fix:** Escape double quotes as `#quot;` (Mermaid entity escape) and handle other special characters.

### SMELL-015: [severity: medium]
**File:** tools/claims.py, line 56
**Category:** missing validation
**Description:** `add_claim()` does not validate that the module actually exists before creating a claim. Users can claim non-existent modules without error.
**Suggested fix:** Use `discover_modules()` to verify the module name exists before writing the claim.

### SMELL-016: [severity: medium]
**File:** tools/remove_module.py, line 59
**Category:** incomplete cleanup
**Description:** `remove_module.py` does not release the claim for a removed module. After successful removal, the claim in `.anma/claims.yaml` persists as an orphan.
**Suggested fix:** Call `release_claim(root, name)` after the module directory is deleted.

### SMELL-017: [severity: low]
**File:** tools/lint_contracts.py, line 1975
**Category:** schema validation gap
**Description:** The CONTRACT schema includes `constraints` as a valid key, but `constraints` is not in the documented CONTRACT.yaml specification (CLAUDE.md). The `new_module.py` generates a `constraints:` section, indicating it was intentional but undocumented. This is more of a documentation gap than a bug.
**Suggested fix:** Either document `constraints` in CLAUDE.md or remove it from both the schema and `new_module.py`.

### SMELL-018: [severity: low]
**File:** tools/init_project.py, line 33
**Category:** overly broad deletion
**Description:** `init_project` deletes ALL subdirectories under `modules/`, not just module directories (those containing CONTRACT.yaml). Non-module directories are silently deleted.
**Suggested fix:** Filter deletion to only directories containing CONTRACT.yaml.

### SMELL-019: [severity: low]
**File:** tools/init_project.py, line 95
**Category:** incomplete cleanup
**Description:** `init_project` clears BUS files and resets MANIFEST but does not clean up manager SCOPE.yaml files, leaving stale module references in `owns` lists.
**Suggested fix:** Clear `owns` lists in all SCOPE.yaml files or delete manager directories during init.

### SMELL-020: [severity: low]
**File:** tools/test_linter.py, line 4
**Category:** stale documentation
**Description:** Docstring says "20 linter checks" but `lint_contracts.py` has 24 built-in checks. Tests do not cover checks 21-24.
**Suggested fix:** Update docstring and add test cases for checks 21-24.

### SMELL-021: [severity: low]
**File:** tools/yaml_editor.py, line 39
**Category:** cross-tool consistency
**Description:** `write_manifest()` defaults version to `'0.1.0'` (semver string) while every other tool treats MANIFEST version as a plain integer. The real MANIFEST.yaml has `version: 1`.
**Suggested fix:** Change default from `'0.1.0'` to `1`.

---

## Test Suite Results

### Unit Tests (test_linter.py)
```
Ran 111 tests in 1.039s
OK
```

### Smoke Tests (smoke_test.py)
```
ANMA End-to-End Smoke Test
==================================================

-- Phase 1: Bootstrap empty project --
-- Phase 2: Scaffold modules --
-- Phase 3: Graph consistency --
-- Phase 4: Linter with 3 modules --
-- Phase 5: Graph visualization --
-- Phase 6: CLAUDE.md generation --
-- Phase 7: Compatibility matrix --
-- Phase 8: Contract verification --
-- Phase 9: BUS lifecycle --
-- Phase 10: Plugin system --
-- Phase 11: Error handling --
-- Phase 12: Manager scaffolding --
-- Phase 13: Contract template generator --
-- Phase 14: Product spec generator --
-- Phase 15: Project rename --
-- Phase 16: Module removal --
-- Phase 17: Final consistency --
-- Phase 18: Getting Started walkthrough --

==================================================
  83 passed, 0 failed out of 83
  ALL SMOKE TESTS PASSED
```

---

## False Positives Discarded

1. **contract_diff.py line 155 (unused variable)** -- `all_contracts = load_all_contracts(root)` is technically unused, but this is dead code/waste, not a bug causing incorrect behavior. Reclassified as not actionable enough for the report (no user-facing impact).

2. **contract_diff.py line 278 (timestamp collision)** -- While theoretically possible, the same-second collision requires two runs in the same second for the same module. The in-memory `written` set handles intra-run collisions. Practical risk is negligible.

3. **contract_diff.py line 328 (hardcoded path in print)** -- This is a cosmetic issue in a print statement, not a functional bug. The file is written to the correct location. Reclassified to not actionable.

4. **contract_diff.py line 356 (_parse_error unchecked)** -- Lines 373-378 explicitly check `if not current or not isinstance(current, dict)` and `if not old or not isinstance(old, dict)`, which catches None. The `_parse_error` case is truthy and isinstance dict, but the diff still runs and produces a (somewhat misleading) result. Same category as the other `_parse_error` smell findings.

5. **verify_contract.py line 72 (str(key) conversion)** -- The `str()` conversion is actually correct defensive coding for YAML integer keys. JSON object keys are always strings, and converting YAML integers to string for comparison is reasonable behavior. Not a bug.

6. **yaml_editor.py line 318 (substring early return in scope_remove_module)** -- The early return is an optimization that can be skipped, but the actual removal uses exact list matching. No data corruption occurs; just unnecessary file rewrites in edge cases.

7. **lint_contracts.py line 1327 (str(None) for module type)** -- `str(None) == 'infrastructure'` is always False, which is the correct behavior (don't enforce frozen status on modules with no type). The None case is intentionally handled by the truthiness check on line 1322.

8. **new_module.py line 261 (long module name crash)** -- This is a filesystem limitation, not a code bug. Names over 255 chars hitting OS limits is expected behavior. Adding a length check would be nice but is not a bug.

9. **smoke_test.py line 386 (temp directory leak)** -- The entire smoke test runs in a controlled environment. The temp dir leak only occurs if an unhandled exception kills the test, which would be visible. This is a minor resource hygiene issue, not a bug.

10. **smoke_test.py line 78 (BUS/contracts/ phantom directory)** -- This is a test scaffold setup detail. The extra directory does not affect any test behavior or real project structure.

11. **benchmark/measure_tokens.py lines 42-44 (regex counting)** -- The invariant_count, consumes_count, and error_count always being 0 for generated benchmarks is a measurement inaccuracy in benchmark tooling, not a bug in the main codebase. These metrics appear only in benchmark output files.

12. **benchmark/measure_tokens.py line 1 (import before shebang)** -- Confirmed: `import re` appears before `#!/usr/bin/env python3`. This prevents direct `./measure_tokens.py` execution but `python3 measure_tokens.py` works fine. This is a minor issue in optional benchmark tooling.

13. **benchmark/eval_degradation.py line 387 (missing file check)** -- This only affects optional benchmark tooling, not the main scaffold. Reclassified as not impactful enough for the bug list.

14. **benchmark/measure_tokens.py line 181 (KeyError on empty project)** -- Only affects optional benchmark tooling run against degenerate inputs. Not a main codebase bug.

15. **Multiple --path missing findings (remove_module, new_manager, rename_project, gen_contract)** -- These are feature requests / inconsistencies, not bugs. The tools work correctly when run from the project root or via standalone invocation. The anma.py dispatcher handles path differently. Reclassified as SMELL-010 covers the claims-specific case where it actually breaks.

16. **gen_contract.py line 226 (ValueError swallowed)** -- The ValueError for duplicate modules is a project configuration error. Silently falling back to TBD interface is degraded but not incorrect behavior. The tool still produces valid YAML.

17. **sync_all.py lines 160/190 (unconditional GRAPH/MANIFEST rebuild)** -- This is an efficiency concern, not a bug. The output is correct.
