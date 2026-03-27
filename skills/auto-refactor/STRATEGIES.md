# Refactoring Strategies

Pick from these based on what the file needs. Don't apply them mechanically. Read the code, understand what's messy about it, and make a targeted fix.

## reduce-complexity
Break down functions with high cyclomatic complexity. Look for:
- Deeply nested if/else chains → flatten with early returns
- Long switch statements → extract to lookup tables or strategy pattern
- Functions doing multiple things → split into focused functions

## extract-functions
Pull coherent blocks of logic into named functions. Good targets:
- Code inside a loop body that does something self-contained
- Repeated patterns (even if not exact duplicates)
- Long expressions that would be clearer with a name

## improve-naming
Rename variables, functions, and parameters to be self-documenting. Targets:
- Single-letter variables outside of tiny loops
- Abbreviations that aren't obvious
- Boolean variables/functions that don't read as questions (e.g., `data` → `isValid`)
- Generic names like `handler`, `process`, `data`, `result` when something more specific fits

## remove-dead-code
Delete code that's never reached or used. Look for:
- Unreachable code after returns/throws
- Unused imports
- Unused variables and functions
- Commented-out code blocks (if clearly abandoned, not temporarily disabled)

## fix-lint-warnings
Address whatever the linter is complaining about. Read the actual warning, understand it, and fix it properly. Don't just suppress warnings with ignore comments.

## simplify-conditionals
Make boolean logic easier to follow:
- De Morgan's law to simplify negated conditions
- Consolidate redundant checks
- Replace boolean flags with early returns or guard clauses

## reduce-duplication
Find repeated code patterns and extract shared utilities. Be careful:
- Only extract if the duplicated code truly represents the same concept
- Don't create abstractions that are harder to understand than the duplication
- Two instances might be coincidence, three is a pattern

## When Stuck

If improvements have stalled, try these in order:

1. **Review results.tsv for patterns** — which strategies worked? What failed?
2. **Combine near-misses** — two discarded changes that each almost improved? Try both together.
3. **Try the opposite** — if simplifying hurt, try extracting instead.
4. **Vary scope** — if large refactors keep failing, try smaller ones. If small ones plateau, try slightly larger.
5. **Hunt for unexplored files** — files not yet attempted may have low-hanging fruit.
6. **Simplification sweep** — remove components one at a time. Equal metric + less code = win.
7. **Switch strategies** — if you've been doing fix-lint-warnings, try reduce-complexity.
