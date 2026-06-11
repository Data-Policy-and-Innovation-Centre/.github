# DPIC Analytics Toolchain — Analyst Guide

**Audience:** Analysts and RAs joining a DPIC project  
**Last updated:** June 2026

---

## Table of contents

1. [What each tool does](#1-what-each-tool-does)
2. [One-time machine setup](#2-one-time-machine-setup)
3. [AWS credentials](#3-aws-credentials)
4. [Box authentication](#4-box-authentication)
5. [GitHub SSH access](#5-github-ssh-access)
6. [Joining a project for the first time](#6-joining-a-project-for-the-first-time)
7. [Daily workflow](#7-daily-workflow)
8. [Data workflow in detail](#8-data-workflow-in-detail)
9. [Common errors and fixes](#9-common-errors-and-fixes)
10. [Rules you must not break](#10-rules-you-must-not-break)

---

## 1. What each tool does

Before you run anything, it helps to understand what each piece does and why it
exists. The whole stack has exactly one job: make it possible to reproduce any
result, on any machine, at any point in the future.

### git

**What it is:** Version control for code. Every change to a script or notebook
is tracked as a commit with a message, author, and timestamp.

**Why we use it:** It is the backbone of everything else. DVC pointer files,
pipeline definitions, and environment specs are all committed to git. A single
`git checkout <old-commit>` combined with `dvc pull` can recreate the exact
state of a project — code, data, and outputs — as it was on any historical
date.

**What you do with it:** You mostly do not run git commands directly. The
Makefile wraps the git operations you need. If you see a message like
`Uncommitted changes. Commit or stash first.`, contact the project lead.

---

### Box

**What it is:** The organization's primary shared file store. Stakeholders,
the ED, and university counterparts upload raw data here. Exhibits (figures,
tables) are delivered back here for review.

**Why we use it:** It is what the rest of the organization already uses. It
is not a version-controlled system — there is no history, no conflict
detection, and no content-addressing. We treat it as an inbound/outbound
transfer layer, not a source of truth.

**What you do with it:** You never interact with Box directly through a
browser for data work. You use `make ingest` to copy a file from Box into
your local `data/raw/`, and `make deliver` to copy exhibits back to Box.

---

### AWS S3

**What it is:** Amazon's cloud object storage. DPIC uses a private S3 bucket
(`dpic-dvc-cache`, in the `ap-south-1` Mumbai region) as the canonical,
versioned store for all approved data files.

**Why we use it:** Box has no versioning. Once a file is overwritten in Box
it is gone. S3, as used through DVC, stores every version of every dataset
forever, identified by a content hash. Any historical dataset can be pulled
back at any time.

**What you do with it:** You never interact with S3 directly. DVC handles
all reads and writes through `make push`, `make pull`, and `make run`.

**Credentials:** You need an AWS access key and secret to authenticate. See
[Section 3](#3-aws-credentials) for exactly how to store them.

---

### DVC (Data Version Control)

**What it is:** A tool that brings version control to large data files. It
works alongside git. Instead of storing a 500 MB CSV in git (which would
break everything), DVC stores a tiny text file called a `.dvc` pointer that
contains the file's content hash. The actual file lives in S3.

**Why we use it:** It makes data reproducible in the same way git makes code
reproducible. The pointer file is committed to git. When you check out an old
commit and run `dvc pull`, you get the exact dataset that existed at that
commit — not whatever happens to be in Box today.

**What a `.dvc` file looks like:**

```yaml
outs:
- md5: 3a1f...e92b
  size: 524288000
  path: data/raw/survey_dump.csv
```

This file is committed to git. The actual CSV lives in S3, identified by that
hash.

**What you do with it:** You never run `dvc` commands directly. The Makefile
handles everything: `make push DATA=file.csv` versions a new file; `make pull`
fetches the approved version.

---

### rclone

**What it is:** A command-line tool for copying files between local storage
and cloud storage services. We use it exclusively for Box transfers.

**Why we use it:** Box deprecated its WebDAV interface, so you can no longer
mount it as a network drive. rclone supports Box natively and handles
authentication, incremental transfers, and progress reporting.

**What you do with it:** You never run `rclone` commands directly. The
Makefile calls `rclone` inside `make ingest`, `make publish-raw`, and
`make deliver`.

---

### uv

**What it is:** A fast Python package manager. It replaces pip, virtualenv,
and pip-tools in a single tool.

**Why we use it:** It is dramatically faster than pip and produces a
reproducible lock file (`uv.lock`) that is committed to git. Anyone who runs
`uv sync` gets exactly the same Python environment, regardless of when or
where they run it.

**What you do with it:** You run `uv sync` as part of `make pull` to keep
your environment current. You do not run `pip install` for anything.

---

## 2. One-time machine setup

You need to install four tools once on each machine you work on. After that,
`make setup` inside any DPIC project repo handles everything else.

### Install git

**macOS:**
```bash
xcode-select --install
```
If it says "already installed", you are done.

**Windows Subsystem for Linux (WSL):**
```bash
sudo apt-get update && sudo apt-get install -y git
```

### Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Close and reopen your terminal after this.

### Install rclone

The `make setup` script will install rclone automatically if it is not found.
To install it manually:

**macOS:**
```bash
brew install rclone
```

**Linux / WSL:**
```bash
curl https://rclone.org/install.sh | sudo bash
```

### Install the AWS CLI

**macOS:**
```bash
brew install awscli
```

**Linux / WSL:** `make setup` installs it automatically. To install manually:
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip /tmp/awscliv2.zip -d /tmp
/tmp/aws/install --install-dir ~/.local/opt/aws-cli --bin-dir ~/.local/bin
```

### Verify everything is installed

```bash
git --version
uv --version
rclone --version
aws --version
```

All four commands should print a version number without errors.

---

## 3. AWS credentials

AWS credentials are how S3 knows who you are and whether you are allowed to
read and write data. You will receive an **access key ID** and a **secret
access key** from the project lead (Yashaswi). These are like a username and
password — do not share them, do not commit them to git, and do not paste
them into Slack or email.

### Where to store them

AWS credentials go in a file called `~/.aws/credentials`. The tilde (`~`)
means your home directory (e.g. `/home/yourname/` on Linux or
`/Users/yourname/` on macOS).

Create or open the file:

```bash
mkdir -p ~/.aws
nano ~/.aws/credentials
```

Add the following block, replacing the placeholder values with what the
project lead sends you:

```ini
[dpic]
aws_access_key_id     = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Save and close (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

**The `[dpic]` part is the profile name.** It must be exactly `dpic` (all
lowercase). The `make setup` script configures DVC to use this profile
automatically.

### Verify the credentials work

```bash
aws sts get-caller-identity --profile dpic
```

You should see a JSON response with an `Account` number and a `UserId`. If
you see `InvalidClientTokenId` or `SignatureDoesNotMatch`, the key or secret
was entered incorrectly.

### Credentials file safety rules

- The `~/.aws/credentials` file should never be committed to git. It lives
  outside every repository for this reason.
- If you accidentally expose credentials, notify the project lead immediately
  so the keys can be rotated.
- AWS credentials expire when the project lead rotates them. If you start
  seeing S3 permission errors, ask for a new key pair.

### The `.dvc/config.local` file

When `make setup` runs `dvc remote modify --local s3remote profile dpic`, it
writes a file called `.dvc/config.local` that tells DVC to authenticate with
the `[dpic]` AWS profile. This file is in `.gitignore` — it is local to your
machine and is never committed.

---

## 4. Box authentication

rclone connects to Box through OAuth2. This requires a browser interaction
once per machine to authorize access. `make setup` starts this process
automatically.

### What happens during setup

1. `make setup` calls `rclone config reconnect box:`.
2. rclone prints a URL and opens your browser (or asks you to open it).
3. You log in to Box with your DPIC/UChicago credentials and click **Grant
   access to Box**.
4. rclone stores an OAuth token in its configuration file
   (`~/.config/rclone/rclone.conf` on Linux/macOS).

### If the browser does not open automatically

Copy the URL rclone prints and paste it into your browser manually.

### Token expiry

Box OAuth tokens eventually expire. If you see `rclone: failed to refresh
token` or similar errors, re-authenticate:

```bash
rclone config reconnect box:
```

### Verify Box access

```bash
rclone ls box:
```

You should see a list of folders in your Box root. If you see a specific
project folder (e.g. `DPIC/grievance-analysis`), verify you can list it:

```bash
rclone ls "box:DPIC/grievance-analysis/Data/Raw/"
```

---

## 5. GitHub SSH access

DPIC project repos depend on a private Python package called `dpic` hosted in
the GitHub organization. pip/uv cannot install private GitHub packages over
HTTPS without a personal access token, so we use SSH authentication instead.

### Generate an SSH key (if you do not have one)

```bash
ssh-keygen -t ed25519 -C "your.email@example.com"
```

Press Enter to accept the default file location (`~/.ssh/id_ed25519`) and
choose a passphrase (or leave blank).

### Add the public key to GitHub

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the output. In GitHub: **Settings → SSH and GPG keys → New SSH key**.
Paste the key, give it a descriptive name (e.g. "Work laptop"), and save.

### Test the connection

```bash
ssh -T git@github.com
```

You should see:

```
Hi yourname! You've successfully authenticated, but GitHub does not
provide shell access.
```

### Request access to the organization

You must be a member of the `Data-Policy-and-Innovation-Centre` GitHub
organization. Ask the project lead to add you. Until you are added, `uv sync`
and `make setup` will fail with a 403 or "repository not found" error when
trying to fetch the `dpic` package.

---

## 6. Joining a project for the first time

Once you have completed Sections 2–5, joining any DPIC project is two
commands.

### Step 1 — Clone the repository

```bash
git clone git@github.com:Data-Policy-and-Innovation-Centre/<repo-name>.git
cd <repo-name>
```

Replace `<repo-name>` with the repository name the project lead gives you.

> **WSL users:** Clone inside the Linux filesystem, not under `/mnt/c/`.
> Python virtual environments often fail on the Windows filesystem. Use:
> ```bash
> mkdir -p ~/Documents/GitHub
> cd ~/Documents/GitHub
> git clone git@github.com:Data-Policy-and-Innovation-Centre/<repo-name>.git
> ```

### Step 2 — Run setup

```bash
make setup
```

This will:

1. Install rclone and the AWS CLI if they are not already on your machine
2. Create the Python virtual environment and install all dependencies via
   `uv sync`
3. Install the OpenCode skills used in this project
4. Configure DVC to use your `[dpic]` AWS profile
5. Authenticate rclone with Box (browser will open)
6. Pull the latest approved data versions via `dvc pull`
7. Install the `nbstripout` git hook that strips notebook outputs before
   every commit

Setup takes 5–15 minutes on the first run, mostly due to package installation.

### What to do if setup fails

| Error message | Likely cause | Fix |
|---|---|---|
| `Git is required` | git not installed | See Section 2 |
| `Go to this link` / auth loop | Box auth issue | Complete the browser step |
| `NoCredentialsError` / S3 error | AWS credentials missing | See Section 3 |
| `Permission denied (publickey)` | SSH key not on GitHub | See Section 5 |
| `repository not found` on `dpic` pkg | Not in GitHub org | Ask project lead |
| `Python virtual environments often fail` | Repo on Windows filesystem (WSL) | Re-clone under `~/Documents/` |

---

## 7. Daily workflow

### Command reference

| Intent | Command |
|---|---|
| Start of day — get latest code and data | `make pull` |
| Copy a stakeholder file from Box | `make ingest DATA=filename.csv` |
| Version and share a file with the team | `make push DATA=filename.csv` |
| Send a locally-pulled API file to Box so stakeholders can see it | `make publish-raw DATA=filename.csv` |
| Run the analysis pipeline | `make run` |
| Regenerate figures and tables only | `make exhibits` |
| Copy exhibits to Box | `make deliver` |
| Check what has changed | `make status` |
| Show configured Box and local paths | `make box-paths` |

### `make pull` — what it does

1. Pulls the latest code from GitHub (`git pull --ff-only`)
2. Updates the Python environment to match `uv.lock` (`uv sync`)
3. Downloads the approved data versions from S3 (`dvc pull`)

Run this every morning before you start work.

### `make push DATA=filename.csv` — what it does

1. Versions the file with DVC (`dvc add data/raw/filename.csv`)
2. Commits the `.dvc` pointer file to git
3. Pushes the data blob to S3 (`dvc push`)
4. Pushes the git commit to GitHub (`git push`)

Run this after you pull new data via an API script or after receiving an
updated file.

### `make ingest DATA=filename.csv` — what it does

Copies a file **from Box into your local `data/raw/`**. The original Box file
is not modified. Use this when a stakeholder has placed a file in the project's
Box folder and you need to work with it locally.

After ingesting, version the file with `make push DATA=filename.csv` so
teammates can pull it too.

### `make publish-raw DATA=filename.csv` — what it does

Copies a file **from your local `data/raw/` up to Box**. Use this when you
have pulled data programmatically (e.g. via an API ingest script) and need to
make the raw file visible in Box for stakeholders or the ED, who do not use
the command-line toolchain.

`publish-raw` is the mirror image of `ingest`. The direction of transfer is
the only difference:

| Command | Direction | When to use |
|---|---|---|
| `make ingest` | Box → local | Stakeholder dropped a file in Box; you need it locally |
| `make publish-raw` | local → Box | You pulled data via API; stakeholders need to see it in Box |

`publish-raw` does **not** version the file in DVC. You still need to run
`make push DATA=filename.csv` separately to create the versioned S3 copy that
teammates can pull. The two commands are usually run together:

```bash
make publish-raw DATA=api_dump.csv   # share with stakeholders via Box
make push DATA=api_dump.csv          # version in S3 + share with team via DVC
```

### `make run` and `make exhibits`

Both commands run `dvc repro`, which re-executes any pipeline stage whose
inputs have changed since the last run. `make run` runs the full pipeline.
`make exhibits` re-runs all figure and table stages.

---

## 8. Data workflow in detail

### The three-location model

Every dataset exists in up to three places:

```
Box                    Local (data/raw/)       S3 (via DVC)
(stakeholder-facing)   (your working copy)     (canonical versioned store)
```

- **Box** is where stakeholders upload raw files and where we deliver
  exhibits. It has no versioning.
- **Local** is your working copy. You run scripts against files here.
- **S3** is the authoritative versioned store. `dvc pull` always brings you
  to the team-approved version.

The direction of data movement determines which `make` command you need.
There are two fundamentally different starting points — a file can originate
in Box (stakeholder upload) or originate locally (API pull). These require
opposite commands and it is the most common source of confusion.

---

### Flow A — Stakeholder uploads data to Box, you need it locally

**When:** A government counterpart, the ED, or a university partner places a
file in the project's Box folder. You need to work with it.

```
Box  ──make ingest──►  data/raw/  ──make push──►  S3
                                                    │
                                              teammates pull
                                              via make pull
```

1. Stakeholder drops the file in Box.
2. `make ingest DATA=filename.csv` — copies the file from Box into your
   local `data/raw/`. Box is not modified.
3. `make push DATA=filename.csv` — versions the file in S3 and commits the
   `.dvc` pointer to git so teammates can pull it.

---

### Flow B — You pull data via an API script, stakeholders need to see it in Box

**When:** You run an ingest script (`src/ingest/*.py`) that fetches data from
a government API or other source and writes it to `data/raw/` on your local
machine. Stakeholders and the ED need to see that raw file in Box.

```
API  ──script──►  data/raw/  ──make publish-raw──►  Box
                      │
                      └──make push──►  S3
                                        │
                                  teammates pull
                                  via make pull
```

1. Your ingest script writes the file to `data/raw/filename.csv`.
2. `make publish-raw DATA=filename.csv` — copies the file **up to Box** so
   stakeholders can see it. Your local file is not modified.
3. `make push DATA=filename.csv` — versions the file in S3 and shares it
   with the team via DVC.

Steps 2 and 3 are independent and both are usually needed:

```bash
make publish-raw DATA=api_dump.csv   # stakeholders can now see it in Box
make push DATA=api_dump.csv          # team can now pull it via DVC
```

**`publish-raw` does not version the file.** If you skip `make push`, the
file exists in Box and on your machine but no one else on the team can pull
it, and there is no versioned record in S3.

---

### The direction cheatsheet

| Where did the file start? | What do you need? | Command |
|---|---|---|
| Box (stakeholder upload) | Get it locally | `make ingest DATA=file.csv` |
| Local (API script output) | Share with stakeholders | `make publish-raw DATA=file.csv` |
| Local (any source) | Version it and share with team | `make push DATA=file.csv` |

`ingest` and `publish-raw` move files between Box and local. `push` moves
files from local into the versioned DVC/S3 store. They serve different
purposes and are not interchangeable.

### Deliver flow (exhibits go to Box)

```
outputs/  →  make deliver  →  Box/Analysis/Exhibits/
```

Copies everything in `outputs/` to the project's Box exhibits folder. Does
not delete existing Box files.

### What never to do

- **Never drag-and-drop raw data files into git.** Files in `data/raw/` and
  `data/clean/` are in `.gitignore` for this reason. Only `.dvc` pointer
  files belong in git.
- **Never run `dvc`, `rclone`, or `git` commands directly.** Use `make`.
- **Never commit a file from `data/` without using `make push` first.** The
  CI pipeline checks for this and will block the merge.

---

## 9. Common errors and fixes

### `Uncommitted changes. Commit or stash first.`

`make pull` and `make push` both require a clean git state. You have
unsaved edits in a tracked file.

```bash
git status          # see what is dirty
git add <file>
git commit -m "wip: describe what you were doing"
make pull           # now retry
```

If you are not sure what to commit, contact the project lead before doing
anything.

---

### `Usage: make <ingest|push> DATA=yourfile.csv`

You ran `make push` or `make ingest` without specifying the filename.

```bash
make push DATA=survey_dump.csv       # correct
```

---

### `git pull --ff-only` fails with `fatal: Not possible to fast-forward`

Your local branch has diverged from the remote. Do not try to fix this
yourself.

```bash
git status
git log --oneline -5
```

Send the project lead the output of both commands.

---

### S3 / DVC errors

**`NoCredentialsError`** — AWS credentials are missing or in the wrong profile.
Verify `~/.aws/credentials` has a `[dpic]` section (see Section 3).

**`An error occurred (InvalidClientTokenId)`** — The access key ID is wrong.
Re-check the credentials file for typos.

**`An error occurred (SignatureDoesNotMatch)`** — The secret access key is
wrong or has trailing whitespace. Open the credentials file and verify the
secret line.

**`An error occurred (AccessDenied)`** — Your key exists but does not have
permission. Contact the project lead to verify your IAM policy.

---

### rclone / Box errors

**`Failed to refresh token`** or **`oauth2: token expired`**

```bash
rclone config reconnect box:
```

**`directory not found`**

The Box path configured for this project may be different from what you
expect. Check:

```bash
make box-paths
```

The output shows the exact remote path rclone is using. Verify that path
exists in Box.

---

### `Permission denied (publickey)` when cloning or pushing

Your SSH key is not added to GitHub, or you are not in the organization.

```bash
ssh -T git@github.com      # should print "Hi <username>!"
```

If that fails, re-add your key (Section 5). If it succeeds but push still
fails, ask the project lead to check your repository permissions.

---

### `make setup` exits with `repository not found` on the `dpic` package

You are not yet a member of the `Data-Policy-and-Innovation-Centre` GitHub
organization, or your SSH key is not accepted. Ask the project lead to add
you.

---

## 10. Rules you must not break

These are the hard constraints that keep data reproducible and prevent data
loss. Violating them creates recovery work for the whole team.

1. **Never commit files from `data/raw/`, `data/clean/`, or `outputs/`
   directly.** Use `make push` for raw data. Exhibits are `dvc repro`
   outputs — never committed manually.

2. **Never run `git`, `dvc`, or `rclone` commands directly.** Use `make`.
   This is how the project lead has tested things work.

3. **Never force-push to `main`.** If you think you need to, contact the
   project lead.

4. **Never store AWS credentials anywhere except `~/.aws/credentials`.**
   Not in `.env`, not in a script, not in Slack.

5. **Always run `make pull` before starting work each day.** Working on a
   stale checkout creates merge conflicts and data inconsistencies.

6. **If `make pull` or `make push` fails with an error you do not
   understand, stop and ask.** Do not run additional git commands to try
   to resolve it.

---

## Appendix A — Installing on Windows (WSL2)

All DPIC tooling runs inside Windows Subsystem for Linux (WSL2). The Windows
filesystem (`C:\`) is mounted at `/mnt/c/` inside WSL, but Python virtual
environments frequently fail there. Always clone repositories into the Linux
filesystem.

**Recommended location:**

```
/home/<yourname>/Documents/GitHub/<repo-name>
```

**Installing WSL2** (if not already installed — run in Windows PowerShell as
Administrator):

```powershell
wsl --install
```

Restart your machine, then open the Ubuntu app from the Start menu.

Inside Ubuntu, run through Sections 2–5 of this guide as if you were on
Linux.

---

## Appendix B — Checking what DVC is tracking

To see all DVC-tracked files in the current project:

```bash
find . -name "*.dvc" -not -path "./.dvc/*"
```

To see whether local files match the latest DVC pointers:

```bash
make status
```

The `=== DVC (local) ===` section shows any files that differ from their
committed `.dvc` pointer. The `=== DVC (remote) ===` section shows what is
in S3 vs what is in your local cache.

---

## Appendix C — Putting this guide on the GitHub organization page

GitHub organization profile pages display the contents of a special
repository named `.github` inside the organization. To host this guide
(or any document) centrally:

### Step 1 — Create the `.github` repository

Go to `github.com/Data-Policy-and-Innovation-Centre` and create a new
repository named exactly `.github`. It should be **public** for the profile
to show up on the org page, or **private** if you only want it visible to
org members.

### Step 2 — Add the profile README

Inside the `.github` repository, create the file at exactly this path:

```
profile/README.md
```

Whatever you put in `profile/README.md` appears on the organization's
GitHub homepage at `github.com/Data-Policy-and-Innovation-Centre`.

### Step 3 — Link to this guide from the profile README

The profile README is best kept short — a paragraph about DPIC with links
to the right places. Host this full analyst guide as a wiki page (see below)
and link to it.

### Using GitHub Wiki for the full guide

GitHub Wikis are the right home for long-form internal documentation. Each
repository has its own wiki, but the `.github` repo's wiki is a natural
organization-level docs hub.

1. In the `.github` repository, go to the **Wiki** tab and click
   **Create the first page**.
2. Paste the contents of this guide as the `Home` page or create a new page
   named `Analyst Guide`.
3. Link to it from `profile/README.md`:

```markdown
## Getting started

New analysts: read the [Analyst Guide](https://github.com/Data-Policy-and-Innovation-Centre/.github/wiki/Analyst-Guide) before running `make setup`.
```

### Keeping the guide up to date

Treat the wiki like code: open a PR to the `.github` repository when the
workflow changes (new AWS profile name, new Box folder structure, etc.) and
update the wiki as part of that PR. Stale documentation is worse than no
documentation.
