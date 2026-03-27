# refactor.json Reference

## Required Fields

| Field | Description |
|-------|-------------|
| `tag` | Branch name: `autorefactor/<tag>` |
| `target_files` | Glob patterns for files to modify |
| `exclude_files` | Glob patterns to skip |
| `test_command` | Must pass for a change to be kept |
| `metric_command` | Must output a single number to stdout |
| `metric_name` | Human-readable metric name |
| `metric_direction` | `"lower"` or `"higher"` |

## Optional Fields

| Field | Default | Description |
|-------|---------|-------------|
| `secondary_metrics` | `[]` | Tracked but don't determine keep/revert |
| `timeout_seconds` | `300` | Max seconds per test/metric run |
| `max_files_per_iteration` | `1` | Files to touch per experiment |
| `strategies` | all | Which strategies to use (see [STRATEGIES.md](STRATEGIES.md)) |

## Metric Catalog

| Metric | Command (JS/TS) | Command (Python) | Direction |
|--------|-----------------|-------------------|-----------|
| Lint warnings | `npx eslint . --format json \| jq '[.[].messages \| length] \| add'` | `ruff check . --statistics 2>&1 \| wc -l` | lower |
| Test coverage | `npx jest --coverage --coverageReporters=json-summary && jq '.total.lines.pct' coverage/coverage-summary.json` | `pytest --cov=. --cov-report=json && jq '.totals.percent_covered' coverage.json` | higher |
| Type errors | `npx tsc --noEmit 2>&1 \| grep "error TS" \| wc -l` | — | lower |
| Complexity | `npx eslint . --rule '{"complexity": ["error", 1]}' --format json \| jq '[.[].messages \| length] \| add'` | `radon cc . -a -nc 2>/dev/null \| tail -1` | lower |
| Custom | Any command that outputs a number | Any command that outputs a number | either |

Default to lint warnings if the user has no preference.

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
  "timeout_seconds": 300,
  "max_files_per_iteration": 1,
  "strategies": ["fix-lint-warnings", "remove-dead-code", "improve-naming"]
}
```
