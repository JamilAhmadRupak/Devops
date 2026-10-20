# Stage 1 — Foundation: step-by-step guide

> **Who this is for:** you, working through Operation TrackPoint with the `trackpoint` starter repo.
> **Time:** 10–14 hours. Spread it over 4–5 sessions (suggested split at the end).
> **Spoiler policy:** this guide shows you *how* to do each step. It does **not** tell you what is
> wrong with the starter app. Where you will hit a planted problem, you get a hint ladder
> (L1 → L3) inside a collapsed `Hints` block. Open one level at a time, and only after you
> have spent 20+ minutes on the problem yourself.
>
> **How each step works:** first a short **📘 Learn** box (what the thing is and why it exists,
> about 5 minutes of reading), then the **do this** commands, then questions for your learning
> log. The Learn boxes teach the tools. They deliberately leave the log questions for you.

---

## 0. Before you start

### 0.1 What "done" means for Stage 1

These are the PDF's checkpoints. Every step below feeds one of them.

| # | Checkpoint | Steps |
|---|---|---|
| C1 | Pushing directly to `main` is rejected; PRs require a passing check | 6, 7 |
| C2 | App runs locally, tests pass, `/healthz` and `/readyz` behave correctly when PostgreSQL is stopped | 4 |
| C3 | SSH as root fails; SSH with a password fails; SSH with your key as your user works | 8 |
| C4 | The API survives a reboot as a systemd service without you logging in | 9 |
| C5 | Risk register committed in `docs/` | 2, 3 |
| C6 | Server inventory document committed | 10 |

Plus the requirements with no checkbox, which are still graded: a branching strategy, PR template,
CODEOWNERS, `.gitignore`, a Makefile, the five "Break it on purpose" experiments, and written
answers to the six "Prove you understand" questions.

### 0.2 Where things live

```
Devops/
├── challenge-trackpoint/   ← the brief + these guides (public repo "Devops")
├── trackpoint/             ← YOUR PROJECT REPO. All Stage 1 deliverables go here.
└── trackpoint-mentor-sealed/
```

Everything you produce in Stage 1 (docs, Makefile, `.github/` files, systemd units, scripts) is
committed to **`trackpoint`**, because that becomes your portfolio repo (Appendix A layout).
Create `trackpoint/docs/learning-log.md` there and use it from now on. The empty one in
`challenge-trackpoint/docs/` can be deleted.

### 0.3 Which shell runs which command

Every code block starts with a comment saying where to run it:

| Comment | Meaning |
|---|---|
| `# laptop · PowerShell` | Windows PowerShell on your laptop |
| `# laptop · Git Bash` | Git Bash on your laptop |
| `# server` | Over SSH on your Linux server |

In **Windows PowerShell 5.1**, `curl` is an alias for `Invoke-WebRequest`. Always type `curl.exe`.

### 0.4 The learning-log habit (start now)

The learning log is graded. Add an entry every time something surprises you, using this shape:

```markdown
## 2026-10-12 — <short title>
**Tried:** what I did (commands)
**Expected:** what I thought would happen
**Actual:** what happened (paste the key error line)
**Why:** the explanation, in my own words
**Fix / takeaway:** what I changed or learned
```

### 0.5 Survival kit: how to read the commands in this guide

You'll type a lot of Bash. These patterns cover almost every line in this guide:

| You see | It means |
|---|---|
| `cd /opt/trackpoint` | Change directory. Linux paths use `/`, start at `/` (the root of the disk) and are case-sensitive. `~` is your home folder |
| `sudo <cmd>` | Run one command as **root** (the administrator). It asks for *your* password, not root's |
| `cmd1 \| cmd2` | Pipe: the output of `cmd1` becomes the input of `cmd2`, e.g. `ss -tlnp \| grep 5432` |
| `cmd > file` / `cmd >> file` | Write the output to a file (overwrite / append) |
| `echo x \| sudo tee file` | Write to a file only root may write. `sudo echo x > file` fails, because the `>` runs as you, not as root |
| `cat <<'EOF'` … `EOF` | Heredoc: every line between the two markers is fed in as text. The quotes around `'EOF'` stop `$` from being expanded |
| `VAR=value`, `$VAR`, `$(cmd)` | Set a shell variable, use it, insert the output of a command |
| `cmd1 && cmd2` | Run `cmd2` only if `cmd1` succeeded |
| `\` at the end of a line | The command continues on the next line |
| `cmd --help`, `man cmd` | Built-in documentation. Inside `man`: `/word` searches, `n` finds the next match, `q` quits |

**Exit codes.** Every command returns a number when it finishes: `0` means success, anything else
means failure. `echo $?` prints the last one. `&&`, `set -e` in scripts, systemd and CI all decide
"did it work?" from this number alone, so it's worth knowing it exists from day one.

### 0.6 Words you'll meet

| Term | Short meaning |
|---|---|
| Server / host / box | A computer that runs software for other computers. Here: a Linux VM in Oracle's cloud |
| VM | Virtual machine: a computer simulated in software on someone else's hardware |
| Process | A running program. `ps aux` lists them; each has a number, the PID |
| Service / daemon | A process that runs in the background with no terminal, usually started at boot |
| Port | A number (0–65535) that picks *which* program on a machine gets the network traffic. SSH 22, HTTP 80, HTTPS 443, PostgreSQL 5432, Redis 6379 |
| Environment variable | A named setting a process receives when it starts (`PORT=3000`). This is how you configure an app without changing its code |
| Backing service | Something the app needs but doesn't contain: a database, a cache, a queue |
| CI | Continuous Integration: a robot that runs lint and tests on every change |
| Runbook | Step-by-step instructions for an operations task, written so someone else (or you, tired, at 3 a.m.) can follow them |
| Idempotent | Safe to run twice: the second run changes nothing. A goal for every script and runbook step |

---

## Step 1 — Work like a team from the first commit (30 min)

> **📘 Learn: commits, branches, pull requests**
>
> - A **commit** is a snapshot of the whole project, plus a message, an author and a pointer to
>   the previous commit. A **branch** is only a movable label that points at a commit. `main` is
>   the branch everyone agrees is the truth.
> - A **pull request (PR)** is a GitHub page that says "please merge branch X into `main`". It
>   shows the diff, runs automatic checks (CI) and collects reviews. Teamwork and quality control
>   happen there.
> - **Trunk-based development:** everyone branches off `main`, keeps branches small, and merges back
>   within a day or two. (You'll compare it with the alternative, GitFlow, in Step 6.1.)
> - **Squash merge** turns all the commits in a PR into one commit on `main`.
> - Why bother when you work alone? In Step 7, `main` will reject direct pushes. In later stages,
>   whatever lands on `main` gets deployed. Practising the habit now costs nothing.

Branch protection comes in Step 7, but start the habit now: **never commit to `main`**. Every
step below is a short-lived branch that ends in a pull request.

```bash
# laptop · Git Bash
cd "/c/COmE HeRe/-_-/Engineer Hoite chaile porar dorkar nai/Devops/trackpoint"
git switch main && git pull
git switch -c docs/stage1-risk-register
```

Naming convention used in this guide: `feat/…`, `fix/…`, `docs/…`, `chore/…`, `hotfix/…`.

**Repository settings (github.com → trackpoint → Settings → General → Pull Requests):**

- Tick **Allow squash merging** and untick the other two merge methods. One PR then becomes one
  commit on `main`, and history stays readable.
- Tick **Automatically delete head branches**, so branches really are short-lived.

---

## Step 2 — Read the handover like an auditor (1.5–2 h)

> **📘 Learn: why Git never forgets**
>
> - Git keeps every version of every file that was ever committed. Deleting a file just makes a
>   *new* commit in which the file is missing. The old commits still contain it, and anyone can
>   bring them back with `git show`.
> - So a secret committed even once must be treated as **leaked**. The real fix is to **rotate**
>   it: create a new credential and revoke the old one. Rewriting history is cleanup, not the fix.
>   You'll do both in Stage 7. Keep this in mind for the "private repo" question in 2.2.
> - A **secret scanner** (gitleaks, GitHub secret scanning) searches every commit for text that
>   looks like a key or password. Teams run one in CI so leaks are caught before they merge.
> - "Read like an auditor" means: trust nothing you haven't seen evidence for, and write down the
>   evidence (file + line) next to every worry.

Before running anything, read what the dev team gave you. Keep a scratch list called
"things that worry me": it becomes your risk register in Step 3.

### 2.1 Read, in this order

1. Section 4 of the PDF ("Initial Infrastructure Situation"). Every row hides at least one risk.
2. `README.md`, especially the "For the new infra person" section. Symptoms are clues.
3. `OPS_NOTES.md`, the contractor's description of the legacy box.
4. Everything under `scripts/`, plus `docker-compose.yml`, `app/api/Dockerfile` and
   `.github/workflows/ci.yml`. You'll replace these in later stages, but they tell you how the
   team works today.
5. `app/api/src/config.js`, which lists every setting the app reads.

### 2.2 Find the leaked `.env` in Git history

The brief says the legacy `.env` was committed and later "deleted". Prove it, and learn the tools:

```bash
# laptop · Git Bash  (inside trackpoint/)
git log --oneline --all -- .env              # every commit that touched .env
git log --all --diff-filter=D --summary | grep -i "delete mode"   # files deleted in history
git show <commit-sha>:.env                   # view the file as it was in that commit
git log -p --all -S 'PASSWORD' --oneline     # "pickaxe": commits that added/removed a string
```

Rules while you do this:

- **Write down the variable names only, never the values.** The risk register lists
  `SMTP_PASSWORD (value redacted)`, not the password. Copying a secret into a new file is how
  leaks multiply.
- Note *who* committed it and *when*. You'll need that for the Stage 7 incident write-up.
- The repo is on GitHub now, so the history is on GitHub too. Ask yourself why that matters even
  though the repo is private. (Stage 7, question 1.)

Optional, but a great habit: install a secret scanner and let it find the leak for you.

```powershell
# laptop · PowerShell
winget install Gitleaks.Gitleaks
cd "C:\COmE HeRe\-_-\Engineer Hoite chaile porar dorkar nai\Devops\trackpoint"
gitleaks git -v        # scans the whole history
```

---

## Step 3 — Write the risk register (1.5 h) → checkpoint C5

> **📘 Learn: what a risk register is**
>
> A risk register is a table of things that could go wrong. Each row has the evidence, how likely
> it is, how bad it would be, and what you'll do about it. It turns a vague "this server scares
> me" into a prioritised to-do list that a manager can read in two minutes.
>
> - **A risk is not a bug.** A bug is broken now. A risk is a *chance* of harm: "no backups" isn't
>   broken today, it's a bad day waiting to happen.
> - **Likelihood × Impact** is a rough score so you can sort the list. The numbers are judgement,
>   not science. What makes them credible is the Evidence column.
> - **Be specific enough to act on.** "Security is weak" is useless. "Root can log in over SSH with
>   a password" is a task with a clear fix.

Create `docs/risk-register.md` with this structure:

```markdown
# Risk register — legacy TrackPoint box

