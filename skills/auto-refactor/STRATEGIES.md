# Refactoring Strategies

Read the code, understand what's messy, make a targeted fix. Don't apply mechanically.

## Strategies

- **reduce-complexity**: Flatten nesting with early returns, extract lookup tables from switch statements, split multi-purpose functions.
- **extract-functions**: Pull self-contained logic blocks into named functions. Good for loop bodies, repeated patterns, and long expressions.
- **improve-naming**: Rename to be self-documenting. Target single-letter vars, abbreviations, misleading booleans, generic names.
- **remove-dead-code**: Delete unreachable code, unused imports/vars/functions, clearly abandoned comment blocks.
- **fix-lint-warnings**: Read the actual warning, understand it, fix it properly. No suppress comments.
- **simplify-conditionals**: De Morgan's law, consolidate redundant checks, replace boolean flags with guard clauses.
- **reduce-duplication**: Extract shared patterns only when 3+ instances exist and they represent the same concept.

## When Stuck

1. Review results.tsv — which strategies worked? What failed?
2. Combine near-misses — two discarded changes that each almost improved.
3. Try the opposite — if simplifying hurt, try extracting.
4. Hunt unexplored files — untouched files may have low-hanging fruit.
5. Switch strategies — break out of a rut.
6. Simplification sweep — remove components one at a time. Equal metric + less code = win.
