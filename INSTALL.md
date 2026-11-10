# Install the omp software factory

You are a coding agent installing this factory into the user's project. Copy
what's generic, adapt what's repo-specific, verify it, and report. Work in the
project's root.

The source is the folder the user pointed you at.

## 0. Check the project

Stop and tell the user if any of these fail:

- It's a git repo with an `origin` on GitHub, and `gh auth status` shows a user
  who can push to it and manage its labels.
- Its default branch exists on `origin`.
- `omp --version` runs (18+).
- `omp models ollama` (or llama.cpp / lm-studio) lists at least one model that
  can call tools. Copy those selectors; do not invent ids.
- `python3 --version` is 3.10 or later.
- `agent-browser --version` runs. If it does not: `npm install -g agent-browser`
  and `agent-browser install`. Page checks call this CLI from bash. Do not add
  a `browser` tool to omp `--tools`; this omp build rejects that name.
- None of the files in step 1 already exist. If one does, show the clash and
  ask before overwriting.

## 1. Copy

Copy these paths unchanged:

```
factory/agents/
factory/bin/job
factory/bin/gate
factory/bin/route
factory/bin/agent
factory/bin/run
factory/bin/program
factory/bin/board
factory/bin/probe
factory/bin/factorylog.py
factory/config.json
factory/omp.yml
factory/checks.sh
factory/dashboard.html
.omp/commands/factory.md
```

```bash
chmod +x factory/checks.sh factory/bin/job factory/bin/gate factory/bin/route factory/bin/agent factory/bin/run factory/bin/program factory/bin/board factory/bin/probe
```

Don't copy `README.md`, `DESIGN.md`, `INSTALL.md`, `LICENSE`, or the
`factory/test_*.py` files. They describe and check the reference, not the
user's project.

## 2. Adapt

**`factory/checks.sh`.** Replace the placeholder with the project's real
checks: install dependencies if a fresh worktree needs them, then lint,
typecheck, tests and build. Non-interactive, no `--fix`, exit non-zero on
failure. Keep `set -euo pipefail`.

**`factory/config.json`.**

- `base_branch`: the default branch.
- `merge_method`: one the repo allows.
- `models`: paste selectors from `omp models ollama` (or the engine in use).

  | Role | Seat |
  |---|---|
  | `default` | Best local coder that actually calls tools |
  | `slow` | Stronger/reasoning local model for spec, security, approve |
  | `smol` | Smaller coder for intake and cheap review |

  On a 24 GB machine with this user's current list, a working map is:

  ```json
  "models": {
    "smol": "ollama/qwen2.5-coder:14b",
    "default": "ollama/qwen2.5-coder-16k:latest",
    "slow": "ollama/qwen3:14b"
  }
  ```

  If only one model stays resident, point all three at it and set
  `concurrency` to 1.
- `protected_paths`: keep the factory entries. Don't protect the project's
  tests.
- `ports.base`: move if 5100–5199 is taken.
- `docs_dir`: where program documents (PRD, research, tickets, decisions,
  change requests, release note, consent log) are committed. Default
  `docs/programs`. Pick a folder the project's docs site will pick up if it
  has one.

**Agents.** Tailor checklists to the stack without changing file names or
first-line formats:

- `reviewer-security.md`: this project's auth, data access, payments.
- `reviewer-ui.md`: design tokens and shared components.
- `reviewer-ux.md`: kinds of users.
- `approver.md`: what a human must always see, added to the ESCALATE list.
- `intake.md`: one line under Decide saying what the project is for.

**`AGENTS.md`.** Every omp agent reads it. Include setup for a fresh
checkout, how to start the dev server on a given port, and the same
validation commands as `checks.sh`. Then add:

```markdown
## Software factory

`python3 factory/bin/board` serves the board at http://127.0.0.1:8000/.
Paste a PRD there; the factory researches, plans tickets with dependencies,
builds every ready ticket in parallel through intake → spec → build → gate →
review → approve (one PR per ticket), and drafts a release note. Approvals,
decisions and change requests are buttons on the board. Program documents are
committed under `docs/programs/<id>/`. Working state is in `factory/jobs/`
and `factory/programs/` (gitignored); each job builds in `.worktrees/<job-id>`
(gitignored).
```

**`.gitignore`.** Append:

```gitignore
/factory/jobs/
/factory/programs/
/factory/probe-results.json
/.worktrees/
```

**Areas and models.** In `factory/config.json`, map each `Touches:` name the
planner will use to the path globs that ticket may edit. `shared_paths` are
allowed on every ticket. `*` matches one path segment; `**` matches across
directories. Point `smol` at a cheaper selector than `default`, and add a
`guard` alias on a different model family for `roles.approver` and
`surface_reviewers.security.security` when you want a second opinion on
approval. Set `write_contract` and `verify_release` to true once `accept.sh`
can start the app.

**Checks.** `factory/checks.sh` runs every test file in `test/`, not one
shared file. `factory/accept.sh` runs `test/hidden/` and, for a server with a
page, a scripted `agent-browser` smoke. Each ticket adds its own test file.

**Labels.**

```bash
gh label create factory --color 5319E7 --description "Build this with the software factory"
gh label create factory:backlog --color C5DEF5 --description "Planned by the factory, waiting for its dependencies"
gh label create factory:needs-info --color FBCA04 --description "The factory asked the author a question"
gh label create factory:needs-human --color D93F0B --description "The factory needs a human decision on the PR"
```

## 3. Verify

1. `bash factory/checks.sh` exits 0 on the current branch.
2. `factory/bin/job` with no args prints usage; `factory/bin/program` too.
3. `python3 -m json.tool factory/config.json` succeeds.
4. `omp models` lists every selector in `config.json` `models`.
5. `git check-ignore factory/jobs/x factory/programs/x .worktrees/x` prints
   all three paths.
6. `python3 factory/bin/board` starts, and http://127.0.0.1:8000/api/readiness
   reports every check `ok`. Stop it afterwards.

## 4. Hand over

Stay on the branch the user is on. Ask before pushing. Then tell them:

- `python3 factory/bin/board`, open http://127.0.0.1:8000/, paste a small PRD.
  Keep `concurrency` at 1 until they trust the local models.
- Approve the enriched PRD, then the ticket list. Tickets become GitHub
  issues and pull requests. Merge or send back on the board.
- Change of mind: the change request box on the program card. The factory
  replans and asks again before filing anything.
- Weights stay on the machine. Search and `gh` still leave it.