Scoring: Likelihood (L) and Impact (I) from 1 (low) to 5 (high). Score = L × I.
15–25 = fix first, 8–14 = planned, 1–7 = accept/monitor.

| ID | Risk | Evidence (where I saw it) | L | I | Score | Fix in stage | Planned mitigation |
|----|------|---------------------------|---|---|-------|--------------|--------------------|
| R-01 | Root can log in over SSH with a password on port 22: brute-force → full takeover | PDF §4 "Server"; OPS_NOTES "Log in as root" | 5 | 5 | 25 | 1 (SSH), 7 | Key-only auth, no root login, named sudo user, fail2ban |
| R-02 | PostgreSQL listens on 0.0.0.0 with superuser password `postgres`; the app connects as superuser | PDF §4 "Database"; OPS_NOTES | 4 | 5 | 20 | 3, 7 | Bind to private network only, rotate, least-privilege roles (SEC-06) |
| R-03 | ... | | | | | | |
```

Requirements: **at least 12 risks** (aim for 15–20). Two are done for you above; the rest are yours.

Where to look for the rest:

- Each remaining row of the §4 table (Redis, secrets, TLS, deploys, logs, backups, monitoring,
  firewall, documentation, OS version, how the app is started).
- The leaked `.env`: one row listing every credential name in it, all marked "rotate in Stage 7".
- What you found in `scripts/`, the README symptoms and `OPS_NOTES.md`.
- Bus factor: what happens if Tomasz (who has the root password) is on holiday?

Finish with a short **"If I had one day"** paragraph naming the top 3 risks you'd fix first and
why. That's also "Prove you understand" question 6.

Commit, push, and open your first PR:

```bash
# laptop · Git Bash
git add docs/risk-register.md docs/learning-log.md
git commit -m "docs: risk register for the legacy box"
git push -u origin docs/stage1-risk-register
```

Then on GitHub: **Compare & pull request** → describe what and why → **Squash and merge**.

---

## Step 4 — Run TrackPoint locally, without Docker (2–3 h) → checkpoint C2

You already have Node 22, PostgreSQL 17 and Redis running on Windows. The point of doing this
without containers is to see exactly what the containers will later replace: three processes,
two backing services and a handful of environment variables.

> **📘 Learn: what you're about to run**
>
> - **The parts.** *api*: answers HTTP requests. *worker*: takes background jobs from a queue, so
>   slow work doesn't make HTTP requests wait. *web*: the pages customers see. *PostgreSQL*: the
>   database, the data you must never lose. *Redis*: a fast in-memory store, used here as the job
>   queue between the api and the worker.
> - **Configuration through environment variables.** The app reads its settings (database address,
>   port, keys) from environment variables, not from code. That's the "config" rule of the
>   *Twelve-Factor App* (12factor.net, worth 20 minutes): the same code runs everywhere, and only
>   the settings change. A `.env` file is a development convenience that loads those variables. It
>   is never committed. `.env.example` is the committed template, with fake values.
> - **Connection URLs** look like `postgres://user:password@host:port/database`. A typo in any part
>   is the most common reason an app won't start.
> - **Lockfile.** `package-lock.json` records the exact version of every dependency, and of their
>   dependencies. In 4.3 you'll find out how `npm ci` and `npm install` treat it differently.
> - **Unit vs integration tests.** Unit tests check one piece of code on its own, with no database
>   or network. Integration tests check the pieces working together against a real database and
>   Redis. That's why you must know *which* database they'll touch (4.5).
> - **Health endpoints.** Load balancers, Docker and Kubernetes regularly call a URL on your app to
>   decide two things: should traffic go to it, and should it be restarted? The contract in PDF §3
>   defines two endpoints, `/healthz` and `/readyz`, with different jobs. Read it before 4.6.

### 4.1 Create a dedicated database user (not the superuser)

> **📘 Learn:** in PostgreSQL a **role** is a user account, and a **database** is a separate set of
> tables owned by a role. The `postgres` role is the **superuser**: it can read, change or drop
> anything on the server. An app should connect with a role that owns only its own database
> (**least privilege**). Then a bug or a stolen password can only damage that one database.
> `psql` is the command-line client: `\l` lists databases, `\du` lists roles, `\q` quits.

```powershell
# laptop · PowerShell
psql -U postgres -h localhost
```

```sql
-- inside psql
CREATE ROLE trackpoint_dev LOGIN PASSWORD 'devpass-change-me';
CREATE DATABASE trackpoint      OWNER trackpoint_dev;
CREATE DATABASE trackpoint_test OWNER trackpoint_dev;
\q
```

> Forgot the `postgres` password from when you installed PostgreSQL? Search
> "reset postgres password windows pg_hba trust". Log what you did in the learning log.

### 4.2 Configure the app

```powershell
# laptop · PowerShell  (inside trackpoint\)
git switch main; git pull
Copy-Item .env.example app\api\.env
Copy-Item .env.example app\worker\.env
```

Edit both `.env` files so `DATABASE_URL=postgres://trackpoint_dev:devpass-change-me@localhost:5432/trackpoint`.

While the files are open, compare them with `app/api/src/config.js`. For every setting, ask:
*"if this variable were missing in production, what would happen?"* Write the answers in your
learning log. You'll come back to them in Step 9.

### 4.3 Start the three processes (three terminals)

```powershell
# laptop · PowerShell — terminal 1: API
cd app\api; npm ci; npm start
```

```powershell
# laptop · PowerShell — terminal 2: worker
cd app\worker; npm ci; npm start
```

```powershell
# laptop · PowerShell — terminal 3: web
cd app\web; npm start
```

If an install command fails, **read the error** before changing anything. `npm ci` and
`npm install` are different tools. Find out the difference and why CI pipelines insist on one of
them. That's a learning-log entry.

### 4.4 Use it like a customer would

```powershell
# laptop · PowerShell — terminal 4
curl.exe -s localhost:3000/healthz
curl.exe -s localhost:3000/readyz
curl.exe -s localhost:3000/api/version
curl.exe -s -X POST localhost:3000/api/parcels -H "content-type: application/json" -H "x-api-key: demo-key-northwind" -d '{\"recipient_name\":\"Ada Example\",\"recipient_address\":\"12 Sample Street, Springfield\",\"recipient_email\":\"ada@example.com\"}'
curl.exe -s localhost:3000/api/parcels/1
```

