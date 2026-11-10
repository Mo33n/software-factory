# Software factory (omp)

This repository is the reference install of a software factory driven by
local [omp](https://omp.sh) agents: a PRD goes in on the board, release-ready
pull requests and committed program documents come out.

## Setup

Python 3.10+, `omp` on PATH, `gh` authenticated, a local model engine
(`ollama serve` or equivalent). Selectors in `factory/config.json` must appear
in `omp models ollama` (or the engine you set).

There is no application to build in this repo. The factory is the product.

## Validation

```bash
python3 -m json.tool factory/config.json >/dev/null
python3 -m py_compile factory/bin/job factory/bin/gate factory/bin/route factory/bin/agent factory/bin/run factory/bin/program factory/bin/board
python3 factory/test_program.py
python3 factory/test_enrichments.py
factory/bin/job
factory/bin/agent
factory/bin/program
factory/bin/run --help
```

The tests run without a model or GitHub. `factory/checks.sh` is a placeholder
and exits 1 until copied into a real project and filled in.

## Software factory

`python3 factory/bin/board` serves the board at http://127.0.0.1:8000/. A
program (`factory/bin/program`) moves research → review-prd → plan →
review-tickets → implementing → release → done; a change request sends it
back to research and the planner diffs the ticket list. `factory/bin/run`
builds every open GitHub issue labeled `factory` through intake → spec →
build → gate → review → approve, one PR per issue, up to `concurrency` at
once. A merge steps the program and refills a free slot. A conflict goes back
to the builder once. A merge GitHub rejects because the base moved is
retried once. The memory note is committed on the job branch before
the pull request merges. State is in `factory/jobs/` and `factory/programs/` (gitignored);
`factory/bin/job` and `factory/bin/program` are their only writers. Program
documents and the consent log are published to `docs/programs/<id>/`
(`docs_dir` in `factory/config.json`).

Each job works in its own git worktree at `.worktrees/<job-id>` (gitignored).
