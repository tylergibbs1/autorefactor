---
name: auto-refactor
description: >-
  Autonomous code refactoring loop for Claude Code. Point it at a codebase, define
  quality metrics (test coverage, cyclomatic complexity, bundle size, type safety,
  lint warnings), and let it loop indefinitely. Each iteration picks a file, refactors
  it, runs the test suite, checks if the metric improved, and keeps or reverts the
  change. Use this skill whenever the user says "auto-refactor", "refactor loop",
  "clean up my codebase", "improve code quality", "reduce complexity", or wants to
  run an automated refactoring agent. Also triggers on requests to iteratively improve
  code health metrics like coverage, complexity scores, lint counts, or bundle size.
  Works in any project with a test suite and a measurable quality metric.
license: MIT
metadata:
  author: tylergibbs
  version: "1.0.0"
  argument-hint: "[config-file]"
---

# Auto-Refactor

Autonomous refactoring agent. Runs an infinite improve loop: pick a file, refactor
it, run tests + measure a quality metric, keep improvements, discard failures.

Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch)
and [tylergibbs1/autoresearchskill](https://github.com/tylergibbs1/autoresearchskill),
adapted for code quality instead of ML metrics.

The core contract: **every change either improves the codebase or gets thrown away.**
Tests must pass and the metric must not regress.

## Discovery Flow

If `refactor.json` exists, skip to [Setup Phase](#setup-phase).

Otherwise:

1. **Explore the project with 2 parallel subagents** — launch both simultaneously
   using the `Agent` tool:

   **Agent 1 — Structure & Stack**: Scan directory tree, identify language/framework/
   build system, read README and top-level config files (package.json, pyproject.toml,
   Cargo.toml, etc.), summarize the project's purpose and tech stack.

   **Agent 2 — Metrics & Quality Tools**: Search for existing test suites, linters,
   type checkers, coverage tools, and quality metrics. Look for files with "test",
   "spec", "lint", "coverage" in the name. Read configs (.eslintrc, tsconfig.json,
   pyproject.toml, etc.) and report what quality tools are configured and how to run them.

   Wait for both agents to return before proceeding.

2. **Present findings and ask one question** (use `AskUserQuestion` if available):

   > I've explored your project. Here's what I found:
   > - [brief project summary from Agent 1]
   > - [existing quality tools/metrics from Agent 2]
   >
   > **What do you want to optimize?**
   > 1. Lint warnings ([N] current)
   > 2. Test coverage (currently [X]%)
   > 3. Type errors ([N] current)
   > 4. Cyclomatic complexity
   > 5. Custom metric

   Infer everything else from the project exploration:
   - `target_files`: from the user's answer + project structure
   - `exclude_files`: tests, generated code, vendor, node_modules
   - `test_command`: from build system / existing test scripts
   - `metric_command`: from the metric catalog — see [CONFIG.md](CONFIG.md)
   - `metric_direction`: infer from name (warnings/errors/complexity → lower; coverage → higher)
   - `tag`: descriptive name based on metric (e.g., `refactor-lint`)
   - `strategies`: pick relevant ones — see [STRATEGIES.md](STRATEGIES.md)

3. **Write `refactor.json`** — see [CONFIG.md](CONFIG.md) for field reference.
   Show the user for confirmation, then proceed.

## Setup Phase

1. Parse and validate `refactor.json`
2. Verify `autorefactor/<tag>` branch doesn't exist — if it does, append `-2`, `-3`, etc.
3. `git checkout -b autorefactor/<tag>`
4. Run `test_command` to confirm tests pass at baseline
5. Run `metric_command` → record baseline value
6. If tests fail at baseline, STOP and tell the user
7. Add `results.tsv` and `run.log` to `.gitignore`
8. Initialize `results.tsv` with tab-separated header row
9. Tell user setup is complete. **This is the last interaction.** From here, fully autonomous.

## The Refactoring Loop

**LOOP FOREVER. NEVER stop. NEVER ask permission to continue.**

The user may be away. They expect you to run indefinitely until manually stopped.

```
1. PICK a target file
   - Scan target_files matching the globs, excluding exclude_files
   - Prefer files with the worst per-file metric (most warnings, highest complexity)
   - Avoid files that failed in the last 3 attempts (cooldown)
   - If all files are on cooldown, reset cooldowns
   - Review results.tsv for trends. Re-read target files for current state.

2. ANALYZE the file
   - Read the file
   - Identify specific issues (lint warnings, complex functions, poor naming, dead code)
   - Pick a strategy from the strategies list — see [STRATEGIES.md](STRATEGIES.md)
   - Form a concrete hypothesis: "Extracting the validation logic from processOrder()
     into a separate validateOrder() function should reduce complexity by ~2 and fix
     the 'complexity' lint warning on line 47"

3. EDIT the file
   - Make the refactoring change
   - Stay conservative: one logical change per iteration
   - Don't change behavior, only structure
   - Preserve all existing tests

4. TEST
   - Run: test_command (with timeout). Redirect ALL output to run.log.
   - If tests fail → REVERT immediately (git checkout -- <file>)
   - Log the failure reason and move on

5. MEASURE
   - Run: metric_command → new_value
   - Compare to last known good value
   - If improved or equal → KEEP
   - If regressed → REVERT (git checkout -- <file>)
   - Exception: equal metric + simpler code → KEEP
   - Exception: tiny gain + lots of ugly code → DISCARD

6. COMMIT (if kept)
   - git add <file>
   - git commit with descriptive message:
     "refactor(<scope>): <what changed>

     Strategy: <strategy used>
     Metric: <metric_name> <old_value> → <new_value>
     Hypothesis: <what we expected>"
   - Do NOT commit results.tsv or run.log.

7. LOG
   - Append to results.tsv:
     iteration | timestamp | file | strategy | hypothesis | metric_before | metric_after | status | reason

8. CONTINUE
   - Every 10 iterations, print a progress summary.
   - Loop back to step 1.
```

## Critical Rules

- **Protect your context window** — see [CONTEXT.md](CONTEXT.md). This is the #1
  risk for long-running sessions. Always redirect output, use grep not read, only
  tail on crash.
- **results.tsv is your memory** — it survives git resets and contains full history.
  Re-read it when you need to review past experiments.
- **One change per iteration** — test one hypothesis at a time.
- **Never change behavior** — every refactoring must be strictly behavior-preserving.
- **Respect the revert** — if tests fail or the metric regresses, revert immediately.
  Don't try to "fix" a failing refactoring with more changes.
- **Don't fight the architecture** — refactoring is about making the existing design
  cleaner, not redesigning the system.
- **When stuck** — see [STRATEGIES.md](STRATEGIES.md) for tactics.

## Example results.tsv

```
iteration	timestamp	file	strategy	hypothesis	metric_before	metric_after	status	reason
1	2026-03-26T10:00	src/api/orders.ts	fix-lint-warnings	fix unused import and no-explicit-any	147	144	keep	3 warnings fixed
2	2026-03-26T10:05	src/utils/format.ts	extract-functions	extract date formatting into formatDate()	144	142	keep	reduced complexity
3	2026-03-26T10:10	src/api/users.ts	reduce-complexity	flatten nested if/else with early returns	142	142	keep	equal metric, simpler code
4	2026-03-26T10:15	src/components/Dashboard.tsx	fix-lint-warnings	fix missing key prop in list	142	140	keep	2 warnings fixed
5	2026-03-26T10:20	src/api/orders.ts	simplify-conditionals	consolidate redundant null checks	140	140	discard	test failure in order.test.ts
```

## Output

After the loop runs, the user gets:

- **Git history on `autorefactor/<tag>`** with each kept improvement as a separate commit
- **results.tsv** with the full experiment log
- **run.log** with the most recent test/metric output

The user can review the git log, cherry-pick changes they like, or merge the whole branch.