Then in a browser:

1. Open `http://localhost:8080` and track parcel `1`.
2. Open `http://localhost:8080/dashboard.html`, log in with `demo-key-northwind` and create a
   parcel. Watch the worker terminal pick up the jobs.
3. Open `http://localhost:3000/metrics` and find the request counters.

Draw (on paper is fine) the processes, ports and connections you just used. That picture becomes
your Stage 2 network diagram.

### 4.5 Run the tests

```powershell
# laptop · PowerShell
cd app\api
npm test
```

If anything fails, don't "fix" the test until you understand why it fails here and (presumably)
passed for the developer who wrote it.

<details><summary>Hints (open one level at a time)</summary>

- **L1:** Read the failing assertion's *expected* and *actual* values. What is different about
  your machine compared with the developers' machines?
- **L2:** Code that formats dates without saying *which* timezone uses the machine's local one.
  Servers, CI runners and containers usually run in UTC.
- **L3:** Look at the functions in `src/lib/eta.js` that read the hours, and at how the test
  builds its expected value. Decide whether the **code** or the **test** is wrong. A test that
  passes only in one city is telling you something about the code.

</details>

**Integration tests: read before you run.** Before running any test suite that talks to a
database, open it and answer two questions: *which database will this connect to, and what will
it do to it?* Do the same for Redis. Then run it against the **test** database and a separate
Redis DB number:

```powershell
# laptop · PowerShell  (in app\api)
$env:DATABASE_URL = "postgres://trackpoint_dev:devpass-change-me@localhost:5432/trackpoint_test"
$env:REDIS_URL    = "redis://localhost:6379/1"
npm run test:integration
Remove-Item Env:DATABASE_URL, Env:REDIS_URL
```

Does the integration suite pass reliably? Run it three times with the worker running and three
times with it stopped. Log what you see.

### 4.6 The health-check checkpoint

First, re-read the API contract table in §3 of the PDF: what must `/healthz` do, and what must
`/readyz` do? Write down the expected status code of each **when PostgreSQL is down**.

Then test it:

```powershell
# laptop · PowerShell (as Administrator)
Stop-Service postgresql-x64-17
curl.exe -s -w " -> %{http_code}\n" localhost:3000/healthz
curl.exe -s -w " -> %{http_code}\n" localhost:3000/readyz
```

While the database is down, also refresh the tracking page a few times and **watch the API
terminal**. Then:

```powershell
Start-Service postgresql-x64-17
curl.exe -s -w " -> %{http_code}\n" localhost:3000/readyz   # does it recover without a restart?
```

Repeat with Redis (`Stop-Service Redis`). The contract says what `/readyz` should do when Redis
is down too.

Record expected and actual for each case in the learning log. Every mismatch is a bug, so fix
each one on its own branch (`fix/healthz-...`) with a PR.

<details><summary>Hints</summary>

- **L1:** A liveness check answers "is this process alive?". Does `/healthz` need anything
  outside the process to answer that?
- **L2:** If liveness depends on the database, an orchestrator will *restart* healthy API
  processes during a database outage. That doesn't fix the database, and it does cause a
  restart storm. ("Prove you understand" question 1 is exactly this.)
- **L3:** `src/routes/health.js`. And for the "watch the API terminal" part: compare how the
  route handlers in `src/routes/parcels.js` handle errors with how the others do it. Express 4
  doesn't catch errors thrown inside `async` handlers by itself.

</details>

C2 is done when `npm test` is green, the integration suite passes against `trackpoint_test`, and
the health endpoints match the contract with PostgreSQL down, PostgreSQL back, Redis down and
Redis back.

---

## Step 5 — A Makefile for everyday commands (1 h) → requirement 7

> **📘 Learn: Make and linters**
>
> - **Make** is a build tool from 1976 that DevOps teams still use as a *menu of commands*.
>   `make test` is easier to remember than `cd app/api && npm test`, and it works the same on every
>   laptop and in CI. A Makefile is a list of **targets**. Each target is one line,
>   `name: prerequisites`, followed by indented **recipe** lines, which are shell commands.
>   `make check` first runs the targets `check` depends on (`lint`, `test`), then its own recipe.
>   Every recipe line runs in a fresh shell, which is why each line starts with its own `cd`.
> - A **linter** (here ESLint) reads code *without running it* and flags likely mistakes: unused
>   variables, misspelled names, unreachable code. It's cheap enough to run on every PR, and it
>   catches some bugs before the tests do.

`make` isn't installed on Windows. Pick one option:

- **Git Bash + make:** `winget install ezwinports.make`, then reopen Git Bash.
- **WSL:** you already have Ubuntu 24.04 in WSL, and `sudo apt install make` works there.

### 5.1 Add a real linter first

Your lint target needs a linter, and the repo doesn't have one yet:

```bash
# laptop · Git Bash
git switch -c chore/makefile-and-lint
cd app/api
npm install --save-dev eslint@9 @eslint/js@9 globals
```

Create `app/api/eslint.config.js`:

```js
const js = require('@eslint/js');
const globals = require('globals');

module.exports = [
  js.configs.recommended,
  {
    languageOptions: {
      ecmaVersion: 2023,
      sourceType: 'commonjs',
      globals: { ...globals.node },
    },
  },
];
```

Set `"lint": "eslint ."` in `app/api/package.json` and run `npm run lint`. A new linter on old
code always finds something. Fix each finding or disable that rule with a comment that says
why. Commit `package.json` **and** `package-lock.json` together.

### 5.2 The Makefile

Create `Makefile` in the repo root. Make insists on a **TAB** character at the start of each
recipe line, not spaces.

```make
# TrackPoint — common local tasks. Run `make` or `make help` to list targets.
API    := app/api
WORKER := app/worker
WEB    := app/web

.DEFAULT_GOAL := help
.PHONY: help install run-api run-worker run-web migrate test test-integration lint check

help: ## List available targets
	@grep -E '^[a-zA-Z_-]+:.*?## ' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "} {printf "  %-18s %s\n", $$1, $$2}'

install: ## Install dependencies for all apps (exactly as locked)
	cd $(API) && npm ci
	cd $(WORKER) && npm ci
	cd $(WEB) && npm ci

run-api: ## Run the API (reads app/api/.env)
	cd $(API) && npm start

run-worker: ## Run the worker (reads app/worker/.env)
	cd $(WORKER) && npm start

run-web: ## Run the web app locally
	cd $(WEB) && npm start

migrate: ## Apply database migrations
	cd $(API) && npm run migrate

test: ## Unit tests
	cd $(API) && npm test

test-integration: ## Integration tests (needs TEST_DATABASE_URL, never your dev DB)
	@test -n "$(TEST_DATABASE_URL)" || { echo "Set TEST_DATABASE_URL to a throwaway database"; exit 1; }
	cd $(API) && DATABASE_URL="$(TEST_DATABASE_URL)" REDIS_URL="$(or $(TEST_REDIS_URL),redis://localhost:6379/1)" npm run test:integration

lint: ## Lint the API
	cd $(API) && npm run lint

check: lint test ## Everything CI runs on a pull request
```

Try every target. If one fails, find out why before editing the Makefile. Each target is
supposed to work on a fresh clone. Then commit, push and open a PR.

Questions for your log:
- Why is the `test-integration` guard there?
- What does `.PHONY` do?
- Why does `help` use `$$1` instead of `$1`?

---

## Step 6 — Repository governance files (1.5 h) → requirement 1

> **📘 Learn: the files that turn a repo into a team repo**
>
> | File | What it does |
> |---|---|
> | `docs/branching-strategy.md` | The written rules: how branches are named, and how code reaches `main` and production |
> | `.gitignore` | Patterns for files Git should never offer to commit: dependencies, secrets, build output |
> | `.github/pull_request_template.md` | GitHub pre-fills every new PR with it, so every PR answers the same questions |
> | `.github/CODEOWNERS` | Maps paths to people. GitHub automatically asks them to review when those files change |
> | `.github/workflows/*.yml` | GitHub Actions: CI jobs that run on events such as a push or a PR |
>
> **GitHub Actions in five sentences.** A **workflow** is a YAML file that starts on an **event**
> (`on: pull_request`). It contains **jobs**, and each job runs on a fresh, throwaway **runner** VM
> (`runs-on: ubuntu-24.04`). A job is a list of **steps**. A step is either a shell command
> (`run:`) or a reusable **action** (`uses: actions/checkout@v4` clones your repo onto the runner).
> If any step exits non-zero, the job fails and the PR shows a red ✗.
>
> YAML tip: indentation is meaningful, and it must be spaces, never tabs (the opposite of Make).

