# refactor.json Reference

## Required Fields

| Field | Description |
|-------|-------------|
| `tag` | Run name for branch `autorefactor/<tag>` |
| `target_files` | Glob patterns for files the agent can modify |
| `exclude_files` | Glob patterns to skip (tests, generated code, vendor) |
| `test_command` | Must pass for a change to be kept. This is the safety gate. |
| `metric_command` | Must produce a single number on stdout. This is what we're optimizing. |
| `metric_name` | Human-readable metric name |
| `metric_direction` | `"lower"` or `"higher"` |

## Optional Fields

| Field | Default | Description |
|-------|---------|-------------|
| `secondary_metrics` | `[]` | Additional metrics to track (don't determine keep/revert) |
| `timeout_seconds` | `300` | Max wall-clock seconds per test/metric run |
| `max_files_per_iteration` | `1` | How many files to touch per experiment. Start with 1. |
| `strategies` | all | Which refactoring strategies to use — see [STRATEGIES.md](STRATEGIES.md) |

## Metric Catalog

Common metrics and how to measure them:

**Lint warnings** (easiest to start with)
- JS/TS: `npx eslint . --format json 2>/dev/null | jq '[.[].messages | length] | add'`
- Python: `flake8 --count --statistics . 2>&1 | tail -1` or `ruff check . --statistics 2>&1 | wc -l`
- Direction: lower is better

**Test coverage**
- JS/TS: `npx jest --coverage --coverageReporters=json-summary 2>/dev/null && jq '.total.lines.pct' coverage/coverage-summary.json`
- Python: `pytest --cov=. --cov-report=json 2>/dev/null && jq '.totals.percent_covered' coverage.json`
- Direction: higher is better

**Cyclomatic complexity**
- JS/TS: `npx eslint . --rule '{"complexity": ["error", 1]}' --format json 2>/dev/null | jq '[.[].messages | length] | add'`
- Python: `radon cc . -a -nc 2>/dev/null | tail -1`
- Direction: lower is better

**Type coverage** (TypeScript)
- `npx type-coverage --json 2>/dev/null | jq '.percent'`
- Direction: higher is better

**Type errors** (TypeScript)
- `npx tsc --noEmit 2>&1 | grep "error TS" | wc -l`
- Direction: lower is better

**Bundle size** (frontend projects)
- `npx webpack --json 2>/dev/null | jq '.assets | map(.size) | add'` or similar
- Direction: lower is better

**Custom metric**
- Any command that outputs a number. The user defines it.

## Writing Good Configs

- **Metric must be machine-readable** — the metric_command must output a single number to stdout.
- **Scope target_files tightly** — fewer files = more focused experiments.
- **Use exclude_files generously** — tests, generated code, vendor, dist, node_modules.
- **Timeout = 2x expected runtime** — long enough to finish, short enough to not waste time on failures.
- **Start with lint warnings** — lowest-friction starting point, almost every project has a linter.

## Example

```json
{
  "tag": "refactor-lint",
  "target_files": ["src/**/*.ts", "src/**/*.tsx"],
  "exclude_files": ["**/*.test.*", "**/*.spec.*", "**/node_modules/**", "**/dist/**"],
  "test_command": "npm test -- --watchAll=false",
  "metric_command": "npx eslint src/ --format json 2>/dev/null | jq '[.[].messages | length] | add // 0'",
  "metric_name": "eslint_warnings",
  "metric_direction": "lower",
  "secondary_metrics": [
    {
      "name": "test_count",
      "command": "npm test -- --watchAll=false 2>&1 | grep -oP '\\d+(?= passed)' || echo 0"
    }
  ],
  "timeout_seconds": 300,
  "max_files_per_iteration": 1,
  "strategies": ["fix-lint-warnings", "remove-dead-code", "improve-naming"]
}
```
