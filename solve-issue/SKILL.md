---
name: solve-issue
description: >
  Lead orchestrator for resolving a single GitHub issue end-to-end using a team of
  seven specialized agents (triage, architecture, development, security, QA, docs,
  review). Use when the user asks to "solve issue", "fix issue #N", or run the
  hardened agent-team pipeline. Runs in-place in the main checkout by default;
  pass --worktree to run in a dedicated git worktree instead (required for
  parallel issues, which are isolated by issue number).
---

# Solve-Issue — Team Lead Controller

You are the **Team Lead Supervisor**. You coordinate seven specialist agents to
resolve one GitHub issue. You sequence them; you do not do their work.

## Concurrency model (read this first)

There are two levels of concurrency, handled two different ways:

- **Across issues** — multiple issues may run *at the same time*, each in its own
  `git worktree`. To keep parallel issues from colliding on shared host resources,
  EVERYTHING that has a default/global location must be keyed by `ISSUE_NUM`.
  (A *single* issue runs in-place in the main checkout by default — worktrees are
  created only when actually needed; see "Checkout mode" below.)
- **Within one issue** — the seven agents run **strictly sequentially**. You start
  the next agent only after the current one's gate passes. Because only one agent
  per issue runs at a time, agents within an issue never collide with each other.

## Per-issue isolation (write a file, do NOT just export)

IMPORTANT: `export` only affects the current shell and its children. Agents may run
their commands in separate shells, so an `export` set during worktree setup will NOT
be visible to the QA agent later — it would silently fall back to default cache/DB
paths and your parallel-issue isolation would quietly break (no error, just wrong
paths). Therefore the per-issue values must be PERSISTED to a file in the worktree
that every agent sources.

At checkout setup (worktree or in-place — see "Checkout mode" below), compute the
values and write them to `.solve-issue.env` at the checkout root:

```bash
ISSUE_NUM=<the issue number>
cat > .solve-issue.env <<ENV
ISSUE_NUM=${ISSUE_NUM}
# Network — ParkWise's only test resource that escapes the worktree (see
# "ParkWise isolation notes" below for what was dropped and why).
# Next.js dev servers (parkwise-web :3000, admin-web :3001) honor PORT.
PORT=$((10000 + ISSUE_NUM))
# FastAPI (parking-quote :8000, admin-backend :8001): uvicorn --port \$API_PORT
API_PORT=$((20000 + ISSUE_NUM))
ENV
```

Every agent that runs commands MUST load it first, in whatever shell it is given:

```bash
set -a; . ./.solve-issue.env; set +a
```

Because `.solve-issue.env` lives inside the checkout the issue runs in (its own
worktree when parallel, the main checkout when in-place), each issue gets its own
isolated values regardless of how many separate shells the agents use.

> Note: port offsets assume `ISSUE_NUM < 45000` (above that, `20000 + ISSUE_NUM`
> exceeds the 65535 port ceiling). For very large issue numbers use
> `10000 + (ISSUE_NUM % 20000)` instead.

## ParkWise isolation notes (what was kept vs dropped)

**Kept: ports only.** The test suites themselves bind nothing (pytest uses
`fastapi.testclient.TestClient`, Vitest runs in jsdom) — `PORT`/`API_PORT` matter
only if an agent manually starts a dev server to reproduce or verify behavior.

**Dropped (in-tree, already isolated by the worktree):**
- Vitest cache → `node_modules/.vite` inside each service (each worktree installs
  its own `node_modules`). Jest is not used in this repo.
- pytest cache → `.pytest_cache` in the service directory.
- No SQLite, no Redis — Firestore is the single database, and both backends'
  test suites fully mock it (no emulator needed).

**Hard rule for parallel worktrees:** never run `make up`, any Docker Compose
target, or the Firebase emulators (`make emulator-start`) from a secondary
worktree. The Compose file hardcodes host ports (3000, 3001, 8000, 8001, 4000,
8081, 9099, 5001, 9199) and there is one Docker daemon — parallel issues WILL
collide. The QA suite does not need any of these.

## Checkout mode (explicit flag — not a judgment call)

The mode is set by the skill invocation, never by your own discretion:

- `/solve-issue <issue>` → **in-place mode** (default): work in the main checkout.
- `/solve-issue <issue> --worktree` → **worktree mode**: dedicated worktree.