Use one branch for all of these:

```bash
# laptop · Git Bash
git switch main && git pull && git switch -c chore/repo-governance
```

### 6.1 `docs/branching-strategy.md`

Write it **in your own words**. Use this skeleton and answer every prompt:

```markdown
# Branching strategy

## Model
Trunk-based development: `main` is always deployable. Work happens on short-lived branches
(< 2 days) merged through pull requests. Why this model and not GitFlow, for a team of 4?

## Branch names
feat/…, fix/…, docs/…, chore/…, hotfix/…  (lowercase, hyphens)

## Commits
Conventional Commits style (`fix: …`, `feat: …`). PRs are squash-merged, so the PR title
becomes the commit message on main.

## Pull requests
- What must every PR contain? (see the PR template)
- Required checks: which ones, and what happens if a check is flaky?
- Reviews: who reviews while I'm the only infra person? (be honest about this)

## Hotfixes
Step by step, how does an urgent fix reach production? (Branch from where? Which checks may be
skipped? Hint: none. How is it recorded?) This is "Prove you understand" question 5.

## Releases
How a version is identified (commit SHA now; tags later in Stage 4).
```

### 6.2 `.gitignore`

Don't copy a giant template. Build it from evidence:

1. Run the API, worker, tests and `npm run build` in `app/web`, then run `git status --short`.
   **Every untracked file you didn't create on purpose** is a candidate.
2. Then think ahead to files that will appear in later stages.

Cover each category:

- dependencies
- environment files (keep `.env.example` tracked: `!.env.example`)
- logs
- build output
- runtime data the apps write
- database dumps and backups
- private keys and certificates
- Terraform state and `.terraform/`
- editor and OS junk

Check your patterns with `git check-ignore -v <path>`.

> `.gitignore` does **not** untrack files that are already committed. If something already in the
> repo matches a new pattern, `git rm --cached <file>` stops tracking it. Removing a file from the
> current tree does not remove it from history, though. You saw that in Step 2.2.

### 6.3 `.github/pull_request_template.md`

```markdown
## What and why
<!-- What changes, and what problem it solves. Link the issue/ticket. -->

## How I tested it
<!-- Commands run, screenshots, environments. "Not tested" is an allowed answer, but say it. -->

## Risk and rollback
- [ ] Database migration included? If yes, it is backward compatible with the previous release
- [ ] New/changed environment variables documented in `.env.example`
- [ ] No secrets, credentials or personal data in this diff
- Rollback plan:

## Checklist
- [ ] `make check` passes locally
- [ ] Docs / runbooks updated if behaviour changed
```

### 6.4 `.github/CODEOWNERS`

```
# Owners are automatically requested for review when a PR touches their files.
# Later matches take precedence over earlier ones.
*                  @JamilAhmadRupak

/.github/          @JamilAhmadRupak
/deploy/           @JamilAhmadRupak
/infra/            @JamilAhmadRupak
/docs/runbooks/    @JamilAhmadRupak
```

Today you own everything. Write a comment explaining what you'd change when the developers
join. (Who should own `app/web/`?)

### 6.5 A minimal PR check: `.github/workflows/pr-checks.yml`

Checkpoint C1 needs "PRs require a passing check", so you need at least one check. Keep it
minimal; Stage 4 replaces it with the real pipeline.

```yaml
name: pr-checks

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  api-checks:
    runs-on: ubuntu-24.04
    defaults:
      run:
        working-directory: app/api
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: app/api/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm test
```

> In Stage 4 you'll pin actions to commit SHAs (CI-08) instead of `@v4`. Note in your log why
> a tag like `@v4` can change under you.

Also read the existing `.github/workflows/ci.yml` carefully. Decide whether you want it running
on every push while you work on Stage 1, and record the decision and your reasoning in the log.
You'll rewrite CI properly in Stage 4.

Commit everything, push, and open the PR. The `api-checks` job runs on the PR. It has to go green
before you continue, so if it fails, go back to Step 4.5.

---

## Step 7 — Protect `main` (30 min) → checkpoint C1

> **📘 Learn: branch protection.** These are rules GitHub enforces *on its servers*, whatever
> anyone's laptop does. For example: "nobody pushes to `main` directly; changes arrive only through
> PRs, and only after check X is green." **Rulesets** are the newer form of branch protection rules.
> A rule that isn't enforced is just a suggestion. This step turns your Step 1 habit into a
> guarantee.

### 7.1 A plan limitation you need to know about

On **GitHub Free**, branch protection and rulesets are only *enforced* on **public**
repositories. `trackpoint` is private, so on a free account the rules won't apply. Your options:

1. **GitHub Student Developer Pack** (free with a student email, gives GitHub Pro):
   education.github.com/pack. This is the best option if you're eligible.
2. **Make `trackpoint` public.** The leaked secrets and the customer dump in it are fake, but
   scanners will still flag them, and the history-cleaning work is planned for Stage 7.
   Decide consciously and log the decision.
3. Neither: document the limitation honestly in your learning log and README, and enforce the
   rule by habit until you can enable it. Graders accept explained gaps; they don't accept
   hidden ones.

### 7.2 Create the ruleset

github.com → trackpoint → **Settings → Rules → Rulesets → New ruleset → New branch ruleset**:

| Setting | Value | Why |
|---|---|---|
| Name | `protect-main` | |
| Enforcement status | **Active** | |
| Bypass list | **empty** | Rules that admins can skip aren't rules |
| Target branches | **Include default branch** | |
| Restrict deletions | ✅ | |
| Require a pull request before merging | ✅, required approvals **0** | You can't approve your own PR; with 1 required approval a solo developer is stuck. Raise it when the devs join. |
| Require status checks to pass | ✅, add **`api-checks`** | The check only shows up in the search box after it has run at least once (Step 6.5) |
| Block force pushes | ✅ | Protects history |

### 7.3 Prove it works

```bash
# laptop · Git Bash
git switch main && git pull
git commit --allow-empty -m "test: direct push should be rejected"
git push origin main          # expected: rejected with "GH013: Repository rule violations"
git reset --hard origin/main  # throw the test commit away
```

Then open a PR that deliberately breaks a unit test. GitHub should block the merge button. Close
that PR without merging. Screenshot both results for your evidence folder (`docs/evidence/stage1/`).

---

## Step 8 — A server you can trust, on Oracle Cloud (3–4 h) → checkpoint C3

> **📘 Learn: the cloud pieces you're about to click**
>
> - **Region:** a city-level group of data centres (Mumbai, Singapore). **Availability domain:** one
>   data centre, or a group of them, inside a region.
> - **VCN** (Virtual Cloud Network; AWS calls it a VPC): your own private network in the cloud.
>   A **subnet** is a slice of its IP range. A **public** subnet has a route to the internet through
>   an **internet gateway**.
> - **Instance / shape / image:** the VM itself, its size (CPU + RAM), and the operating system it
>   boots from.
> - **Private vs public IP:** the private IP (`10.x.x.x`) only works inside the VCN. The public IP is
>   how the internet reaches you.
> - **Security list:** a firewall Oracle applies *outside* the VM, on the network. The VM also has
>   its own firewall *inside* it (8.9).

### 8.1 Where the server runs: Oracle Cloud Always Free

You're using **one Oracle Cloud Always Free Arm VM** (Ubuntu 24.04) for this stage. It's a real
server with a public IP, so mistakes are real too. Three things are different from a plain VPS:

| Difference | What it means for you |
|---|---|
| **Arm CPU** (Ampere A1) | Irrelevant in Stage 1. From Stage 2, images must be built for `arm64`. |
| **Two firewalls** | Oracle's *security list* (in the cloud) **and** `iptables` rules on the VM. A port is open only if both allow it. |
| **Serial console** | A way in that doesn't use SSH. It's your safety net if you break `sshd`. |

Alternatives, if Oracle doesn't work out: a budget VPS (Hetzner or DigitalOcean, US$4–8 a month)
or a local VirtualBox VM. The steps from 8.6 on are almost identical everywhere.

### 8.2 Create your SSH key first (laptop)

