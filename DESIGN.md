# Design

How the omp factory works, and why it's built this way.

## The idea

A product idea becomes merged, release-ready code through a fixed pipeline.
Scripts handle the deterministic stages (state, tests, merge, scheduling,
publishing). Agents do the judgment in between. People make the decisions
that are theirs, from one page, and every one of those decisions is written
down in the repository.

omp is an MIT harness you can point at Ollama. The factory does not care
which weights sit in the builder seat, only that the seat writes `build.md`
and the gate still runs `checks.sh`.

## Pieces

| Piece | File | Job |
|---|---|---|
| Program | `factory/bin/program` | PRD → research → plan → parallel tickets → release. Only writer of `program.json` |
| Board | `factory/bin/board` + `dashboard.html` | Local UI and JSON API. Spawns program steps and runs; never blocks on a model |
| Queue | GitHub issues labeled `factory` | What to build next |
| Orchestrator | `factory/bin/run` | Starts jobs, spawns omp, calls scripts. Writes nothing itself |
| Spawner | `factory/bin/agent` | One `omp -p` per role, session under `jobs/<id>/sessions/<role>/` |
| Agents | `factory/agents/*.md` | One file per role. Each writes exactly one factory file |
| Overlay | `factory/omp.yml` | Yolo approvals, no ask/advisor/web search for unattended runs |
| Job state | `factory/jobs/<id>/` | `job.json` plus every agent's file. Gitignored |
| Program state | `factory/programs/<id>/` | `program.json`, PRD, research, tickets, changes, decisions, release. Gitignored |
| Published docs | `docs/programs/<id>/` | The same documents plus an index with the consent log. Committed |
| Spine | `factory/bin/job`, `gate`, `route` | Deterministic. `job` is the only writer of `job.json` |
| Checks | `factory/checks.sh` | The repo's test/lint/typecheck/build |
| Exit | the PR, then the release note | Merging is the approval per ticket; accepting the release is the approval per program |

## Program stages

```
research → review-prd → plan → review-tickets → implementing → release → done
    ▲                                              │
    └──────────── change request ──────────────────┘
```

`stage` in `program.json` moves only through `factory/bin/program`. `step`
does the next thing for the stage: run the researcher, run the planner, label
every ready ticket and drain the queue, or draft the release. The board calls
`step` after every button. A person's approval is a stage change with a log
entry; a `running` marker stops two steps running at once.

## Per-ticket pipeline

```
intake → route → worktree → spec → builder → gate → reviewers → approver
                                                      └ CHANGES ┘
```

`stage` in `job.json` is one of `intake`, `needs-info`, `spec`, `build`,
`gate`, `review`, `approve`, `merging`, `needs-human`, `merged`, `rejected`.
Legal moves are in `TRANSITIONS` at the top of `factory/bin/job`.

## Decisions

**Python orchestrator, with a session command.** The loop stays in code. omp
only sits in the judgment seats. If a process dies, `program step` and
`factory/bin/run` read their JSON and resume the stage. When a drain merges a
ticket, it steps that ticket's program after releasing the drain lock, so the
next ready ticket starts without a person pressing Step. A step already in
progress is left to notice the merge itself.

**Board first, terminal optional.** The board is the product surface. Every
action it offers is a one-line call to `program` or `run`, so `/factory` in an
omp session, or the scripts directly, can do the same thing. The board spawns
long work and returns 202; the page polls `/api/state`. `FACTORY_NO_DRAIN=1`
keeps `run feedback` and `run answer` from draining the queue inside the HTTP
handler.

**Ticket headers carry the schedule.** `Depends:` (hard), `After:` (soft),
`Touches:` (area claims), `Kind:` (feature, contract, spike), `Decision:`
(a question for a person), `Status: superseded`. `ready_tickets` returns
every filed ticket whose dependencies have merged, whose decision is accepted,
and whose areas no running ticket claims; `implement` labels all of them and
lets `run` take up to `concurrency`. The planner is told to land contracts
first so the rest fans out. This is optimistic: `After:` tickets build on the
base branch at the same time and the approver's sync-with-main catches the
rebase.

**One change funnel.** A change request is a file with a status
(`open → planned → done`), a log entry, and a research rerun. The planner
replans in diff mode: filed numbers and titles are kept, merged tickets may
not change, superseded tickets keep their section with a status line.
`merge_tickets` enforces that and tags every ticket as merged, same, changed,
superseded, or added, so the board can show the diff and `file_tickets` can
comment, close, or file the right GitHub issues. Nothing is filed until the
person approves the new list.

