# Install the factory

Install it into the root of a project that already has a GitHub `origin`. The factory directory comes from [github.com/Mo33n/software-factory](https://github.com/Mo33n/software-factory). Do not copy this README, `DESIGN.md`, or `LICENSE` into the project.

## 0. Check

Stop if any of these fail:

- `git remote -v` shows a GitHub `origin`, and `gh auth status` is a user who can push and manage labels.
- The default branch exists on `origin`.
- `omp --version` is 18 or newer.
- `python3 --version` is 3.10 or newer.
- `omp models` lists at least one model that can call tools. Copy those selectors. Do not invent ids.
- For a web app, `agent-browser --version` runs. If it does not: `npm install -g agent-browser && agent-browser install`.
- `factory/` is not already in the project. If it is, show the clash and ask before replacing it.

## 1. Fetch `factory/` from GitHub

From the project root:

```bash
rm -rf /tmp/software-factory
git clone --depth 1 https://github.com/Mo33n/software-factory.git /tmp/software-factory
rm -rf factory
cp -R /tmp/software-factory/factory ./factory
mkdir -p .omp/commands
cp /tmp/software-factory/.omp/commands/factory.md .omp/commands/factory.md
chmod +x factory/checks.sh factory/accept.sh factory/bin/*
rm -rf /tmp/software-factory
```

That is the whole `factory/` directory: agents, `bin/`, `config.json`, `checks.sh`, `accept.sh`, `contract/`, and the board. The reference tests (`factory/test_*.py`) come along. They check the factory, not your app.

## 2. Make it this project

**`factory/checks.sh`.** Replace the placeholder with this repo's real checks: install what a fresh worktree needs, then lint, typecheck, test, and build. Non-interactive, no `--fix`, exit non-zero on failure. Keep `set -euo pipefail`.

**`factory/config.json`.**

- `base_branch`: the default branch.
- `merge_method`: one the repo allows.
- `models`: selectors from `omp models`. `default` builds, `slow` writes the spec and reviews, `smol` does intake and cheap review. If only one model fits, point all three at it and set `concurrency` to `1`.
- `page`: for a web app, set `start`, `port_env`, and `path`. Until `page.start` is set, a repo with `index.html` or `public/index.html` will not auto-merge.

  ```json
  "page": { "start": "node src/server.js", "port_env": "PORT", "path": "/" }
  ```

- `protected_paths`: keep the factory entries, including `factory/contract/*`. Do not protect the project's tests.
- `ports.base`: move it if 5100–5199 is taken.
- `docs_dir`: where PRDs, tickets, decisions, and the release note are committed. Default `docs/programs`.
- `areas`: map each `Touches:` name to the path globs that ticket may edit. `shared_paths` are allowed on every ticket.

**`AGENTS.md`.** Add setup, how to start the dev server on a given port, and the same commands as `checks.sh`. Then:

```markdown
## Software factory

`python3 factory/bin/board` serves the board at http://127.0.0.1:8000/.
Paste a PRD there. Program documents land under `docs/programs/<id>/`.
Job state is in `factory/jobs/` and `factory/programs/` (gitignored).
Each job builds in `.worktrees/<job-id>` (gitignored).
```

**`.gitignore`.** Append:

```gitignore
/factory/jobs/
/factory/programs/
/factory/probe-results.json
/.worktrees/
```

**Labels.**

```bash
gh label create factory --color 5319E7 --description "Build this with the software factory"
gh label create factory:backlog --color C5DEF5 --description "Planned by the factory, waiting for its dependencies"
gh label create factory:needs-info --color FBCA04 --description "The factory asked the author a question"
gh label create factory:needs-human --color D93F0B --description "The factory needs a human decision on the PR"
```

## 3. Verify

1. `bash factory/checks.sh` exits 0 on the current branch.
2. `factory/bin/job` and `factory/bin/program` with no args print usage.
3. `python3 -m json.tool factory/config.json` succeeds.
4. `omp models` lists every selector in `config.json`.
5. `git check-ignore factory/jobs/x factory/programs/x .worktrees/x` prints all three.
6. `python3 factory/bin/board` starts, and http://127.0.0.1:8000/api/readiness reports every check `ok`. Stop it afterwards.

## 4. Hand over

Stay on the branch you were on. Ask before pushing. Then:

```bash
python3 factory/bin/board
```

Open http://127.0.0.1:8000/ and paste a small PRD. Approve the enriched PRD, then the tickets. Merge each pull request, or send it back from the board. Restart omp once so `/factory` is available.