> **📘 Learn: SSH and its two kinds of keys**
>
> - **SSH** gives you an encrypted terminal on a remote machine. `ssh` is the client on your laptop;
>   `sshd` is the server program on the VM.
> - **Your user key pair** proves *who you are*. The public half (`id_ed25519.pub`) goes on the
>   server, in `~/.ssh/authorized_keys`. The private half (`id_ed25519`) never leaves your laptop.
>   To log in, the server sends a challenge that only the private key can sign, so no password
>   ever crosses the network.
> - The **passphrase** encrypts the private key file on your disk. A stolen laptop then doesn't
>   mean a stolen server.
> - The **host key pair** belongs to the server and proves *which machine* you reached. Your laptop
>   remembers it in `~/.ssh/known_hosts` after the first connection. (8.6 asks you why the first
>   connection deserves care.)
> - `ed25519` is the key algorithm: modern, short and fast. Prefer it over RSA for new keys.

Oracle asks for your public key when it creates the VM, so make it now.

```powershell
# laptop · PowerShell
ssh-keygen -t ed25519 -C "rupak@laptop"     # accept the path, SET A PASSPHRASE
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub   # one line, starts with ssh-ed25519. You'll paste it in 8.5
```

The **public** key (`.pub`) goes to Oracle. The private key (`id_ed25519`, no extension) never
leaves your laptop.

### 8.3 Sign up and lock down the Oracle account

1. Go to **cloud.oracle.com → Sign up**. A card is needed for identity checks. The Always Free tier
   doesn't charge it.
2. **Home region:** pick **India West (Mumbai)**, **India South (Hyderabad)** or **Singapore**.
   This choice is **permanent**, and Always Free compute only runs in your home region.
3. **MFA on your Oracle login.** Oracle usually prompts you to enrol on first sign-in. If it
   doesn't, find it under your profile or the Identity & Security settings. Whoever controls this
   account controls every server in it, so this account is the new "root password on a sticky
   note" (R-16).
4. **Check your real Arm allowance.** Menu → *Governance & Administration* → *Limits, Quotas and
   Usage* → service **Compute**. Search for `A1`. Write the core and memory limits in your learning
   log, because they changed in 2026 and your number is the one that counts.
5. **Budget alert.** Menu → *Billing & Cost Management* → *Budgets* → *Create budget*: amount
   **US$1**, with an alert on *actual* spend. If anything ever costs money by mistake, you find out
   that day, not at the end of the month.

### 8.4 Network: a VCN with a public subnet

Menu → *Networking* → *Virtual cloud networks* → **Start VCN Wizard** → *Create VCN with Internet
Connectivity*. Name it `tp-vcn`, keep the defaults, and click **Create**.

Then open `tp-vcn` → *Subnets* → the **public** subnet → its **Default Security List** and read the
ingress rules. You should see TCP 22 from `0.0.0.0/0`, plus some ICMP. Leave them. Your home IP
probably changes, so restricting port 22 to it would lock you out. Key-only SSH is the protection
(8.8).

Log question: *A security list and a host firewall both filter traffic. What can each one see or do
that the other can't?*

### 8.5 Create the VM

Menu → *Compute* → *Instances* → **Create instance**:

| Field | Value |
|---|---|
| Name | `tp-stage1` (it also becomes the default hostname) |
| Placement | default availability domain |
| Image | *Change image* → Ubuntu → **Canonical Ubuntu 24.04** (not "Minimal") |
| Shape | *Change shape* → Ampere → **VM.Standard.A1.Flex**, **1 OCPU, 6 GB** (keep the rest of your allowance for Stages 9–10) |
| Networking | `tp-vcn`, the **public** subnet, *Automatically assign public IPv4 address*: **yes** |
| SSH keys | *Paste public keys* → paste the line from 8.2 |
| Boot volume | default size is fine |

Click **Create**, wait for **RUNNING**, and copy the **Public IP address**.

<details><summary>"Out of host capacity"?</summary>

The free Arm machines are popular. Try another availability domain, or try again at a quiet time
(early morning, your time). If you're stuck for days, the small AMD shape
`VM.Standard.E2.1.Micro` (1 GB, x86) is enough for Stage 1, but plan to move to Arm before Stage 2.

</details>

### 8.6 First login, and your personal admin user

> **📘 Learn: Linux users and sudo**
>
> - **root** (user id 0) can do anything on the machine. Normal users can't change system files.
> - **sudo** lets a user in the `sudo` group run *one* command as root, after typing *their own*
>   password, and it logs who ran what. A shared root password gives you none of that: you can't
>   tell who did something, and you can't remove one person's access without changing it for all.
> - `adduser` creates a person's account. `usermod -aG sudo rupak` **a**ppends the user to the
>   sudo **G**roup. `install -d -m 700 -o rupak` creates a directory with a given mode (permission
>   bits) and owner in one command.
> - `apt` is Ubuntu's package manager. `apt update` refreshes the list of available packages;
>   `apt full-upgrade` installs the new versions, including security fixes.

**Verify the host key, don't just trust it.** On the instance page, open *Console history* →
*Capture console history*, and look for the block `BEGIN SSH HOST KEY FINGERPRINTS`. That's the
VM telling you its fingerprint through a channel an attacker can't sit in. Now connect:

```powershell
# laptop · PowerShell
ssh ubuntu@<PUBLIC_IP>
```

Compare the `ED25519` fingerprint SSH shows you with the one from the console before you type
`yes`. Write down what that comparison protects you from. That's "Prove you understand" Q3.

```bash
# server (as ubuntu)
sudo apt update && sudo apt full-upgrade -y
sudo adduser rupak                 # set a password: used for sudo and the serial console, never for SSH
sudo usermod -aG sudo rupak
sudo install -d -m 700 -o rupak -g rupak /home/rupak/.ssh
sudo install -m 600 -o rupak -g rupak ~/.ssh/authorized_keys /home/rupak/.ssh/authorized_keys
[ -f /var/run/reboot-required ] && sudo reboot      # a new kernel needs a reboot
```

Create `C:\Users\HP\.ssh\config` so you can type `ssh tp`:

```
Host tp
    HostName <PUBLIC_IP>
    User rupak
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

```powershell
# laptop · PowerShell
ssh tp                    # asks for the key passphrase, not the account password
```
```bash
# server (as rupak)
sudo whoami               # asks for rupak's password, then prints: root
```

**Do not continue until both work.**

### 8.7 Your safety net: the serial console (before you touch `sshd`)

Instance page → *Resources* → **Console connection** → **Launch Cloud Shell connection**. A
terminal opens in the browser. Press **Enter** until you see a `login:` prompt, then log in as
`rupak` with the password from 8.6. Type `exit` and close it.

That worked without SSH, which is the point. The console is a virtual serial port wired straight
into the VM, so a broken `sshd_config` or firewall rule can't block it. It **does** need a user with
a password, so never lock or delete `rupak`'s password.

### 8.8 Harden SSH, without locking yourself out

**Golden rule:** keep one SSH session open the whole time, and test changes in a **second**
session. The serial console (8.7) is your last resort.

Oracle's image already blocks password logins, and root's `authorized_keys` just prints "Please
login as the user ubuntu". So why write your own config? Because a default you didn't choose isn't
a decision. Your file states the intent, survives image changes, and adds `AllowUsers`.

```bash
# server (as rupak)
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null <<'EOF'
# TrackPoint SSH hardening (SEC-01)
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
AllowUsers rupak
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
EOF

