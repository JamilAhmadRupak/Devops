# Stage 1 · Step 3: risk register walkthrough (R-03 → R-20)

> **What this is:** a full (L4) walkthrough for the risk register only. For each risk you get
> **Find it** (commands or files to check yourself), **What it means** (why it matters), and
> **Write it** (a model row).
> **Rule:** run every *Find it* step yourself before you copy a row. The *Evidence* column must
> describe what **you** saw. The L/I scores are suggestions, so change any you'd argue differently.
> **Not covered:** the bugs inside the app code (`app/`). Those belong to later stages and stay
> unspoiled. R-20 records only the *symptoms*.
>
> **How each risk works:** **📘 Learn** (the general idea, in a few lines) → **Find it** →
> **What it means** (this risk on this box) → **Write it**. Read the Learn box first if the risk's
> title contains a word you don't know.

All commands: `# laptop · Git Bash`, run inside `Devops/trackpoint/`.

Scoring reminder: L and I from 1 to 5, Score = L × I. 15–25 = fix first · 8–14 = planned · 1–7 = accept/monitor.

> **📘 Learn: words this walkthrough uses**
>
> - **Vulnerability vs exposure.** A vulnerability is a weakness, such as a missing password. Exposure
>   is whether an attacker can actually reach it. Many rows below are a vulnerability *plus* an
>   exposure, and that combination is what makes the score high.
> - **Mitigation:** an action that lowers a risk's likelihood or its impact. Most risks get
>   mitigated rather than eliminated.
> - **Requirement IDs** (BUS-02, SEC-06, CI-08, …) point at the requirement lists in the PDF. Citing
>   one ties a risk to something the business asked for, and that's what gets a fix prioritised.
> - **Desk audit:** reviewing documents, code and history instead of the live system. You can't log
>   in to the legacy box, so this register is a desk audit, and it should say so.

---

## R-03 Unsupported, never-patched OS

> **📘 Learn: OS support and CVEs.** Ubuntu releases an **LTS** (Long-Term Support) version every two
> years: 20.04, 22.04, 24.04. Each one gets 5 years of free security updates ("standard support").
> After that, patches need a paid subscription (Ubuntu Pro / ESM). A **CVE** (Common Vulnerabilities
> and Exposures) is a public ID for a known security hole, for example CVE-2024-6387 in OpenSSH.
> Once a CVE is published, attackers scan the internet for machines that haven't patched it, often
> within days. **unattended-upgrades** is the Ubuntu package that installs security updates
> automatically every day.

**Find it**
1. PDF §4, row "Server": *Ubuntu 20.04 (end of standard support) … never updated.*
2. `OPS_NOTES.md` → "The box".
3. Look up the end-of-standard-support date for 20.04 (ubuntu.com/about/release-cycle).
4. Optional, on any Ubuntu VM: `lsb_release -a` and `apt list --upgradable | wc -l` show the version and how many patches are waiting.

**What it means:** with standard support over, the box gets no free security patches. "Never updated"
means it is also missing everything released *before* support ended. Every public CVE for that kernel,
OpenSSH, nginx and so on is an open door.

**Write it**
`| R-03 | Ubuntu 20.04 past end of standard support, never updated: known public CVEs stay unpatched | PDF §4 "Server"; OPS_NOTES "The box" | 4 | 5 | 20 | 1 | New server on a supported LTS (24.04), unattended-upgrades for security patches |`

---

## R-04 Nothing restarts the app (tmux + nohup)

> **📘 Learn: tmux, nohup and supervisors.** **tmux** is a terminal multiplexer: it keeps a terminal
> session alive on the server after you disconnect, so the programs started inside it keep running.
> **nohup** does a similar job for a single command, by ignoring the "hang-up" signal sent when your
> SSH session closes. Both are tools for *people*. A **process supervisor** is a program whose job is
> running services: it starts them at boot, watches them, and collects their logs. On Linux that's
> systemd (Stage 1, Step 9). With containers, it's Docker's restart policy or Kubernetes.

**Find it**
1. PDF §4, row "Running the app".
2. `OPS_NOTES.md`: find the bold line *"After a reboot nothing starts by itself."*
3. `scripts/deploy.sh` lines 20–29: `tmux kill-session` / `tmux new-session`.