**Decisions are ADRs the factory proposes and the person accepts.** A
`Decision:` line becomes `decisions/NNN-slug.md` with `status: proposed`, the
options copied from the ticket, and the ticket blocked. `decide` writes the
chosen option, who decided, and `status: accepted`. Spec writers read accepted
decisions from `docs/programs/<id>/decisions/` and may not contradict them.

**Consent is recorded, not assumed.** `program["log"]` holds every approval,
revision, change, decision, and release acceptance with the operator
(`FACTORY_OPERATOR`, then `gh api user`, then the OS user). `publish_docs`
renders it into `docs/programs/<id>/README.md` and commits on every step.
The PRD text itself is still read from a file, never pasted into a prompt.

**Local models, named as roles.** `config.json` maps `smol` / `default` /
`slow`, and any extra alias such as `guard`, to `omp models` selectors.
Intake and UI review use smol. Builder uses default. Research, planning and
spec use slow. Security review and approval can point at `guard` when that
alias is a different model family from the builder. A role value that
contains `/` is used as a selector directly. Copy whatever `omp models`
prints; do not invent ids. `probe` checks every distinct selector.

**Concurrency 2 by default.** One local GPU swapping two 14B models already
hurts. Turn it up when the machine can keep the weights resident; the ready
set is usually bigger than the concurrency.

**Builder sessions resume.** `factory/bin/agent --resume` passes `omp
--continue` against `jobs/<id>/sessions/builder/`, so round 2 keeps the
builder's files and tool history.

**Gate still owns "does it work".** Reviewers do not re-run `checks.sh`. A
branch that edits `protected_paths` fails even if tests pass. omp has no
stop hook, so when a builder writes `STATUS: READY`, `run` calls
`gate --nudge` once. A failure sends the builder back with the log and
writes no round report. A second failure is left to the real gate, which
is what spends a round.

**The line keeps moving.** A drain that merges a ticket steps its program
after the drain lock is released. While a drain still has a free slot, `program
refill` labels the tickets that merge just unlocked. A merge conflict goes
back to the builder once, with the earlier passing reviews kept and only code
re-reviewed; a second conflict waits for a human. After a merge, parked
branches that would conflict with the new base get that same one try. Active
worktrees are not touched. The memory note is committed on the job branch
before `gh pr merge`, so it lands with the pull request. When GitHub rejects
that merge because the base branch moved, the factory merges again if the
branch already contains main, and otherwise syncs, re-runs the gate, and
tries once more. A second failure waits for a human. The merge gate allows
the memory note the factory just committed; the builder's gate does not.

**Plans have to be able to run wide.** The planner sets `Risk:` and
`Surfaces:` and names `Touches:` areas narrowly. `areas` in config maps those
names to path globs; `shared_paths` are allowed on every ticket. The gate
fails a branch that edits anything else. On the first plan, a schedule where
most waves hold one ticket is sent back to the planner once, then accepted
with the warning. Filed issues are grouped under one parent issue, with
sub-issues and blocked-by links. That hierarchy is a view; the schedule the
factory runs is still the ticket headers.

**The contract is written before the builders.** Research cites external
docs and adds an `## Acceptance` section. When `write_contract` is on, a
contractor turns those scenarios into `test/hidden/contract.test.js` and
commits it on the base branch before planning. `checks.sh` runs the whole
`test/` directory, so each ticket adds its own file. When `verify_release`
is on, the release note is drafted only after `checks.sh`, `accept.sh`, and
a browser walk of the PRD pass on the base checkout.

**Cost is tokens first.** omp session jsonl records `usage.totalTokens` and
`usage.cost.total`. Ollama reports $0. Each transition records the tokens it
spent and, when the stage being left was a wait, how long the person held it.
The board shows both. `job summary` includes the wait.

**omp overlay, not global config.** `--config factory/omp.yml` so a factory
run does not rewrite `~/.omp/agent/config.yml`.

## Guardrails

- Only issues labeled `factory` run.
- Agents push only `factory/*`. Only `job merge` and the board's Merge button
  call `gh pr merge`.
- Factory never deploys. The release note is the last artifact.
- `max_rounds` then a human.
- Force-push is out: omp `bash.gitGuard` plus the builder prompt.
- A person approves the enriched PRD, the ticket list, every replanned list,
  every decision, every escalated PR, and the release. An approver APPROVE
  merges after the branch is synced with the base and the gate passes again.