sudo sshd -t                     # syntax check; silence means OK
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|allowusers) '
sudo systemctl restart ssh
```

Why the `10-` prefix? `sshd` uses the **first** value it reads for most settings, and it reads
`sshd_config.d/*.conf` in alphabetical order. Cloud images often ship `50-cloud-init.conf`, and `10-`
beats it. `sshd -T` shows the *effective* values.

Now open a **second** terminal, run `ssh tp`, and check it works. Then retire the default `ubuntu`
account. It's a shared, well-known username with passwordless sudo:

```bash
# server (as rupak, from the second session)
sudo cat /etc/sudoers.d/90-cloud-init-users     # read it: who gets sudo WITHOUT a password?
sudo deluser --remove-home ubuntu
sudo rm /etc/sudoers.d/90-cloud-init-users      # only if it mentioned ubuntu and nothing else
sudo visudo -c                                  # sudo config still valid?
```

### 8.9 The two firewalls

> **📘 Learn: how a Linux firewall works.** A firewall decides which network packets may come in
> or go out. On Linux the kernel does the filtering itself (*netfilter*). `iptables`, `nftables` and
> `ufw` are only different tools for writing its rules. Rules are checked from top to bottom and the
> **first match wins**, so a final "REJECT everything else" catches whatever wasn't explicitly
> allowed above it. That pattern is called **default deny**: list what is allowed, refuse the rest.

Look at what Oracle put on the VM:

```bash
# server
sudo iptables -L INPUT -n -v --line-numbers
sudo cat /etc/iptables/rules.v4
```

You should see roughly: accept established traffic, accept ICMP and loopback, **accept TCP 22**, and
then **REJECT everything else**. You'll also see rules mentioning `169.254.0.0/16`. **Never delete
those.** They're how the VM reaches Oracle's metadata service, DNS, time and storage.

Rules for this stage:

- **Don't enable `ufw`.** It writes to the same tables as Oracle's rules, and two tools managing one
  firewall is how ports end up mysteriously open or closed. The brief schedules the host-firewall
  work for Stage 7, and in Stage 3 you'll open 80 and 443 in **both** layers.
- Nothing else needs to be reachable now. The API will listen on `127.0.0.1` (Step 9).

Experiment for the log:

```powershell
# laptop · PowerShell
Test-NetConnection <PUBLIC_IP> -Port 22      # TcpTestSucceeded : True
Test-NetConnection <PUBLIC_IP> -Port 5432    # False. Which layer(s) blocked it? How could you tell?
```

### 8.10 Hostname, timezone, time sync

> **📘 Learn:** **cloud-init** is the program that sets up a cloud VM when it first boots. It creates
> the default user, installs your SSH key and sets the hostname. Some of its modules run again at
> *every* boot, and that's how it can quietly undo your changes. **NTP** keeps the clock right by
> asking time servers. Correct time matters for logs, TLS certificates and anything else that
> compares timestamps between machines.

On a cloud image, **cloud-init** may reset the hostname and `/etc/hosts` at every boot. Tell it not
to:

```bash
# server
sudo tee /etc/cloud/cloud.cfg.d/99-trackpoint.cfg > /dev/null <<'EOF'
preserve_hostname: true
manage_etc_hosts: false
EOF

sudo hostnamectl set-hostname tp-stage1
grep -q 'tp-stage1' /etc/hosts || echo '127.0.1.1 tp-stage1' | sudo tee -a /etc/hosts
sudo timedatectl set-timezone UTC
timedatectl                 # look for: Time zone UTC, "System clock synchronized: yes", "NTP service: active"
timedatectl timesync-status 2>/dev/null || chronyc tracking    # which tool syncs the clock on this image, and how far off it is
```

Why UTC on servers? Write the answer in your log, connecting it to what you found in Step 4.5.

### 8.11 Prove C3 from a second terminal

```powershell
# laptop · PowerShell
ssh root@<PUBLIC_IP>                                                                  # expect: Permission denied (publickey)
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password rupak@<PUBLIC_IP>  # expect: Permission denied (publickey)
ssh ubuntu@<PUBLIC_IP>                                                                # expect: Permission denied (publickey)
ssh tp "whoami; hostname"                                                             # expect: rupak / tp-stage1
```

Save the outputs to `docs/evidence/stage1/ssh-checks.txt`. Then reboot once (`sudo reboot`) and
check `ssh tp hostname` still says `tp-stage1`. That proves the cloud-init change worked.

Now look at the server's side. Your VM has been on the internet for an hour or two:

```bash
# server
sudo journalctl -u ssh --since "24 hours ago" --no-pager | grep -c "Invalid user"
sudo journalctl -u ssh --since "24 hours ago" --no-pager | grep -oP 'Invalid user \K\S+' | sort | uniq -c | sort -rn | head
```

Those are bots guessing usernames, and probably passwords. Write a log entry connecting this to
risk R-01 on the legacy box (root + password on port 22). Your register's likelihood score of 5 now
has evidence behind it.

### 8.12 Keep it alive, keep it free

- **Stop vs terminate.** *Reboot* and *Stop* keep the disk and the public IP. *Terminate* deletes
  the VM and its IP (and the disk too, if you tick that box). In Stage 3 you'll switch to a
  **reserved** public IP so DNS doesn't break when you rebuild.
- **Idle reclaim.** Oracle may stop Always Free VMs that stay almost idle for about a week. You have
  two options: upgrade the account to *Pay As You Go* (free resources stay free; your US$1 budget
  alert from 8.3 is the guard), or accept it and **rebuild from your runbook**, which is exactly
  what Step 13 asks you to practise anyway.
- **Check the bill** once a week: *Billing & Cost Management* → *Cost analysis* should show
  US$0.00.

### 8.13 Write it down as you go (the "everything is code" rule)

Copy every server command you run into `docs/runbooks/server-setup.md` in order, with a short
reason for each. Include the **console clicks** from 8.3–8.5 as numbered steps too. In Step 13
you'll rebuild the server by following that file literally. In Stage 5, Terraform (the Oracle
provider) replaces the clicks, and Ansible replaces the commands.

---

## Step 9 — Run the API under systemd (2–3 h) → checkpoint C4

> **📘 Learn: systemd**
>
> - **systemd** is the first process Linux starts (PID 1). It starts everything else, and keeps
>   watching it.
> - It manages **units**. A `.service` unit describes one program: what to run (`ExecStart=`), as
>   which user, with which environment, and what to do when it exits (`Restart=`).
> - A unit file has three sections. `[Unit]`: description, and ordering against other units
>   (`After=`, `Wants=`). `[Service]`: how to run the program. `[Install]`: when it should start at
>   boot (`WantedBy=multi-user.target` roughly means "a normal boot").
> - `systemctl start` runs it now. `systemctl enable` makes it start at every boot. `enable --now`
>   does both. After editing a unit file, run `systemctl daemon-reload` so systemd re-reads it.
> - Everything the service prints to stdout/stderr goes to the **journal**. Read it with
>   `journalctl -u <name>`. No log files to manage.
> - The **hardening** lines (`ProtectSystem=`, `NoNewPrivileges=`, …) put the service in a sandbox.
>   Each line takes away something the app shouldn't need. `man systemd.exec` explains each one.

### 9.1 Install the runtime and backing services

```bash
# server
sudo apt install -y postgresql redis-server git curl

# Node.js 22 from NodeSource. Download the script, READ it, then run it.
curl -fsSL https://deb.nodesource.com/setup_22.x -o /tmp/nodesource_setup.sh
less /tmp/nodesource_setup.sh
sudo bash /tmp/nodesource_setup.sh
sudo apt install -y nodejs
node -v

sudo ss -tlnp | grep -E ':(5432|6379)'    # which addresses do PostgreSQL and Redis listen on?
```

Compare that `ss` output with the legacy box in §4. Note the difference in the log.

### 9.2 Database role and password

```bash
# server
DBPASS=$(openssl rand -hex 24)     # hex: no characters that need URL-escaping
echo "$DBPASS"                     # you'll paste it into the env file in 9.4, then forget it
sudo -u postgres psql -c "CREATE ROLE trackpoint LOGIN PASSWORD '$DBPASS';" \
                      -c "CREATE DATABASE trackpoint OWNER trackpoint;"
```

(Stage 3 splits this into migration, app and read-only roles; SEC-06.)

### 9.3 A dedicated system user, and the code

> **📘 Learn: file ownership and permissions**
>
> `ls -l` prints something like `-rw-r----- 1 root trackpoint 512 api.env`. That file is owned by
> user **root** and group **trackpoint**. The nine letters are three sets of **r**ead / **w**rite /
> e**x**ecute: for the owner, for the group, and for everyone else. In numbers r=4, w=2, x=1, so
> `640` means owner rw (6), group r (4), others nothing (0). On a *directory*, `x` means "may
> enter it". `chown` changes the owner, `chmod` changes the bits.
>
> A **system user** (`useradd --system`) exists only to run one service: it has no password, no
> home directory and no login shell (`/usr/sbin/nologin`).

```bash
# server
sudo useradd --system --no-create-home --home-dir /nonexistent --shell /usr/sbin/nologin trackpoint
getent passwd trackpoint
```

Get the code onto the server. This is a manual "deploy", which is fine for Stage 1; Stage 4
automates it.

```bash
# laptop · Git Bash  (inside trackpoint/, on an up-to-date main)
git archive --format=tar.gz -o /tmp/trackpoint.tar.gz HEAD
scp /tmp/trackpoint.tar.gz tp:/tmp/
```

```bash
# server
sudo mkdir -p /opt/trackpoint
sudo tar -xzf /tmp/trackpoint.tar.gz -C /opt/trackpoint
sudo chown -R root:root /opt/trackpoint     # code is owned by root: the service can read it, not change it
cd /opt/trackpoint/app/api && sudo npm ci --omit=dev
```

Why is the code owned by `root` and not by `trackpoint`? Think about what an attacker who takes
over the API process could do if the process could rewrite its own code. Write it in the log
(it feeds "Prove you understand" question 2).

### 9.4 Configuration file (secrets live outside the code directory)

```bash
# server
sudo install -d -m 750 -o root -g trackpoint /etc/trackpoint
sudoedit /etc/trackpoint/api.env
```

Use your notes from Step 4.2 (every setting in `config.js`, and what happens if each is missing)
to write this file. At minimum:

```ini
NODE_ENV=production
HOST=127.0.0.1
PORT=3000
DATABASE_URL=postgres://trackpoint:<DBPASS>@127.0.0.1:5432/trackpoint
REDIS_URL=redis://127.0.0.1:6379
ADMIN_TOKEN=<output of: openssl rand -hex 32>
# ...every other setting config.js reads, set deliberately. Don't rely on a default you haven't read.
```

```bash
# server
sudo chown root:trackpoint /etc/trackpoint/api.env
sudo chmod 640 /etc/trackpoint/api.env
```

Why `HOST=127.0.0.1`? There's no reverse proxy yet, so nothing outside the server needs to reach
port 3000.

### 9.5 The unit file

Commit this to your repo as `deploy/systemd/trackpoint-api.service`, then install it:

```ini
[Unit]
Description=TrackPoint API
Documentation=https://github.com/JamilAhmadRupak/trackpoint
After=network-online.target postgresql.service redis-server.service
Wants=network-online.target
# give up if it crashes 5 times within 60 s (no infinite crash loop)
StartLimitIntervalSec=60
StartLimitBurst=5

[Service]
Type=simple
User=trackpoint
Group=trackpoint
WorkingDirectory=/opt/trackpoint/app/api
EnvironmentFile=/etc/trackpoint/api.env
ExecStart=/usr/bin/node src/server.js
Restart=on-failure
RestartSec=3
TimeoutStopSec=25

# --- hardening (man systemd.exec) ---
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
LockPersonality=true
CapabilityBoundingSet=
# ReadWritePaths=   <- only if the app has a legitimate reason to write somewhere (see below)

[Install]
WantedBy=multi-user.target
```

```bash
# server
sudo cp /opt/trackpoint/deploy/systemd/trackpoint-api.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now trackpoint-api
systemctl status trackpoint-api --no-pager
journalctl -u trackpoint-api -f          # Ctrl+C to stop following
curl -s localhost:3000/readyz; echo
```

It may not start cleanly the first time. That's expected, and it's the most educational part of
the stage.

<details><summary>Hints if the service won't start or keeps restarting</summary>

- **L1:** `journalctl -u trackpoint-api -n 50 --no-pager` always has the answer. Find the
  *first* error, not the last.
- **L2:** With `ProtectSystem=strict`, the entire filesystem is read-only for this service. If
  the error mentions a path, ask two questions: *which code writes there?* and *should a
  production service write there at all, or is it doing something that belongs somewhere else
  (stdout, a database, object storage)?*
- **L3:** Look for filesystem writes in `src/`, and for the configuration that controls where
  they go. Fixing it with `ReadWritePaths=` is allowed, but justify it in the log. Changing the
  behaviour so the write isn't needed is often the better fix.

</details>

Run `systemd-analyze security trackpoint-api.service`. It scores how exposed the service is.
Record the score, and try to improve it by one notch with another directive.

### 9.6 The reboot test (C4)

```bash
# server
sudo reboot
```

```powershell
# laptop · PowerShell — wait ~30 s, then:
ssh tp "systemctl is-enabled trackpoint-api; systemctl is-active trackpoint-api; curl -s localhost:3000/readyz"
ssh tp "systemd-analyze critical-chain trackpoint-api.service"
```

The service should be `enabled` and `active`, `/readyz` should return OK, and you never logged
in to start anything. Save the output as evidence.

**Bonus:** do the same for the worker (`deploy/systemd/trackpoint-worker.service`). Anything that
blocks you there is worth a learning-log entry.

---

## Step 10 — Server inventory, produced by commands (1 h) → checkpoint C6

> **📘 Learn: listening sockets and bind addresses**
>
> A server program **listens** on an address and a port, waiting for connections. The address
> decides who can connect:
>
> - `127.0.0.1` (*localhost*, the loopback interface): only programs on the same machine.
> - `0.0.0.0` (shown as `*`, or `[::]` for IPv6): every network interface. Anyone who can reach the
>   machine can connect, if the firewalls let them through.
>
> `ss -tulpn` lists them: **t**cp, **u**dp, **l**istening only, the owning **p**rocess, **n**umbers
> instead of names. An inventory produced by a script is better than one typed by hand: you can
> run it again tomorrow and `diff` the two.

Add `scripts/server-inventory.sh` to the repo:

```bash
#!/usr/bin/env bash
# Prints a Markdown inventory of this server. Run with sudo:
#   sudo bash server-inventory.sh > inventory.md
set -euo pipefail

section() {
  local title=$1; shift
  printf '\n## %s\n\n```\n' "$title"
  "$@" 2>&1 || true
  printf '```\n'
}

echo "# Server inventory: $(hostname)"
echo
echo "_Generated $(date -u +%Y-%m-%dT%H:%M:%SZ) by scripts/server-inventory.sh_"

section "Operating system"        bash -c 'grep PRETTY_NAME /etc/os-release; uname -r; uptime -p'
section "Time"                    timedatectl
section "CPU and memory"          bash -c 'echo "vCPUs: $(nproc)"; free -h'
section "Block devices"           lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
section "Filesystem usage"        df -hT -x tmpfs -x devtmpfs -x squashfs -x efivarfs
section "Listening sockets"       ss -tulpn
section "Running services"        systemctl list-units --type=service --state=running --no-pager --no-legend
section "Enabled services"        systemctl list-unit-files --type=service --state=enabled --no-pager --no-legend
section "Users with a login shell" bash -c "getent passwd | awk -F: '\$7 !~ /(nologin|false)\$/ {printf \"%-12s uid=%-6s shell=%s\n\", \$1, \$3, \$7}'"
section "sudo group members"      getent group sudo
section "SSH effective settings"  bash -c "sshd -T | grep -Ei '^(port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|allowusers) '"
section "Pending package updates" bash -c 'apt list --upgradable 2>/dev/null | tail -n +2'
```

```bash
# laptop · Git Bash
scp scripts/server-inventory.sh tp:/tmp/
ssh -t tp "sudo bash /tmp/server-inventory.sh > /tmp/inventory.md"
scp tp:/tmp/inventory.md docs/server-inventory.md
```

Then, **by hand**, add a section at the top of `docs/server-inventory.md`:

```markdown
## What each listening port is, and why it's there
| Port | Process | Bound to | Reachable from | Why it exists |
|------|---------|----------|----------------|---------------|
| 22   | sshd    | 0.0.0.0  | anywhere (Oracle security list + iptables allow only 22) | admin access |
| ...  |         |          |                |               |
```

Every row must be explainable. "I don't know what this is" means investigate it
(`systemctl status <unit>`, `man <program>`). Pay particular attention to **0.0.0.0 versus
127.0.0.1**: that one column decides what the internet can reach.

---

## Step 11 — Break it on purpose (2 h)

For each experiment, add a learning-log entry: **expected → actual → why**. Take a boot-volume backup first (instance page → *Boot volume* → *Boot volume backups* → *Create*; Always Free includes a few).

> **📘 Learn: why break things on purpose?** Outages always arrive at a bad time. If the first time
> you see "No space left on device" is during a real incident, you're learning under pressure. If
> you've already caused it yourself on a test box, you recognise it in seconds. Each experiment is a
> small controlled outage: predict, break, observe, explain.

### 11.1 Signals and systemd

> **📘 Learn: signals.** A **signal** is a short message the kernel delivers to a process.
> `SIGTERM` means "please shut down" (it's what `kill` sends by default, and what `systemctl stop`
> sends first). `SIGINT` is what Ctrl+C sends. `SIGKILL` ends the process immediately; the process
> can't catch it, so it gets no chance to clean up. Programs can install **handlers** for TERM and
> INT to finish in-flight work before exiting. When a process ends, systemd records *how*: a normal
> exit with an exit code, or killed by a signal. `Restart=` decides what to do based on that.

```bash
# server
for SIG in TERM INT KILL; do
  PID=$(systemctl show -p MainPID --value trackpoint-api)
  echo "== sending SIG$SIG to $PID"
  sudo kill -s "$SIG" "$PID"
  sleep 6
  systemctl show trackpoint-api -p ActiveState -p SubState -p NRestarts -p ExecMainStatus
  journalctl -u trackpoint-api -n 8 --no-pager
done
```

For each signal: did systemd restart the service? What did the app log, or did it log nothing?
Compare with the table under `Restart=` in `man systemd.service`. Then find the signal handlers
in `src/server.js` and explain what you saw.

### 11.2 The service user can't read the app directory

```bash
# server
sudo chmod 700 /opt/trackpoint/app/api
sudo systemctl restart trackpoint-api; sleep 2
systemctl status trackpoint-api --no-pager | head -15
sudo chmod 755 /opt/trackpoint/app/api          # restore!
sudo systemctl reset-failed trackpoint-api; sudo systemctl restart trackpoint-api
```

Where did the error appear: the app's own log, or systemd? What does the `status=` code mean?
Why?

### 11.3 A full disk

> **📘 Learn: mounting.** Linux has one directory tree, starting at `/`. A disk or partition gets
> attached ("**mounted**") at a directory, and its files then appear under that directory. `df -h`
> shows each mounted filesystem and how full it is. A **loop device** lets an ordinary file act like
> a disk. That's how the commands below create a tiny 100 MB disk you can fill safely.

Don't fill your root disk (unless you have a fresh boot-volume backup to restore). Use a small test partition:

```bash
# server
sudo fallocate -l 100M /var/tmp/small.img
sudo mkfs.ext4 -q /var/tmp/small.img
sudo mkdir -p /mnt/small && sudo mount -o loop /var/tmp/small.img /mnt/small
df -h /mnt/small
```

Pick something on this server that writes to disk (candidates are in your inventory and in the
app's configuration), point it at `/mnt/small`, then fill the partition:

```bash
sudo dd if=/dev/zero of=/mnt/small/filler bs=1M status=progress   # runs until "No space left on device"
```

Observe what breaks first, and how it fails: a loud error, or silently? Clean up afterwards:

```bash
sudo umount /mnt/small && sudo rm /var/tmp/small.img
```

Revert whatever you re-pointed, too. **Optional:** only with a fresh boot-volume backup, fill `/` with
`fallocate -l <size> /bigfile` and see which service complains first: PostgreSQL, journald or SSH.

### 11.4 Port already in use

```bash
# server
sudo systemctl stop trackpoint-api
python3 -m http.server 3000 --bind 127.0.0.1 &
sudo systemctl start trackpoint-api; sleep 8
journalctl -u trackpoint-api -n 15 --no-pager
sudo ss -ltnp 'sport = :3000'                 # who owns the port?
systemctl show trackpoint-api -p NRestarts -p Result
kill %1; sudo systemctl reset-failed trackpoint-api; sudo systemctl start trackpoint-api
```

What did `StartLimitBurst` do here?

### 11.5 Merge conflicts, and rebase versus merge

> **📘 Learn: the vocabulary.** Both commands combine work from two branches. A **merge** creates
> a new commit with two parents that joins the two histories. A **rebase** takes your branch's
> commits and replays them one by one on top of another branch. A **conflict** happens when both
> sides changed the same lines. Git stops, marks the file with `<<<<<<<`, `=======` and `>>>>>>>`,
> and waits for you to choose. You'll see for yourself what each one does to the graph.

Practise in a throwaway clone, so your real repo stays clean:

```bash
# laptop · Git Bash
cd /tmp && rm -rf gitlab && git clone "/c/COmE HeRe/-_-/Engineer Hoite chaile porar dorkar nai/Devops/trackpoint" gitlab && cd gitlab

git switch -c exp/a && sed -i '1s/.*/# TrackPoint (version A)/' README.md && git commit -am "A: title"
git switch main && git switch -c exp/b && sed -i '1s/.*/# TrackPoint (version B)/' README.md && git commit -am "B: title"
git switch main && echo "main moved on" >> OPS_NOTES.md && git commit -am "main: unrelated change"

