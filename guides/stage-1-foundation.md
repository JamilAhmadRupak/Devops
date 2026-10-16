# Stage 1 — Foundation: step-by-step guide

> **Who this is for:** you, working through Operation TrackPoint with the `trackpoint` starter repo.
> **Time:** 10–14 hours. Spread it over 4–5 sessions (suggested split at the end).
> **Spoiler policy:** this guide shows you *how* to do each step. It does **not** tell you what is
> wrong with the starter app. Where you will hit a planted problem, you get a hint ladder
> (L1 → L3) inside a collapsed `Hints` block. Open one level at a time, and only after you
> have spent 20+ minutes on the problem yourself.

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
├── trackpoint-challenge/   ← the brief + these guides (public repo "Devops")
├── trackpoint/             ← YOUR PROJECT REPO. All Stage 1 deliverables go here.
└── trackpoint-mentor-sealed/
```

Everything you produce in Stage 1 (docs, Makefile, `.github/` files, systemd units, scripts) is
committed to **`trackpoint`**, because that becomes your portfolio repo (Appendix A layout).
Create `trackpoint/docs/learning-log.md` there and use it from now on. The empty one in
`trackpoint-challenge/docs/` can be deleted.

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

---

## Step 1 — Work like a team from the first commit (30 min)

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

### 4.1 Create a dedicated database user (not the superuser)

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

## Step 8 — A server you can trust (2–3 h) → checkpoint C3

### 8.1 Choose where the server runs

| Option | Cost | Pros | Cons |
|---|---|---|---|
| **Budget VPS** (Hetzner, DigitalOcean, Vultr…), Ubuntu 24.04, 2 vCPU / 2–4 GB | ~US$4–8/month | Real public IP; Stage 3 (domain + TLS) needs one anyway | Costs money; mistakes are exposed to the internet |
| **Local VM** (VirtualBox + Ubuntu Server 24.04 ISO) | free | Snapshots let you undo; safe to break | Needs ~4 GB RAM free; no public IP until Stage 3 |
| **WSL Ubuntu** (you have it) | free | Already installed | Not a real server: it doesn't boot like one, has no real SSH exposure, and "reboot" means `wsl --shutdown`. Fine for practising commands, **weak evidence** for the checkpoints. |

Recommendation: a **local VirtualBox VM** for Stages 1–2 (snapshot before every risky step), then a
VPS from Stage 3. If you can afford a VPS now, using it from day 1 is also fine.

For a VirtualBox VM: Windows Home has no Hyper-V, so install VirtualBox. Create a 2 vCPU / 2 GB /
20 GB VM, use a **NAT** adapter, and add a port-forwarding rule: host `127.0.0.1:2222` → guest
`22`. During the Ubuntu install, tick **Install OpenSSH server**.

### 8.2 Create your SSH key (laptop)

```powershell
# laptop · PowerShell
ssh-keygen -t ed25519 -C "rupak@laptop"     # accept the path, SET A PASSPHRASE
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

Create `C:\Users\HP\.ssh\config` so you can type `ssh tp`:

```
Host tp
    HostName 127.0.0.1        # VPS: its public IP
    Port 2222                 # VPS: 22
    User rupak
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

### 8.3 First login: create your personal admin user

```bash
# server (first login: as root on a VPS, or as the installer user on a VM)
sudo apt update && sudo apt full-upgrade -y
sudo adduser rupak                 # set a password: used for sudo only, never for SSH
sudo usermod -aG sudo rupak
```

Copy your public key to the new user:

```powershell
# laptop · PowerShell   (VM example; for a VPS use: ssh root@<IP> -p 22 ...)
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 rupak@127.0.0.1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Now `ssh tp` should log you in **with the key**: it asks for the key passphrase, not the account
password. **Do not continue until this works.**

### 8.4 Harden SSH, without locking yourself out

**Golden rule:** keep one SSH session open the whole time, and test changes in a **second**
session.

```bash
# server
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
the `sshd_config.d/*.conf` files in alphabetical order. Some cloud images ship
`50-cloud-init.conf` with `PasswordAuthentication yes`, and `10-` beats it. The `sshd -T` output
shows the *effective* values, so check that it's what you expect.

### 8.5 Hostname, timezone, time sync

```bash
# server
sudo hostnamectl set-hostname tp-stage1
sudo sed -i 's/^127\.0\.1\.1.*/127.0.1.1 tp-stage1/' /etc/hosts
sudo timedatectl set-timezone UTC
timedatectl                       # look for: Time zone UTC, "System clock synchronized: yes", "NTP service: active"
timedatectl timesync-status       # which NTP server, and how far off the clock is
```

Why UTC on servers? Write the answer in your log, connecting it to what you found in Step 4.5.

### 8.6 Prove C3 from a second terminal

```powershell
# laptop · PowerShell
ssh -p 2222 root@127.0.0.1                                                          # expect: Permission denied (publickey)
ssh -p 2222 -o PubkeyAuthentication=no -o PreferredAuthentications=password rupak@127.0.0.1   # expect: Permission denied (publickey)
ssh tp "whoami; hostname"                                                           # expect: rupak / tp-stage1
```

Save the three outputs to `docs/evidence/stage1/ssh-checks.txt`. Then read the server's auth log
to see the rejected attempts from the server's side:

```bash
# server
sudo journalctl -u ssh --since "10 min ago" --no-pager | tail -20
```

> Brute-force protection (fail2ban) and the host firewall are SEC-01/SEC-02, scheduled for
> Stage 7. On a **VPS**, consider at least restricting port 22 in the provider's firewall now.

### 8.7 Write it down as you go (the "everything is code" rule)

Copy every server command you run into `docs/runbooks/server-setup.md` in order, with a short
reason for each. In Step 13 you'll rebuild the server by following that file literally. Ansible
replaces it in Stage 5.

---

## Step 9 — Run the API under systemd (2–3 h) → checkpoint C4

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
| 22   | sshd    | 0.0.0.0  | anywhere (VM: only via NAT forward) | admin access |
| ...  |         |          |                |               |
```

Every row must be explainable. "I don't know what this is" means investigate it
(`systemctl status <unit>`, `man <program>`). Pay particular attention to **0.0.0.0 versus
127.0.0.1**: that one column decides what the internet can reach.

---

## Step 11 — Break it on purpose (2 h)

For each experiment, add a learning-log entry: **expected → actual → why**. Snapshot the VM first.

### 11.1 Signals and systemd

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

Don't fill your root disk (unless it's a throwaway VM snapshot). Use a small test partition:

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

Revert whatever you re-pointed, too. **Optional:** on a throwaway snapshot only, fill `/` with
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

1. Make sure `docs/runbooks/server-setup.md` contains every command from Steps 8 and 9, in order.
2. Destroy the server: delete the VM (or rebuild the VPS from a fresh image).
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