**What it means:** no process supervisor. If the API crashes at 03:00 it stays down until a human
logs in. A kernel-update reboot means a full outage. `tmux kill-session` also kills processes
mid-work with no graceful shutdown.

**Write it**
`| R-04 | API/worker/web run by hand in tmux; no supervisor, so a crash or reboot = outage until someone logs in | PDF §4 "Running the app"; OPS_NOTES "After a reboot nothing starts"; deploy.sh L20-29 | 5 | 4 | 20 | 1, 2 | systemd units with Restart=on-failure (Stage 1), then containers with restart policy + health checks (Stage 2) |`

---

## R-05 Redis open to the internet without a password

> **📘 Learn: Redis and bind addresses.** **Redis** is an in-memory key-value store. It's very fast,
> and it's often used as a cache, a session store or, here, a job queue. Unless you configure a
> password or ACL users, it trusts every client that can connect, and every client can run admin
> commands such as `FLUSHALL` (delete everything) and `CONFIG SET`. It was designed to sit on a
> private network behind the app. **Binding** to `0.0.0.0` means listening on *every* network
> interface, the public one included. Binding to `127.0.0.1` means only this machine can connect.

**Find it**
1. PDF §4, row "Redis": *bound to all interfaces, no password.*
2. `OPS_NOTES.md` → "Database / Redis": the reason given (a dev's laptop worker).
3. Combine with R-06 (firewall off): is anything in between Redis and the internet?

**What it means:** anyone who finds port 6379 can read or flush the job queue, inject fake jobs, or
use well-known Redis tricks (`CONFIG SET dir …`) to write files on the server. Exposed Redis is one
of the most commonly scanned and hijacked services.

**Write it**
`| R-05 | Redis bound to 0.0.0.0 with no password and no firewall: anyone can read/flush/inject queue jobs or abuse CONFIG to write files | PDF §4 "Redis"; OPS_NOTES "Database / Redis" | 5 | 4 | 20 | 2, 3, 7 | Private network only, requirepass/ACL user, never publish port 6379 |`

---

## R-06 Firewall disabled

> **📘 Learn: firewalls, attack surface, defence in depth.** A **host firewall** (on Ubuntu,
> **ufw**: a friendly front-end for iptables/nftables) decides which incoming connections are
> allowed. The normal policy is **default deny**: block everything, then allow only the ports you
> need. Your **attack surface** is everything an attacker can reach and try. **Defence in depth**
> means several independent layers (firewall, bind address, password, least privilege), so that a
> single mistake isn't enough to get in. Disabling the firewall removes one layer from under every
> service at once.

**Find it**
1. PDF §4, row "Firewall".
2. `OPS_NOTES.md`: *"ufw was blocking pgAdmin so I disabled it."* Note the reason: convenience beat security.
3. List what that exposes: 22 (root + password), 5432 (R-02), 6379 (R-05), 3000 and 8080 (app ports
   that should sit behind nginx).

**What it means:** this one multiplies other risks. Every service listening on 0.0.0.0 is reachable
from the internet. With a firewall, R-02 and R-05 would be only "badly configured". Without it they
are "exploitable".

**Write it**
`| R-06 | Firewall disabled, so every listening port (22, 3000, 5432, 6379, 8080) is internet-reachable; amplifies R-01, R-02, R-05 | PDF §4 "Firewall"; OPS_NOTES "ufw was blocking pgAdmin so I disabled it" | 5 | 4 | 20 | 1, 3 | ufw default-deny inbound; allow 22 (key-only), 80, 443 only; DB access via SSH tunnel |`

---

## R-07 Production `.env` leaked in Git history

> **📘 Learn: secrets and rotation.** A **secret** is any value that grants access: passwords, API
> tokens, session-signing keys, private keys. **Rotation** means creating a new secret, switching
> everything that uses it to the new one, then revoking the old one. A leaked secret stays dangerous
> until it is rotated, however well the leak is hidden afterwards. **git filter-repo** rewrites
> commits to remove a file from history. That changes the SHA of every commit from that point on,
> so everyone has to re-clone. In the commands below, `cut -d= -f1` splits each line at `=` and keeps
> only the first part, the variable name, so no value ever reaches your screen.

**Find it**
```bash
git log --oneline --all -- .env                   # which commits touched .env
git show --stat <sha-of-"server-setup">           # who added it, when, with what
git show <sha>:.env | cut -d= -f1                 # VARIABLE NAMES ONLY, no values
```
Check that "deleting" it did nothing for the history: `git show <sha>:.env` still works on the
current clone, so the file is still there.

**What it means:** the delete commit only removed the file from *new* snapshots. The old commit, and
every clone and fork, and GitHub, still has it. Anything in that file is compromised, *including
values that look harmless*. The fix is **rotate first** (a new password makes the leaked one
useless), *then* rewrite history.

**Write it** (fill in the names you saw)
`| R-07 | Production .env committed in "server setup" (Marek, 2026-01-13), "deleted" 2026-02-16 but still in history and on GitHub. Exposed: DATABASE_URL, ADMIN_TOKEN, SESSION_SECRET, SMTP_USER, SMTP_PASSWORD, … (values redacted) | git log -- .env; git show <sha>:.env | 5 | 5 | 25 | 7 (rotate all now-ish, rewrite history) | Rotate every credential, git filter-repo, force-push + re-clone, add gitleaks pre-commit + CI scan |`

---

## R-08 Real secrets still in tracked files

> **📘 Learn: config vs secrets.** A port number is *config*; a password is a *secret*. Config can
> be committed. Secrets must reach the app from outside the repo at runtime: an environment
> variable, a file only the service can read, or a secret store (HashiCorp Vault, a cloud secrets
> manager, Docker/Kubernetes secrets). An `.env.example` documents *which* settings exist, so it
> holds placeholders like `change-me`. Two shell tricks below: `cmp -s` compares two inputs silently
> and reports only through its exit code, and `<( … )` (*process substitution*) lets a command's
> output be used as if it were a file. Together they compare two values without printing either one.

**Find it**
```bash
grep -rn -E "PASSWORD|TOKEN|SECRET|postgres:postgres" --exclude-dir=node_modules --exclude-dir=.git .
```
Then check whether the "example" token is a real one, **without printing it**:
```bash
cmp -s <(git show <sha>:.env | grep '^ADMIN_TOKEN=') <(grep '^ADMIN_TOKEN=' .env.example) \
  && echo "SAME as leaked prod value" || echo "different"
```
Also look at `scripts/backup.sh` line 5 and `docker-compose.yml` (passwords, token).

**What it means:** rotating the leaked `.env` is not enough while the same values sit in files that
are committed today. Example files should hold obvious placeholders (`change-me`), never a value
that ever worked anywhere.

**Write it**
`| R-08 | Live/legacy credentials hard-coded in tracked files: ADMIN_TOKEN in .env.example and docker-compose.yml (same as leaked prod value), DB superuser password in backup.sh and compose | grep results; cmp check; backup.sh L5 | 5 | 4 | 20 | 2, 7 | Placeholders in .env.example, secrets only via env/secret store, rotate, secret-scan in CI |`

---

## R-09 Customer personal data committed to the repo

> **📘 Learn: personal data.** **PII** (personally identifiable information) is any data that
> identifies a person: name, address, phone number, e-mail. Data-protection laws (the EU's GDPR,
> and similar laws in many countries) require it to be kept only where it's needed, protected,
> deleted when no longer needed, and some leaks must be reported. The brief's BUS-05 requirement
> covers this. When developers need realistic data to reproduce a bug, they use **synthetic**
> (generated, fake) or **anonymised** data, never a copy of production.

**Find it**
1. `git ls-files db/`: why does an app repo have a `db/debug/` folder?
2. Open the file and read only the header comment. Where did the data come from?
3. `git log --oneline -- db/` for who and when.
4. Note *what kinds* of data it contains (names, addresses, phones, e-mails). **Do not copy any of it
   into the register.**
5. Try `gitleaks git -v`. Does it flag this file? (It won't. Secret scanners look for keys, not
   people. Only a human audit finds this.)

**What it means:** this is a **data-protection incident**, not just bad hygiene. Real customers' data
is on every developer laptop and on GitHub, with no access control and no deletion date (BUS-05).

**Write it**
`| R-09 | Production customer PII (names, addresses, phones, e-mails) committed in db/debug/customer-issue-4471.sql "to repro a ticket" | git ls-files db/; header comment; commit "data to repro #4471" | 5 | 5 | 25 | 1 (remove), 7 (purge history) | Delete file now, purge from history with the .env, treat as a data-protection incident, use synthetic/anonymised data for repros |`

---

## R-10 TLS certificate renewal unverified

> **📘 Learn: TLS certificates.** **TLS** is the encryption behind `https://`. The server proves its
> identity with a **certificate** signed by a **certificate authority (CA)** that browsers trust.
> Every certificate has an expiry date. After it, browsers show a full-page warning and most
> customers leave. **Let's Encrypt** is a free CA. **certbot** is its client: it proves you control
> the domain (the *ACME challenge*, for example by serving a file on port 80), then downloads the
> certificate. Let's Encrypt certificates are short-lived on purpose, so renewal must be automatic,
> and something must warn you when it fails. `openssl s_client` connects the way a browser does, and
> `openssl x509 -noout -dates` prints the certificate's validity dates.

**Find it**
1. PDF §4, row "TLS": *obtained manually 11 months ago.*
2. `OPS_NOTES.md` → "HTTPS": note the **date** and the **command** (`certbot certonly --standalone`).
3. Look up how long a Let's Encrypt certificate is valid, then do the maths from that date.
4. Think about it: `--standalone` starts its own web server on port 80. What is already on port 80
   when the renewal timer runs?
5. Practise the check you'd run on the real box (try it on any site):
   ```bash
   echo | openssl s_client -connect github.com:443 -servername github.com 2>/dev/null | openssl x509 -noout -dates
   ```

**What it means:** LE certificates last 90 days. A manual cert from 11 months ago has either renewed
three-plus times without anyone knowing, or the site has been running on an expired cert. The
renewal method also fights nginx for port 80. Either way, nobody would know before customers
see a browser warning, since there's no monitoring (R-15).

**Write it**
`| R-10 | TLS cert issued by hand (certbot --standalone, 2025-10-28); 90-day validity, renewal never verified and conflicts with nginx on :80. Expiry = site down for customers with no warning | PDF §4 "TLS"; OPS_NOTES "HTTPS" | 4 | 4 | 16 | 3 (6 for expiry alert) | Automatic TLS in the reverse proxy (webroot/HTTP-01 or Caddy), expiry alert |`

---

## R-11 Deploy script: root, password, MITM, working-copy zip

> **📘 Learn: what a good deploy looks like.**
>
> - A **deploy** gets one specific version of the code running in production. A good deploy is
>   **repeatable** (same input, same result), **traceable** (you can say exactly which commit is
>   live) and **reversible** (you can roll back).
> - **Build once, deploy many:** CI builds one **artifact** (a tarball or a container image) from a
>   commit and tests it, and that exact artifact goes to every environment. The opposite is building
>   on the server, where the result depends on whatever gets downloaded that day.
> - **Man-in-the-middle (MITM):** someone on the network path between you and the server pretends to
>   be the server. SSH has a built-in defence against it. Find out which line turns it off.

**Find it**: read `scripts/deploy.sh` slowly and ask about each line:
- L3, L5: who does it log in as, and how?
- L13, L16: look up what `StrictHostKeyChecking=no` disables (link to Stage 1 "Prove you understand" Q3).
- L10: it zips `.`, your **working copy**. What is *not* excluded? Is `app/api/.env` on your laptop excluded?
- L19, L23: `npm install` on the production server. Which versions get installed?
- Is there anything that records *which* version is now running? Any health check? Any rollback?

**What it means:** each deploy is root over a password with host-key checking off (anyone in the
network path can impersonate the server and grab the root password). It ships whatever is on the
developer's disk, uncommitted changes and their local `.env` included, which overwrites prod config.
And nobody can answer "what version is running?" (PDF §4 "Deploys").

**Write it**
`| R-11 | Deploys = zip of the dev's working copy (incl. uncommitted code and local .env) via root+password with StrictHostKeyChecking=no; npm install on prod; no version record, health check or rollback | PDF §4 "Deploys"; deploy.sh L3,5,10,13,16,19 | 5 | 4 | 20 | 4 | Build once in CI, versioned image/artifact, deploy user with key, health-checked deploy + rollback |`

---

## R-12 CI auto-deploys main as root, even when tests fail

> **📘 Learn: CI/CD and the supply chain.**
>
> - **CI** (continuous integration) runs checks on every change. **CD** (continuous delivery or
>   deployment) ships the changes that *pass*. That word is the whole point: a pipeline that deploys
>   failing code is just an automated way to break production.
> - Your **software supply chain** is all the code you didn't write that runs in your build: npm
>   packages, base images, GitHub Actions. If any of it gets compromised, so do you. **Pinning**
>   fixes each dependency to an exact version that can't change. A branch reference like `@main`
>   moves every time its owner pushes.
> - **GitHub secrets** are encrypted values a workflow can read. Whoever can change a workflow that
>   has access can usually get at them, so be careful what you store there. A root password is the
>   worst possible choice.

**Find it**: read `.github/workflows/ci.yml`:
- L3: when does it run? Is there any PR gate?
- L19: what does `continue-on-error: true` do to a failing test?
- L9, L10, L27: `@main` / `@master`. Whose code runs in your pipeline, and can it change tomorrow?
- L30–31: what credential is stored in GitHub secrets?
- L33–34: what happens on prod?

**What it means:** a red test still deploys (the job "needs" `test`, but `test` always reports
success). Third-party actions are pulled from a moving branch, so if their repo is compromised, so
is your prod root password. Prod builds itself with `git pull` + `npm install`.

**Write it**
`| R-12 | CI deploys every push to main as root with a password; tests are continue-on-error so failures still deploy; actions pinned to @main/@master (supply chain) | ci.yml L3,9-10,19,27,30-34; README "test step is set to continue on error" | 4 | 5 | 20 | 4 | PR-gated CI, failing tests block, actions pinned by SHA, deploy key/OIDC, environment approval, smoke + rollback |`

---

## R-13 Logs never rotated, disk at 87%

> **📘 Learn: logs and disks.**
>
> - **Log rotation:** on a schedule, or at a size limit, rename the current log file, compress the
>   old ones and delete the oldest (`logrotate` does this on Linux). Without it, a log file grows
>   until the disk is full.
> - The modern alternative is for the app to write logs to **stdout**, and let the platform
>   (journald, Docker, Kubernetes) store them, cap their size and ship them elsewhere. Twelve-Factor
>   calls logs "event streams".
> - **WAL** (write-ahead log): PostgreSQL records every change in a log *before* applying it, so it
>   can recover after a crash. No disk space means no WAL, and no WAL means no writes.
> - Two readings of disk usage (80% in January, 87% now) give you a growth rate, and from that you
>   can predict the day it hits 100%. That's **capacity planning** in its simplest form.

**Find it**
1. PDF §4, row "Logs": *never rotated. Disk is at 87%.*
2. `OPS_NOTES.md` → "Logs" (the file path, and "the tmux scrollback has the rest") and the advice
   *"if it fills up, delete old stuff"*.
3. `README.md`: last bullet. Compare 80% (Jan) with 87% (now). How fast is it growing?
4. Link it to the "Break it on purpose" experiment *Fill a disk to 100%*: what broke first for you?

**What it means:** logs and backups (R-14) share one disk and both only grow. At 100%, PostgreSQL
can't write its WAL and stops accepting writes, and logging fails, so you lose the evidence too. And
logs in a tmux scrollback are lost on every restart.

**Write it**
`| R-13 | Logs written to a file in the app dir, never rotated; backups on the same disk; disk 80% (Jan) → 87% (now); at 100% Postgres stops writing | PDF §4 "Logs"; OPS_NOTES "Logs"; README "disk is at 87%" | 4 | 4 | 16 | 1, 6 | Log to stdout → journald/Docker with size limits, central logs (Stage 6), disk-usage alert |`

---

## R-14 Backups: same disk, never restored, "success" is a lie

> **📘 Learn: backups.**
>
> - **The 3-2-1 rule:** keep 3 copies of your data, on 2 different kinds of storage, with 1 of them
>   off-site. A backup on the same disk dies with the disk.
> - **RPO** (Recovery Point Objective): how much data you can afford to lose, e.g. "the last 24
>   hours". **RTO** (Recovery Time Objective): how long a restore may take. Both come from the
>   business, and only a real restore test proves you can meet them.
> - **Bash strict mode:** `set -e` stops a script at the first failing command, `set -u` treats an
>   unset variable as an error, and `set -o pipefail` changes how a pipeline (`a | b`) reports
>   failure. The experiment in *Find it* shows you why that last one matters.

**Find it**
1. PDF §4, row "Backups". OPS_NOTES → "Backups": where do dumps go? Same disk as the DB?
2. `scripts/backup.sh`: note L5 (password) and L11 (the pipeline).
3. **Experiment.** Does a failing `pg_dump` make the script fail?
   ```bash
   false | gzip > /tmp/t.gz; echo "exit=$?"; ls -l /tmp/t.gz
   ```
   `false` stands in for a failed `pg_dump`. Look at the exit code and the file size. Then reread
   L13. What does the nightly "Backup successful" actually prove?
4. Ask the questions a restore needs: encrypted? Off-site? Retention? Ever restored?

**What it means:** a pipeline's exit code is the **last** command's (`gzip`), so a dead `pg_dump`
still produces a tiny valid `.gz` and a "successful" log line. The backup sits on the disk it's meant
to protect, unencrypted, full of customer data. A backup that has never been restored is a hope,
not a backup.

**Write it**
`| R-14 | Backups on the same disk as the DB, unencrypted PII, no off-site copy, never restored; backup.sh reports "successful" even when pg_dump fails (pipeline exit code) | PDF §4 "Backups"; backup.sh L5,L11,L13; my experiment: exit=0, ~20-byte file | 4 | 5 | 20 | 8 | set -euo pipefail, verify dump size, encrypted off-site copy, retention, scheduled restore test, freshness alert |`

---

## R-15 No monitoring

> **📘 Learn: monitoring.**
>
> - **Monitoring** collects measurements: is it up, how fast is it, how many errors, how full is the
>   disk? **Alerting** wakes a human when a measurement crosses a line. You need both.
> - **Black-box** monitoring tests from outside, like a customer (an uptime check). **White-box**
>   monitoring reads the app's own numbers (its `/metrics` endpoint).
> - **MTTD** is mean time to *detect* a problem, and **MTTR** is mean time to *recover*. You can't
>   start recovering before you've detected.
> - **Availability maths:** an average month is about 43,830 minutes, and 0.5% of that is about 219
>   minutes, or 3 h 39 min. That's the downtime 99.5% allows (BUS-02), also called the **error
>   budget**.

**Find it**: PDF §4, row "Monitoring": *Outages are reported by customers.* Then reread the
README symptoms: the team only knows about them *because someone noticed*.

**What it means:** your mean time to *detect* equals however long a customer takes to complain.
BUS-02 asks for ≤ 3 h 39 min of downtime a month, and you can't measure or defend that without
monitoring.

**Write it**
`| R-15 | No monitoring or alerting; outages detected by customers; availability (BUS-02 99.5%) unmeasurable | PDF §4 "Monitoring"; README symptoms list | 5 | 4 | 20 | 6 | Uptime check on /healthz + /readyz, metrics + alerts (disk, cert expiry, backup freshness, error rate) |`

---

## R-16 Shared root password and bus factor 1

> **📘 Learn: shared accounts.**
>
> - **Bus factor:** how many people would have to disappear (hit by a bus, or just on holiday)
>   before the work stops. 1 is the worst possible number.
> - **Audit trail:** a record of who did what, and when. It needs personal accounts. On a shared
>   account, every action was done by "root".
> - **Revocation:** removing one person's access without disturbing anyone else. With personal SSH
>   keys you delete one line from `authorized_keys`. With a shared password you have to change it
>   and tell everyone else the new one.
> - When a shared credential really must exist, it belongs in a **password manager** (Bitwarden,
>   1Password, KeePassXC), with access granted per person.

**Find it**
1. PDF §4, row "Documentation": *a sticky note with the root password.*
2. `OPS_NOTES.md`: who has the password, and where else is it?
3. `grep -rn -i "tomasz" README.md scripts/`: how many processes depend on one person?
4. With one shared root account, can you tell *who* did something on the server?

**What it means:** a shared password means no audit trail and no way to remove one person's access.
A physical note on a monitor means anyone in the office (or on a video call) has root. If Tomasz is
on holiday, nobody can deploy, and the only real documentation left with the contractor.

**Write it**
`| R-16 | One shared root password (on a sticky note, known by Tomasz); no personal accounts → no audit trail, can't revoke one person; bus factor 1 for deploys | PDF §4 "Documentation"; OPS_NOTES "Log in as root"; deploy.sh L3; README "ask Tomasz" | 4 | 5 | 20 | 1, 5 | Personal sudo users with SSH keys, no root login, password manager, runbooks + IaC so the box is reproducible |`

---

## R-17 Smoke test can never fail

> **📘 Learn: smoke tests.** A **smoke test** is a quick check run right after a deploy ("switch it
> on and see whether smoke comes out"). It doesn't test everything, only that the key paths answer.
> If it fails, the deploy is rolled back. A check is only useful if it can **fail**, so test your
> tests: point them at something broken and confirm they go red. That's exactly what *Find it* does.

**Find it**: point it at a port where nothing is listening:
```bash
bash scripts/smoke.sh http://localhost:9; echo "exit=$?"
```
What did it print? What exit code? Then read the script: what does `curl -s` do on an HTTP error, and
which command runs last?

**What it means:** a check that always passes is worse than no check, because it gives false
confidence ("pipeline green, site down"). It also checks only `/healthz` (liveness), not `/readyz`.

**Write it**
`| R-17 | smoke.sh always exits 0 (curl without -f, last command is echo); "passed" even with nothing listening; checks liveness only | my run: "Smoke test passed", exit=0 against a dead port; smoke.sh L8-11 | 4 | 3 | 12 | 4 | set -e, curl -fsS, check /readyz + a real API call, non-zero exit on any failure |`

---

## R-18 The half-finished `docker-compose.yml`

> **📘 Learn: Docker Compose in brief** (Stage 2 goes deep).
>
> - **Docker Compose** describes several containers ("services") and how they connect, in one YAML
>   file. `docker compose up` starts them all.
> - An **image tag** labels one version of an image, e.g. `postgres:17.2`. `latest` is just a tag
>   that its publisher moves to each new release, so the same file gives different results on
>   different days.
> - A container's own filesystem is **disposable**: remove the container and everything written
>   inside it is gone. A **volume** is storage that lives outside the container and survives it.
>   Databases need one.
> - `ports: "5432:5432"` **publishes** a container port on the host machine, which makes it
>   reachable from outside. Containers on the same Compose network can already talk to each other
>   without it.
> - `depends_on` controls the order in which services start. *Find it* asks what exactly it waits
>   for.

This row is your bridge to **Stage 2**, which replaces it entirely.

**Find it**: read `docker-compose.yml` and list:
- every `image:` tag (L6, L14, L32, L45). What does `latest` give you next month?
- every `ports:` block. Which ones are databases?
- the `postgres` service: is there a `volumes:` entry? Now reread the README's "Docker" section
  (*"Postgres data sometimes disappears"*).
- hard-coded passwords and token (L8, L23, L25, L38).
- any `healthcheck:`? Does `depends_on` wait for *ready* or just *started*?

**What it means:** unpinned images make builds unrepeatable. Published DB ports put Postgres and
Redis on the host's public interface (and Docker's port publishing goes *around* ufw). With no named
volume, the "Docker Desktop bug" is actually expected Docker behaviour.

**Write it**
`| R-18 | Legacy compose: :latest images, Postgres/Redis ports published on the host, hard-coded credentials, no Postgres volume (explains "data sometimes disappears"), no health checks | docker-compose.yml L6-16,23,25; README "Docker" | 4 | 4 | 16 | 2 | Rewrite in Stage 2: pinned images, named volumes, internal networks, no DB ports, healthchecks + depends_on: service_healthy |`

---

## R-19 Single small server = single point of failure

> **📘 Learn: single points of failure.**
>
> - A **single point of failure (SPOF)** is one component whose failure stops the whole system.
>   **Redundancy** (having two of it) removes the SPOF, at a cost.
> - **Vertical scaling** means a bigger machine. **Horizontal scaling** means more machines behind a
>   load balancer. Horizontal scaling also gives you redundancy, but the app must be *stateless*,
>   with its data kept somewhere shared (database, Redis).
> - **Zero-downtime deploy:** start the new version, check it's healthy, move the traffic over, then
>   stop the old version (*rolling* or *blue-green* deploys). With one instance and a restart, every
>   deploy is a short outage.
> - Not every SPOF is worth removing: the BUS-04 budget limits what you can buy. The register
>   records the risk, and the business decides what to accept.

**Find it**: PDF §4 "Server" (1 VPS, 2 vCPU / 2 GB RAM, running Postgres + Redis + 3 Node
processes + nginx) vs. BUS-02 (99.5%) and BUS-03 (deploys without customer impact).

**What it means:** one box, one disk, everything co-located. A disk failure, a provider outage, or one
process eating the RAM takes everything down. Every deploy restarts the only copy.

**Write it**
`| R-19 | Everything on one 2 GB VPS: any host/disk failure or memory pressure = total outage; deploys restart the only instance (conflicts with BUS-02, BUS-03) | PDF §4 "Server"; BUS-02, BUS-03 | 3 | 5 | 15 | 8, 9 | Resource limits, zero-downtime deploys, tested restore to a new host, scale-out plan within BUS-04 budget |`

---

## R-20 Known symptoms with unknown cause

> **📘 Learn: symptoms vs root causes.** A **symptom** is what you observe ("the API restarts by
> itself"). The **root cause** is why it happens. Writing a guessed cause down as fact sends the team
> off to fix the wrong thing. Good practice: record the observation, when it happens and the
> evidence, then investigate with logs and metrics. Many teams write up incidents as **blameless
> postmortems** (what happened, what we saw, why, what we'll change), which focus on systems, not on
> people.

**Find it**: the README's "For the new infra person" bullets: the API restarts on its own, label jobs
go missing after deploys, the reports page is slow, a flaky test.

**What it means:** you don't know the root causes yet, and that's fine (you'll find them in later
stages). What a register *should* record is that unexplained failures exist in production right now.
Don't guess causes in the register. Write what is observed.

**Write it**
`| R-20 | Unexplained production symptoms: API restarts by itself, label jobs lost after deploys, slow reports page, flaky test silenced in CI | README "For the new infra person" | 4 | 4 | 16 | 1 (observe), 2, 6, 8 | Supervisor + logs to capture crashes, investigate each with evidence, track as separate issues |`

---

## Finish the document

1. **Sort the table by Score, highest first.** That makes it a priority list the board can read.
2. Add a **Scope** line at the top: *"Desk audit. The legacy box (203.0.113.47) was not
   accessible; findings are based on the handover documents (brief §4, OPS_NOTES.md, README.md),
   the repository and its Git history."* An audit that states its limits is more credible.
   Then add a short **Summary** line above the table, e.g. *"20 risks: N critical (15–25), N planned, N
   accepted. The box should be treated as compromised."* (After R-07 and R-09, it should be.)
3. Add an **"If I had one day"** paragraph (also Stage 1 Q6). Write it yourself. Pick 3 using:
   - **highest score** first,
   - **cheapest to fix** (minutes, not weeks),
   - **stops active harm** (exposure happening right now) over future risk.
   For each, say *what* you'd do in that day and *why it beats the others*.
4. Add a `Status` column (`Open`) now. You'll flip rows to `Mitigated (Stage N)` as you go. In
   Stage 2, R-04 (partly) and R-18 move.

Then commit through a PR (Stage 1 guide, Step 3).

> Once you've written it, ask me to **check your register**. I'll compare it with the answer key and
> tell you how many legacy-box risks you've covered. I won't say which ones you missed unless you
> ask.
