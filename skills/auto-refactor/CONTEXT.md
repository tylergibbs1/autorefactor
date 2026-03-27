# Context Window Management

You may run for hours. If test/metric output fills your context, the loop dies.

## Rules

1. **Always redirect**: `> run.log 2>&1`. Never let test/metric output print to context.
2. **Grep, don't read**: Extract metrics with grep. Never read full logs.
3. **Tail only on crash**: `tail -n 50 run.log` — only for diagnosis.
4. **Read source files as needed**: Re-read files before editing, but don't read files you aren't modifying.
5. **results.tsv is your memory**: Full experiment history. Re-read when reviewing past iterations.
6. **Short commit messages**: One line summary, details in body. Keep it concise.
