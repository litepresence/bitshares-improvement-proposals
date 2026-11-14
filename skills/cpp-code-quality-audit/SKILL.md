---
name: cpp-code-quality-audit
description: >
  Use when auditing C++ code quality in bitshares-core worktrees: formatting
  (clang-format), static analysis (clang-tidy, cppcheck), compiler warnings,
  include hygiene, naming conventions, dead code, and LLM-assisted read-through
  for logic smells. Vantage: deterministic, reviewable, warning-free C++.
version: 1.0.0
---

# Skill: cpp-code-quality-audit

Use when finishing or reviewing C++ changes in a bitshares-core worktree: before
declaring a feature complete, before a merge, or when the build emits warnings.
Combines modern terminal tooling with an LLM read-through. Emits a checklist
and a findings list, not a verdict.

## Tooling (terminal)

Run from the worktree root. Prefer these over ad-hoc grep.

1. **Formatting — clang-format.**
   - Note: BitShares core ships no `.clang-format`. Do not run clang-format
     with a default style; it will fight the project's hand-formatted style.
     Instead, check formatting by eye against neighbouring code and the
     project's implicit conventions (brace placement, 3-space indent).
   - If a `.clang-format` is later added, use `--style=file` and
     `--dry-run --Werror`.
2. **Static analysis — clang-tidy.**
   - Run on changed TUs: `clang-tidy <file> -- -std=c++14 -I...` with checks
     `cppcoreguidelines-*,modernize-*,performance-*,readability-*,bugprone-*`.
   - Pay special attention to: narrowing conversions, implicit int-to-float,
     use-after-move, dangling references, unchecked `new`.
3. **Static analysis — cppcheck.**
   - `cppcheck --enable=all --std=c++14 --suppress=missingInclude <dir>`.
   - Catches: null deref, memory leaks, out-of-bounds, unused functions.
4. **Compiler warnings.**
   - Build with `-Wall -Wextra -Wpedantic -Werror` on the changed TUs. Treat
     every warning as a finding. Common BitShares offenders: sign-compare,
     unused parameters, shadowing.
5. **Include hygiene.**
   - `include-what-you-use` (IWYU) where available: every header includes only
     what it uses; no transitive includes relied upon.
   - Check for circular includes and for `.cpp` files including `.cpp`.
6. **Dead code / unused symbols.**
   - `nm`/`objdump` or compiler `-Wunused` to find unused functions and
     variables. Remove or justify with a comment.
7. **Complexity / size.**
   - Flag functions over ~80 lines or cyclomatic complexity > 15 for split.
   - `lizard` for cyclomatic complexity and function length reports.

8. **Sanitizers (runtime).**
   - ASan (`-fsanitize=address`), UBSan (`-fsanitize=undefined`), TSan
     (`-fsanitize=thread`), MSan (`-fsanitize=memory`). Run the test suite
     under at least ASan+UBSan before declaring done. BitShares is multi-threaded
     (witness node), so TSan is valuable but noisy; scope to changed TUs.
9. **Dynamic analysis — valgrind.**
   - `valgrind --leak-check=full --track-origins=yes` on chain_test for
     changed paths. Slower than ASan but catches uninitialized reads ASan
     misses.
10. **Security-oriented static analysis.**
    - `flawfinder` and `rats` for dangerous C/C++ patterns (buffer ops, crypto
      misuse, TOCTOU). Especially relevant for crypto/consensus code.
11. **Clang static analyzer.**
    - `scan-build cmake --build ...` for path-sensitive bug finding (null
      deref, use-after-free, logic errors) beyond clang-tidy.
12. **Coverage.**
    - `gcov`/`lcov` or `llvm-cov` to verify new branches are exercised. Pair
      with test-vacuity-audit.
13. **Build/compile hygiene.**
    - `bear` or `compiledb` to regenerate `compile_commands.json` so
      clang-tidy/IWYU/clangd see the right flags. `cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON`.
    - `cmake-lint` / `cmake-format` for CMake files.
14. **Header guards.**
    - Verify every header has `#pragma once` or a unique include guard; check
      for duplicate guard macros across files.
15. **Documentation coverage.**
    - `doxygen` warnings for undocumented public APIs; pair with docs-standard.

## Efficiency, DRY, and stylistic consistency (LLM read-through)

Before the per-file checklist, assess the change as a whole:

1. **DRY.** No duplicated logic across the new code and existing code. If the
   same computation appears twice, extract a helper. If a new helper duplicates
   an existing utility in `libraries/`, use the existing one.
2. **Efficiency.**
   - No O(n^2) where O(n) or O(n log n) is available; no repeated linear
     scans inside loops.
   - Avoid unnecessary copies: pass large objects by `const&`, move on return,
     reserve vector capacity when size is known.
   - No redundant database lookups: cache `_pool`/`_share_asset` pointers rather
     than re-resolving per call.
   - Hot paths (block application, consensus evaluation) must not allocate on
     the heap or do I/O.
3. **Stylistic consistency.**
   - Match the surrounding code's control-flow style (brace placement, early
     returns vs nested ifs), error-reporting style (`FC_ASSERT` with messages),
     and naming. New code should be indistinguishable in style from
     neighbouring code.
   - Follow the project's file layout: evaluators in `*_evaluator.{hpp,cpp}`,
     objects in `*_object.hpp`, ops in protocol headers, hardfork in
     `hardfork.d/`.
   - Reuse existing patterns (e.g. `d.modify`, `FC_CAPTURE_AND_RETHROW`,
     `result.updated_objects.insert`) rather than inventing new ones.
   - Consistent comment style: `///<` for field docs, `//` for inline, section
     banners matching neighbouring files.

## LLM read-through (manual pass)

After tooling is clean, read each changed file top-to-bottom and check for:

1. **Naming.** Types `CamelCase`, functions/variables `snake_case`, members
   with trailing underscore only where project convention requires. No
   single-letter names outside loop indices.
2. **Const correctness.** Parameters and locals that are never mutated are
   `const`; member functions that don't mutate are `const`.
3. **Resource management.** RAII; no raw `new`/`delete`; no manual lock/unlock
   without scope guards; `std::unique_ptr`/`shared_ptr` preferred.
4. **Error handling.** Every `FC_ASSERT` has a message; no swallowed
   exceptions; no `catch(...)` without rethrow or logging.
5. **Integer safety.** No implicit narrowing; divisions checked for zero;
   multiplication before division where possible; signed/unsigned mixing flagged.
6. **Floats in consensus code.** Any `float`/`double`/`long double` in
   `libraries/chain` or `libraries/protocol` is a Critical finding (AGENTS.md
   rule). `long double` in math helpers must be justified and documented.
7. **Floats in consensus code.** (duplicate check, intentional emphasis.)
8. **Magic numbers.** Constants named and documented; no bare `100`, `10000`
   without a named constant where a domain meaning exists.
9. **Comment quality.** Every guard has a one-line "what this defends against";
   every supply-relevant mutation has an invariant statement; no ephemeral
   references (commit hashes, round numbers) except audit finding IDs with a
   one-line threat description. No "TODO"/"TBD" in committed code.
10. **Test coverage.** Every new branch has a test; no vacuous assertions;
    rounding tests favour the pool.

## Output

Produce a findings list grouped by severity (Critical / Important / Minor / Note)
with `file:line` citations, plus a pass/fail per check above. Do not fix code
in this skill; hand findings to the repair loop.