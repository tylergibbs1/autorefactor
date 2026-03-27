# auto-refactor skill

An autonomous refactoring agent skill for Claude Code. Runs an infinite improve loop: pick a file, refactor it, run the test suite, measure a quality metric, keep improvements, discard failures.

Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) and [tylergibbs1/autoresearchskill](https://github.com/tylergibbs1/autoresearchskill), adapted for code quality instead of ML metrics.

## Install

```bash
npx skills add tylergibbs1/autorefactor
```

Or install manually:

```bash
cp -r skills/auto-refactor ~/.claude/skills/auto-refactor
```

## Usage

Just run the skill — no config needed:

```
/auto-refactor
```

The agent explores your project, asks what metric you want to optimize, writes a `refactor.json` config, then loops indefinitely — trying refactorings, keeping improvements, discarding failures. Leave it running and come back to a cleaner codebase.

If you already have a `refactor.json`, it skips straight to the loop.

## Configuration

The agent writes this for you, but here's the format:

```json
{
  "tag": "refactor-lint",
  "target_files": ["src/**/*.ts"],
  "exclude_files": ["**/*.test.*", "**/node_modules/**"],
  "test_command": "npm test",
  "metric_command": "npx eslint src/ --format json | jq '[.[].messages | length] | add'",
  "metric_name": "eslint_warnings",
  "metric_direction": "lower",
  "secondary_metrics": [],
  "timeout_seconds": 300,
  "max_files_per_iteration": 1,
  "strategies": ["fix-lint-warnings", "extract-functions", "reduce-complexity"]
}
```

See `skills/auto-refactor/examples/` for more:
- `typescript-lint.json` — ESLint warning cleanup
- `typescript-types.json` — Type error reduction
- `python-coverage.json` — Test coverage improvement

## Supported Metrics

| Metric | Language | Direction |
|--------|----------|-----------|
| Lint warnings | JS/TS (ESLint), Python (ruff/flake8) | lower |
| Test coverage | JS/TS (Jest), Python (pytest-cov) | higher |
| Cyclomatic complexity | JS/TS (ESLint rule), Python (radon) | lower |
| Type coverage | TypeScript (type-coverage) | higher |
| Type errors | TypeScript (tsc --noEmit) | lower |
| Bundle size | Any bundler with JSON output | lower |
| Custom | Any command that outputs a number | either |

## Refactoring Strategies

- **fix-lint-warnings** — Address linter complaints properly (no suppress comments)
- **extract-functions** — Pull coherent logic blocks into named functions
- **reduce-complexity** — Flatten nesting, early returns, simplify control flow
- **improve-naming** — Make variables and functions self-documenting
- **remove-dead-code** — Delete unreachable code, unused imports, abandoned comments
- **simplify-conditionals** — Clean up boolean logic, consolidate checks
- **reduce-duplication** — Extract shared patterns (only when 3+ instances exist)

## Output

- **Git history** on `autorefactor/<tag>` — each improvement is a commit with context
- **results.tsv** — full experiment log (iteration, file, strategy, metric before/after, kept/reverted)
- **run.log** — most recent test output

## The Contract

Every change either improves the codebase or gets thrown away. Tests must pass. The metric must not regress. No exceptions.