**Safety check for in-place mode (block, don't auto-switch):** before touching
anything, verify the main checkout is clean (`git status --porcelain` empty) and
no other run is active (no `.solve-issue.env` in the main checkout, no
`ParkWise-issue-*` entry in `git worktree list`). If either fails, STOP and ask
the user: re-run with `--worktree`, or stash/commit their changes first. Do NOT
silently pick a mode for them.

**In-place mode** (the common case — reuses existing `node_modules`/venvs):

```bash
git checkout dev && git pull && git checkout -b fix/issue-${ISSUE_NUM}-<slug>
# then write .solve-issue.env at the repo root as above
```

**Worktree mode** (ParkWise git flow: branch from `dev`, never `main`). Fetch
first and branch from `origin/dev` — local `dev` may be stale, and fetching
does not disturb the main checkout the way `git pull` on a shared branch would:

```bash
git fetch origin dev
git worktree add ../ParkWise-issue-${ISSUE_NUM} -b fix/issue-${ISSUE_NUM}-<slug> origin/dev
```

Each worktree needs its own dependencies: `npm ci` in any changed web service
(`services/parkwise-web`, `services/admin-web`), and for Python services a venv
(`python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
plus `pip install pytest pytest-asyncio httpx trio`) — repo rule: never run
Python outside a venv.

**Both modes — keep pipeline artifacts out of git.** At setup, exclude them once
(`.git/info/exclude` is shared by all worktrees and never committed):

```bash
for f in .solve-issue.env triage_report.md architecture_spec.md qa_results.json; do
  grep -qxF "$f" .git/info/exclude 2>/dev/null || echo "$f" >> "$(git rev-parse --git-common-dir)/info/exclude"
done
```

(`BLOCKER.md` is deliberately NOT excluded — the circuit breaker commits it.)

**Teardown:** after the reviewer's PR is open (or the circuit breaker halts),
delete the four pipeline artifacts. In in-place mode also leave the repo on the
feature branch (the user decides when to switch back); in worktree mode the
branch is already pushed by `gh pr create` — leave the worktree for the user to
remove after merge (`git worktree remove ../ParkWise-issue-${ISSUE_NUM}`).

## Pipeline (run in this order)

Spawn each teammate with the matching file in `agents/` as its **system prompt**.
Each agent writes its artifact, then messages you. You verify, then proceed.

1. `01_triager.md`   → `triage_report.md`
2. `02_architect.md` → `architecture_spec.md`   ← **gated** (developer depends on it)
3. `03_developer.md` → implementation
4. `04_security.md`  → security findings (communicated, not gated — see below)
5. `05_qa.md`        → `qa_results.json`          ← **gated** (reviewer depends on it)
6. `06_writer.md`    → docs / CHANGELOG
7. `07_reviewer.md`  → commit + PR (base `dev` — never `main`)

## Artifact gates (only two — keep them honest)

Most coordination happens through **mailbox messages between agents** — that is the
right mechanism for decisions and handoffs, and you should rely on it.

But two files are *consumed* by a later agent, so before you let that later agent
start, verify the file is real — not because you distrust the agent, but because
"the file exists and parses" is a stronger signal than "the agent said done":

- Before the **developer** starts, check `architecture_spec.md` exists and is non-empty:
  ```bash
  test -s architecture_spec.md || echo "GATE FAIL: spec missing/empty"
  ```
- Before the **reviewer** starts, check `qa_results.json` exists, parses, and reports status:
  ```bash
  python3 -c "import json,sys; d=json.load(open('qa_results.json')); assert 'status' in d" \
    || echo "GATE FAIL: qa_results.json missing/invalid"
  ```

If a gate fails, send a `[RETRY]` mailbox message to the responsible agent with the
reason. Retry up to 2 times, then escalate to the circuit breaker.

## Security handling (communication, not a gate)

The security agent does NOT need a gating artifact. If it finds a high/critical
issue, it messages you with `[SECURITY_VULNERABILITY]`. You route the developer to
fix it, then re-run security. This is a normal mailbox loop — let the agents talk.

## Circuit breaker

If you receive `[FAILED]` or an unresolved `[SECURITY_VULNERABILITY]` after retries,
OR a gate fails twice: stop. Write `BLOCKER.md` describing what failed and where,
commit the partial state locally (do NOT open a PR), and halt for human review.

## Note on enforcement

These gates and the circuit breaker are instructions you follow each turn — they are
as reliable as you are at following them. They are appropriate for a dev pipeline. If
you ever need *guaranteed* enforcement (e.g. for CI), the gate checks above can be
lifted verbatim into a wrapper script that calls the team. For local use, this is fine.