# 1) merge: conflict, resolve, look at the graph
git switch exp/a && git merge exp/b        # CONFLICT: edit README.md, then:
git add README.md && git commit --no-edit
git log --oneline --graph --all

# 2) rebase: replay exp/b onto the new main, look at the graph again
git switch exp/b && git rebase main
git log --oneline --graph --all
```

Explain in the log how the two graphs differ, why rebase changes commit SHAs, and why you never
rebase a branch other people have already pulled.

---

## Step 12 — Prove you understand (1 h)

Answer the six questions from the PDF in `docs/learning-log.md` under a heading
`## Stage 1 — Prove you understand`. Use **your own words and your own evidence** (outputs from
your experiments). If you can't answer one without searching, the stage isn't finished.

| Question | Your evidence comes from |
|---|---|
| 1. Liveness vs readiness, and why conflating them causes an outage | Step 4.6 |
| 2. Why not run the app as root, even on your own server | Steps 9.3 and 9.5 |
| 3. What happens step by step during `ssh user@host`; where host keys live and what they protect | Step 8; your `~/.ssh/known_hosts`; try `ssh -v tp` and read it |
| 4. What `Restart=on-failure` does that `nohup` doesn't, and its limits | Steps 11.1 and 11.4 |
| 5. How a hotfix reaches production in your branching strategy | Step 6.1 |
| 6. Which legacy risk you'd fix first with only one day, and why | Step 3 |

---

## Step 13 — The golden rule: destroy and rebuild (1–2 h)

"At the end of each stage, destroy what you built and recreate it from the repository."

> **📘 Learn: why destroy and rebuild?** A server set up by hand slowly becomes a **snowflake**:
> unique, and impossible to recreate, because nobody remembers every change made to it. Modern
> teams treat servers as replaceable ("cattle, not pets"). The *instructions* are the real asset,
> and any single server can be thrown away. A runbook you've never followed from scratch is only a
> guess; rebuilding is how you test it. In Stage 5, Terraform and Ansible turn this runbook into
> code.

1. Make sure `docs/runbooks/server-setup.md` contains every command from Steps 8 and 9, in order.
2. Destroy the server: terminate the Oracle instance (tick *Permanently delete the attached boot volume*), then recreate it following your runbook, console clicks included.
3. Start a timer. Follow the runbook **literally**, copy-pasting from it and never from memory.
4. Every time the runbook is wrong or missing something, stop, fix the runbook, and continue.
5. Record the total time and the number of runbook fixes in the progress tracker.

You'll repeat this in Stage 5 with Terraform and Ansible, and compare the times. That comparison
makes a great paragraph for your portfolio.

---

## Final checklist

| Done | Item | Evidence (file / screenshot) |
|---|---|---|
| ☐ | C1 Direct push to `main` rejected; PR blocked by a failing check | `docs/evidence/stage1/` screenshots |
| ☐ | C2 Unit + integration tests green; health endpoints match the contract with the DB and Redis down | PR links, log entry |
| ☐ | C3 root ✗ / password ✗ / key ✓ | `docs/evidence/stage1/ssh-checks.txt` |
| ☐ | C4 API back after reboot without logging in | evidence output |
| ☐ | C5 `docs/risk-register.md` with ≥ 12 risks | file |
| ☐ | C6 `docs/server-inventory.md`, generated plus the hand-written port table | file |
| ☐ | Branching strategy, PR template, CODEOWNERS, `.gitignore` | files |
| ☐ | Makefile with run / test / lint (+ ESLint) | file |
| ☐ | `deploy/systemd/trackpoint-api.service` committed | file |
| ☐ | Five break-it experiments logged | learning log |
| ☐ | Six questions answered | learning log |
| ☐ | Rebuilt from the runbook; time recorded | progress tracker |

Add your row to the progress tracker (Appendix D) in `docs/progress.md`: target date, done date,
hours, hints used (be honest, including the ones in this guide), rebuilt from code?, biggest lesson.

### Suggested PRs (one per branch, squash-merged)

1. `docs/stage1-risk-register`: risk register + learning log
2. `fix/…`: one PR per bug found in Step 4 (tests, health endpoints, …)
3. `chore/makefile-and-lint`: Makefile + ESLint
4. `chore/repo-governance`: branching strategy, `.gitignore`, PR template, CODEOWNERS, `pr-checks.yml`
5. `feat/systemd-api`: unit file, inventory script, runbook, inventory doc, evidence

### Suggested sessions

| Session | Steps | ~Time |
|---|---|---|
| 1 | 0–3: read, find the leak, risk register | 3 h |
| 2 | 4–5: run locally, fix tests and health checks, Makefile | 3–4 h |
| 3 | 6–7: governance files, branch protection | 2 h |
| 4 | 8–10: server, SSH, systemd, inventory | 4 h |
| 5 | 11–13: break it, answer the questions, rebuild | 3–4 h |

When you're stuck, ask me for help and say which level you want: **L1** nudge, **L2** concept,
**L3** pointer or **L4** walkthrough. When Stage 1 is done, ask for the Stage 2 guide.
