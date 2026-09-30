# Fusion — Full System Setup

> ## Setting up for the first time? Use the quickstart for your machine
>
> | You are on | Follow |
> |---|---|
> | **Windows** | [QUICKSTART.md](QUICKSTART.md) — step 0 installs Ubuntu environment via wsl, the rest runs inside it |
> | **Ubuntu, WSL or other Linux** | [QUICKSTART.md](QUICKSTART.md) |
> | **macOS** | [QUICKSTART-macOS.md](QUICKSTART-macOS.md) |
>
> Those are one linear list of commands each, no options and nothing to decide.
> **Come back to this page when a step does not do what it says** — the
> [troubleshooting table](#when-something-goes-wrong) at the bottom is here, and
> so is every alternative the quickstarts leave out.

This page is the reference: the same setup with every platform, option and
explanation kept in. Copy each block, paste it, check the output matches.

Set aside **2–3 hours** the first time. Most of that is downloading.

---

## On Windows? Do this before anything else

**Stop and install WSL first.** Nothing in this guide works in Command Prompt or
PowerShell.

**What WSL is:** Ubuntu running inside Windows. Not a virtual machine you have
to manage, not a dual boot, and nothing is removed or replaced. You get one
extra app in the Start menu that opens a terminal. Windows carries on as normal,
your files stay where they are, and you use the same browser and editor. VS Code
has a WSL extension so it edits those files directly.

**Why it is needed:** the project is built and run with Linux tooling. `make` is
the command that installs and starts both services, and Windows does not have
it. Installing `make` on its own does not help either, because the build file
points at `.venv/bin/python`, a path that only exists on Linux and macOS. The
same is true of the shell commands used throughout to write config files.

You could run each underlying command by hand in PowerShell. Nobody has, so you
would be debugging these instructions rather than the project. Inside WSL you
run exactly what everyone else runs, including the servers and the Mac
developers, so you hit the same problems and the same fixes.

It is a one-time install of a few minutes.

Open **PowerShell as Administrator** — Start menu, right-click PowerShell, *Run
as Administrator* — and run:

```powershell
wsl --install -d Ubuntu-24.04
```

Reboot when it asks. Ubuntu opens by itself and asks you to invent a username
and password; that is your Linux account, unrelated to Windows. If it does not
open, launch **Ubuntu** from the Start menu.

**Then turn on copy-paste.** Right-click the Ubuntu window's title bar →
**Properties** → **Options** tab → tick **Use Ctrl+Shift+C/V as Copy/Paste** →
OK. `Ctrl+V` does nothing in that window until you do, and this guide is mostly
blocks to paste. `Ctrl+Shift+V` pastes afterwards. No Properties entry means you
are on Windows Terminal, where it already works.

**Everything from here happens in that Ubuntu window.** You can tell them apart
by the prompt:

```
C:\Users\you>                 ← Command Prompt. Wrong window.
PS C:\Users\you>              ← PowerShell. Wrong window, except for the line above.
you@machine:~$                ← Ubuntu. This is the one.
```

Typing a command from this guide into the wrong one gives you
`'export' is not recognized as an internal or external command`, or
`'make' is not recognized`. That is the only thing it means — you are in a
Windows shell.

macOS and Linux: nothing to do, carry on.

---

## Do this first, in every new terminal

You will end up with several terminal windows open. **Paste this into each one**
— every block below depends on it. On Windows, that means each **Ubuntu**
window.

```bash
export FUSION=~/Documents/Fusion
export PGPASSWORD='your-postgres-password'
```

Replace the password with the one you set for `fusion_admin` in Step 1. Without
`PGPASSWORD` every database command stops and waits for input without saying so.

---

## What you are building

Four things running at once:

| Port | What | Started in |
|---|---|---|
| 5432 | PostgreSQL — all the data | Step 1 |
| 8001 | **The IAM** — who you are, what you may do | Step 3 |
| 8002 | Fusion-Integrated — placement | Step 5 |
| 5173 | The placement web interface ← **open this in a browser** | Step 5 |

**The IAM must be running or nothing else works.** Everything asks it on every
request.

That set is enough to build and test a module. It is **not** what a student sees.
The portal — the legacy `Fusion` backend and the `Fusion-client` React app — is the
shell every module renders inside, and it owns the sidebar, the header and the role
switcher. Add it when you need to see your module the way it ships, in
[Step 6](#step-6--the-portal-optional-until-you-need-the-real-sidebar).

A server command never finishes — that is what running means. Open a **new**
terminal for the next step instead of pressing `Ctrl+C`.

---

## Step 0 — Install the tools

```bash
git --version; python3 --version; uv --version; node --version; npm --version
```

Anything that says *command not found* needs installing. Required: **Git**,
**Python 3.12+**, **uv**, **Node 20+** (npm comes with it), **PostgreSQL 14+**.

<details>
<summary><b>macOS</b></summary>

```bash
# Homebrew first, if you do not have it
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install git python@3.12 node uv
xcode-select --install          # some Python packages compile from source
```
</details>

<details open>
<summary><b>Windows — run the Ubuntu block below, inside WSL</b></summary>

You installed WSL at the top of this guide. Ubuntu **is** your Linux, so use the
**Ubuntu / Debian** instructions below and ignore everything Windows-specific
from here on.

Two things to get right:

- **`cd ~` first.** A fresh Ubuntu window often opens in
  `/mnt/c/WINDOWS/system32`, which is the Windows disk seen from Linux and is
  several times slower for this work. Keep everything in your Linux home, which
  is where `$FUSION` points.
- **Reach the servers from your normal Windows browser** at
  `http://localhost:5173`. WSL forwards the ports for you.
</details>

<details>
<summary><b>Ubuntu / Debian</b></summary>

```bash
sudo apt update
sudo apt install -y git curl build-essential python3.12 python3.12-venv python3.12-dev

curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
```

Ubuntu will suggest `snap install astral-uv` when it cannot find `uv`. **Ignore
it** — snapd does not run under WSL. Use the installer above.

**On Ubuntu 22.04 the first command fails** — 22.04 has no `python3.12`. Add the
PPA first, then repeat it:

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install -y python3.12 python3.12-venv python3.12-dev
```

24.04 has it already and needs none of this.

**Never run `sudo npm install`** — it creates root-owned files in the project and
every later command fails on permissions.
</details>

<details>
<summary><b>Fedora / Arch</b></summary>

```bash
# Fedora
sudo dnf install -y git python3.12 nodejs npm
curl -LsSf https://astral.sh/uv/install.sh | sh

# Arch
sudo pacman -S git python nodejs npm uv
```
</details>

You do **not** need Docker. Redis is optional — without it the services fall back
to an in-memory cache automatically.

---

## Step 1 — PostgreSQL and the data

Install PostgreSQL first: **[Fusion_Database_Setup.md](Fusion_Database_Setup.md)**
covers it per OS, along with creating the `fusion_admin` login. Then come back
here for the data.

```bash
mkdir -p $FUSION && cd $FUSION
curl -L -o fusion-dev.dump \
  https://github.com/FusionIIIT/Fusion-README/raw/main/fusion-dev.dump

psql -U fusion_admin -h 127.0.0.1 -d postgres -c "CREATE DATABASE fusionlab;"
pg_restore -U fusion_admin -h 127.0.0.1 -d fusionlab --no-owner fusion-dev.dump

psql -U fusion_admin -h 127.0.0.1 -d postgres -c "CREATE DATABASE fusion_system_db;"
psql -U fusion_admin -h 127.0.0.1 -d postgres -c "CREATE DATABASE fusion_integrated;"

psql -U fusion_admin -h 127.0.0.1 -d fusionlab -tAc "select count(*) from auth_user;"
```

On Windows, run that `curl` **inside the Ubuntu window**, not in a Windows
browser — a file downloaded to `C:\Users\...\Downloads` is reachable from WSL
only under `/mnt/c/`, which is a detour you do not need.

**You should see** `CREATE DATABASE` three times, then:

```
3277
```

If that is `0`, the restore did not load. Nothing after this will work.

`pg_restore`, not `psql -f` — the file is PostgreSQL's compressed custom format,
which `psql` cannot read.

**Logging in.** Usernames are `stu00001`, `fac0001`, `stf0001` and so on, and
**every account has the password `fusion123`**.

Three databases, and they are **not** interchangeable:

| Database | Holds | Created by |
|---|---|---|
| `fusionlab` | every user, grade, registration | the dump you restored |
| `fusion_system_db` | the IAM's own records | you, just now — stays empty until Step 3 |
| `fusion_integrated` | everything placement | you, just now — stays empty until Step 5 |

---

## Step 2 — Get the code

You cannot push to `FusionIIIT` directly, so you work on a **fork**.

```bash
mkdir -p $FUSION && cd $FUSION

gh auth login          # skip if already logged in
gh repo fork FusionIIIT/Fusion_System_Administrator --clone
gh repo fork FusionIIIT/Fusion-Integrated --clone
```

**You should see** two folders, each with two remotes:

```bash
git -C $FUSION/Fusion_System_Administrator remote -v
```

```
origin    https://github.com/<you>/Fusion_System_Administrator.git
upstream  https://github.com/FusionIIIT/Fusion_System_Administrator.git
```

`origin` is your fork — you push there. `upstream` is the real repository — you
pull from there.

<details>
<summary><b>Without the <code>gh</code> command</b></summary>

Click **Fork** on the GitHub page, then:

```bash
cd $FUSION
git clone https://github.com/<you>/Fusion_System_Administrator.git
cd Fusion_System_Administrator
git remote add upstream https://github.com/FusionIIIT/Fusion_System_Administrator.git
```
</details>

<details>
<summary><b>When you have something to contribute</b></summary>

```bash
git checkout -b my-change
git commit -am "describe the change"
git push -u origin my-change
gh pr create -R FusionIIIT/Fusion_System_Administrator --base main
```

The pull request goes against **FusionIIIT**, not your own fork's `main`. That is
the usual first mistake.

To pick up other people's work: `git fetch upstream && git rebase upstream/main`
</details>

You do **not** need the main `Fusion` repository — only its database, which you
already restored.

---

## Step 3 — The IAM

### 3.1 Install and configure

Paste the whole block:

```bash
cd $FUSION/Fusion_System_Administrator/Backend
python3 -m venv venv
./venv/bin/pip install -r requirements.txt

cd ..
cat > .env <<EOF
SECRET_KEY=dev-secret-change-me-0123456789abcdef
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1

DB_NAME=fusionlab
DB_USER=fusion_admin
DB_PASSWORD=$PGPASSWORD
DB_HOST=127.0.0.1
DB_PORT=5432

SYSTEM_DB_NAME=fusion_system_db
SYSTEM_DB_USER=fusion_admin
SYSTEM_DB_PASSWORD=$PGPASSWORD
SYSTEM_DB_HOST=127.0.0.1
SYSTEM_DB_PORT=5432

AUTH_COOKIE_NAME=auth_token
AUTH_COOKIE_PATH=/
AUTH_COOKIE_SAMESITE=Lax
TOKEN_TTL_HOURS=8
APP_BASE_PATH=/

EMAIL_HOST_USER=
EMAIL_HOST_PASSWORD=
EMAIL_TEST_USER=
EOF
chmod 600 .env

cd Backend/backend
./../venv/bin/python manage.py check
```

**You should see:**

```
System check identified no issues (0 silenced).
```

> **The block writes `.env` for you**, so it works whether or not the repository
> ships an example to copy. Older checkouts have none; newer ones have
> `.env.example`, and `cp .env.example .env` is then equivalent — you still have
> to fill in the two passwords.
>
> **Do not delete the three blank `EMAIL_` lines.** They have no default, so
> without them every command below dies on `ImproperlyConfigured: Set the
> EMAIL_HOST_USER environment variable`, which tells you nothing about the real
> cause.

> Ubuntu 22.04 has no `python3.12`. Add `ppa:deadsnakes/ppa` first — the Step 0
> block does this.

### 3.2 Build the tables and load the people

```bash
./../venv/bin/python manage.py migrate
./../venv/bin/python manage.py migrate --database system_db
./../venv/bin/python manage.py sync_identity
```

**You should see:**

```
users seen         3277
users written      3277
designations       3155
module grants      0
academic standings 3004
took               0.7s
succeeded
```

- **Both `migrate` lines are needed** — there are two databases. Skipping the
  second gives `relation "iam_…" does not exist` at the first login.
- `sync_identity` copies users and posts out of the ERP. Safe to re-run; takes
  about a second. Run it again whenever people or posts change.
- `module grants 0` on the first run is normal — permissions are loaded in
  **Step 5.3**, once the placement service exists to declare them.

> Nobody can do anything yet. Login will work and every screen will say
> forbidden until Step 5.3. That is expected at this point, not a fault.

### 3.3 Start it

```bash
./../venv/bin/python manage.py runserver 127.0.0.1:8001
```

**Leave this window open.** In a **new** terminal:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8001/api/iam/v1/me
```

**You should see** `401`. That is correct — the server is up and refusing you
because you have not logged in. `000` means it is not running.

<details>
<summary><b>The admin console — optional</b></summary>

```bash
cd $FUSION/Fusion_System_Administrator/client
npm install
npm run dev -- --port 5175
```

Its proxy is hardcoded to port **8000**, not the 8001 you just started, so its
API calls fail until you either also run `manage.py runserver 127.0.0.1:8000` or
change the target in `client/vite.config.js`.

It logs in with **operator accounts**, not ERP users — a separate account set in
`fusion_system_db`. `stu00001` and `fusion123` will not work here. Make one:

```bash
./../venv/bin/python manage.py createsuperuser --database system_db
```

`--database system_db` is not optional. Without it the account lands in the ERP
database, where the console never looks, and the login fails while the
credentials are perfectly correct.
</details>

---

## Step 4 — Service tokens

In the IAM folder, in a **new** terminal:

```bash
cd $FUSION/Fusion_System_Administrator/Backend/backend
./../venv/bin/python manage.py service_token --issue fusion-integrated
```

**You should see** a value starting `fsvc_`. **Copy it now** — it is stored
scrambled and cannot be shown again. Keep it handy for Step 5.

```bash
export TOKEN=fsvc_paste-the-value-here
```

On a development machine this is optional — the services start without one. In
production it is mandatory.

```bash
./../venv/bin/python manage.py service_token --list      # what exists
./../venv/bin/python manage.py service_token --rotate fusion-integrated
./../venv/bin/python manage.py service_token --revoke fusion-integrated
```

---

## Step 5 — Fusion-Integrated (placement)

### 5.1 Install and configure

```bash
cd $FUSION/Fusion-Integrated
make install
cd client && npm install && cd ..

cat > .env <<EOF
DJANGO_SETTINGS_MODULE=config.settings.dev

DB_NAME=fusion_integrated
DB_USER=fusion_admin
DB_PASSWORD=$PGPASSWORD
DB_HOST=127.0.0.1
DB_PORT=5432
DB_CONN_MAX_AGE=0

IAM_BASE_URL=http://127.0.0.1:8001
IAM_API_PREFIX=/api
IAM_SERVICE_TOKEN=$TOKEN
IAM_TIMEOUT_SECONDS=5
IAM_SESSION_CACHE_SECONDS=60
IAM_AUTH_COOKIE_NAME=fusion_session

DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
DJANGO_CORS_ALLOWED_ORIGINS=http://localhost:5173
EOF
```

- **If `$TOKEN` is not set in this terminal**, the line comes out blank. That is
  fine in development — the service starts without one. Re-run the `export TOKEN=…`
  from Step 4 if you want it filled in.
- **Two installs.** `make install` does the backend only — nothing in the
  Makefile runs `npm install`. Skipping the second gives `vite: command not
  found` later.
- **`DB_NAME` is `fusion_integrated`, not `fusionlab`.** Placement is new work
  the old system knows nothing about. Point it at `fusionlab` and `make migrate`
  will quietly create placement tables inside the ERP's database.
- **`IAM_AUTH_COOKIE_NAME` is how one login reaches every service, so the rule is
  not "make it unique".** The portal mints an IAM session under this name and
  Fusion-Integrated reads it, so **those two must match exactly** or the walk-across
  fails silently and the module asks you to log in again. What must *not* collide is
  the IAM admin console, which sets `auth_token` at path `/`: give the console
  `AUTH_COOKIE_PATH=/sysadmin/`, or set this to `fusion_session` in both the portal
  and Fusion-Integrated. Sharing the console's name at the same path means logging
  into one logs you out of the other.

### 5.2 Run it

```bash
make migrate
.venv/bin/python manage.py seed_modules
make dev
```

**You should see** from `seed_modules`:

```
registered placement_cell (active)
1 module(s) registered
```

> **Use `seed_modules`, not `make seed`.** `make seed` also runs `seed_demo`,
> which crashes on a clean database with
> `IntegrityError: ... violates check constraint
> "posting_published_has_required_content"` — it creates a published job posting
> with an empty description. That is a bug in the sample data, not your setup.

**Leave `make dev` running.** In a **new** terminal:

```bash
cd $FUSION/Fusion-Integrated/client
npm run dev
```

Open **http://localhost:5173**.

> Use `localhost`, not `127.0.0.1` — Vite listens on IPv6 and
> `curl 127.0.0.1:5173` will be refused.

### 5.3 Tell the IAM what these permissions mean

**Everything says forbidden until this runs.** You log in fine, the sidebar is
empty or shows one module, and every screen refuses you. This is the step people
skip.

The IAM has no idea what `placement_cell.offer.issue` is, or who should have it,
until it is told:

```bash
cd $FUSION/Fusion-Integrated
make permissions          # writes registry/permissions.json

cd $FUSION/Fusion_System_Administrator/Backend/backend
./../venv/bin/python manage.py seed_iam_permissions \
  --manifest $FUSION/Fusion-Integrated/registry/permissions.json
```

**You should see:**

```
directory: 5 mapping(s)
placement_cell: 40 mapping(s)
56 added, 0 revoked; 45 total
```

`make permissions` has to run first: it writes the list the IAM reads, so the
two cannot drift apart. Re-run both after adding or renaming a permission.

### 5.4 Log in

Open **http://localhost:5173** and sign in as a student with `fusion123`:

```bash
psql -U fusion_admin -h 127.0.0.1 -d fusionlab -tAc \
  "select username from auth_user where username like 'stu%' limit 5;"
```

Any of those, password `fusion123`. **You should get a sidebar with Placement in
it**, not an empty one and not a page saying forbidden. If the sidebar is empty,
re-run Step 5.3.

If you would rather check without a browser:

```bash
TOKEN=$(curl -s -X POST http://127.0.0.1:8001/api/iam/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"stu00001","password":"fusion123"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

curl -s http://127.0.0.1:8001/api/iam/v1/me -H "Authorization: Token $TOKEN" \
  | python3 -m json.tool
```

---

## Step 6 — The portal (optional until you need the real sidebar)

Steps 3–5 give you a module and its own development interface. They do not give
you what a student sees. The **portal** is the shell: the legacy `Fusion` backend
plus the `Fusion-client` React app, which owns the sidebar, the header, the role
switcher and the profile. Your module contributes pages into it and nothing else.

Set this up when you need to check how your module appears in the sidebar, that it
reaches the right roles, or that one login carries across. Skip it while you are
writing domain logic and tests — the suite needs none of it.

```bash
cd $FUSION
gh repo fork FusionIIIT/Fusion --clone
gh repo fork FusionIIIT/Fusion-client --clone
```

**The backend**, which serves `fusionlab` and mints an IAM session beside its own:

```bash
cd $FUSION/Fusion/FusionIIIT
python3.9 -m venv ../venv
../venv/bin/pip install -r ../requirements.txt

export IAM_BASE_URL=http://127.0.0.1:8001
export IAM_AUTH_COOKIE_NAME=auth_token      # must equal Fusion-Integrated's

../venv/bin/python manage.py runserver 127.0.0.1:8000
```

- **This one uses `fusionlab`**, the database you restored in Step 1. It is the old
  monolith and it owns that data — there is nothing to migrate and nothing to seed.
- **Python 3.9**, not 3.12. This is Django 3.1 and will not install on newer.
- The IAM bridge is configured entirely from the environment, so no settings file is
  touched. If the IAM is down the portal still logs you in — you simply do not get
  the plugged modules until it is back.

**The web interface**, in a new terminal:

```bash
cd $FUSION/Fusion-client
npm install
npm run dev
```

Open the port it prints — 5173, or 5174 if Fusion-Integrated's own client already
has it. Sign in with the same ERP accounts (`stu00001` / `fusion123`).

**You should see** your module in the sidebar under its own section, expanding into
the screens that role is granted — drawn by the portal, served by your service. If
the section is missing, check the grant before the frontend: only the designations
in your module's `ROLE_GRANTS` see it, and only after Step 5.3 has run.

---

## Checking it all works

```bash
# 1. The IAM is up.  401 is the correct, healthy answer.
curl -s -o /dev/null -w "IAM        %{http_code}\n" http://127.0.0.1:8001/api/iam/v1/me

# 2. Placement is up.
curl -s -o /dev/null -w "Integrated %{http_code}\n" http://127.0.0.1:8002/healthz
curl -s -o /dev/null -w "postings   %{http_code}\n" http://127.0.0.1:8002/api/v1/placement/postings

# 3. Permissions actually resolve for somebody.
cd $FUSION/Fusion_System_Administrator/Backend/backend
./../venv/bin/python manage.py shell -c "
from iam import services
from iam.models import IamUser, IamToken, IamUserDesignation, RolePermission
mapped = set(RolePermission.objects.values_list('designation', flat=True))
d = IamUserDesignation.objects.filter(designation__in=mapped).first()
u = IamUser.objects.get(erp_user_id=d.erp_user_id)
s = services.build_session(IamToken(erp_user_id=u.erp_user_id, username=u.username))
print(u.username, '|', d.designation, '|', len(s['permissions']), 'permissions')
"
```

**You should see:**

```
IAM        401
Integrated 200
postings   401
22BCS009 | student | 5 permissions
```

The username will be a different one on your machine — any name is fine. What
matters is that the last number is **not** `0`.

Test 3 picks somebody holding a post that Step 5.3 **mapped**. That filter is the
point — pick a random account instead and you get `0 permissions` from a
perfectly healthy system, because only placement is mapped so far.

`0 permissions` *from this test* means Step 5.3 did not run.

---

## The terminals you end up with

| Window | Folder | Command |
|---|---|---|
| 1 | `Fusion_System_Administrator/Backend/backend` | `./../venv/bin/python manage.py runserver 127.0.0.1:8001` |
| 2 | `Fusion-Integrated` | `make dev` |
| 3 | `Fusion-Integrated/client` | `npm run dev` |

Start window 1 first — the others depend on it. On Windows these are three
Ubuntu (WSL) windows.

With the portal from Step 6, two more:

| Window | Folder | Command |
|---|---|---|
| 4 | `Fusion/FusionIIIT` | `../venv/bin/python manage.py runserver 127.0.0.1:8000` |
| 5 | `Fusion-client` | `npm run dev` |

---

## When something goes wrong

| What you see | Fix |
|---|---|
| `ImproperlyConfigured: Set the EMAIL_HOST_USER environment variable` | The three blank `EMAIL_` lines are missing from the IAM `.env` — Step 3.1 |
| `psql` prints nothing and sits there | It wants a password. `export PGPASSWORD=…` |
| `cp: .env.example: No such file` in the IAM | Older checkouts ship none. Step 3.1 writes the file for you |
| `Unknown command: 'seed_iam_roles'` | An old checkout of the IAM. Pull the latest |
| `unrecognized arguments: --manifest` | An old checkout of the IAM. Pull the latest, or run `seed_iam_permissions` with no arguments |
| `make seed` → `posting_published_has_required_content` | Known demo-data bug. Use `seed_modules` |
| Placement tables appeared in `fusionlab` | `DB_NAME` should be `fusion_integrated` — Step 5.1 |
| Logged in, every screen forbidden | `seed_iam_permissions` never ran — Step 5.3 |
| Sidebar empty, or "1 module" | Same. Re-run Step 5.3 |
| Login fails everywhere, 401 | The IAM on 8001 is not running |
| Logging into one service logs you out of another | Two services share `IAM_AUTH_COOKIE_NAME` |
| `curl 127.0.0.1:5173` refused but the site loads | Vite is on IPv6. Use `localhost` |
| `vite: command not found` | `npm install` was not run in that `client/` folder |
| `make: .venv/bin/python: No such file` | `make install` was not run there, or `uv` was missing |
| `uv: command not found` | Install `uv` (Step 0), then re-run `make install` |
| Port 5173 already in use | The admin console also defaults to it. Use `--port 5175` |
| `relation "iam_…" does not exist` | `migrate --database system_db` was skipped |
| A user has no roles | `sync_identity` has not run since they were created |
| Your module is missing from the portal sidebar | That role is not in the module's `ROLE_GRANTS`, or Step 5.3 has not run since you changed it |
| Logged into the portal, the module asks you to log in again | The portal and Fusion-Integrated disagree on `IAM_AUTH_COOKIE_NAME` — Step 5.1 |
| `EBADENGINE` from npm | Node older than 20 |
| `make: command not found` (Windows) | You are in PowerShell. Open the Ubuntu (WSL) window — Step 0 |
| `make` runs but says `.venv/bin/python: No such file` (Windows) | Same cause. The Makefile is POSIX-only; use WSL |
| `wsl --install` says it needs a reboot | It does. Reboot, then reopen Ubuntu |
| Everything is very slow (Windows) | The code is under `/mnt/c/`. Move it to `~` inside Linux |
| `FATAL: password authentication failed` | `PGPASSWORD` does not match what you set in Step 1 |

---

## The role catalogue

Permissions decide what a post may do. The **catalogue** decides whether a person
may hold that post at all — the live data has a student holding Dean Academic,
which carries 54 permissions.

Load it once, after `sync_identity`:

```bash
cd $FUSION/Fusion_System_Administrator/Backend/backend
./../venv/bin/python manage.py seed_iam_roles
```

Leave `IAM_ENFORCE_ROLE_POLICY` at `False` to begin with. Anything the catalogue
disagrees with is then written to `iam_role_violation` and reported, but still
allowed. Switch it on only after reading that table, or you cut off access for
whoever the catalogue is wrong about, with no warning.

`Unknown command: 'seed_iam_roles'` means an old checkout of the IAM — pull the
latest. Check the file rather than asking Django, because `manage.py help` fails
the same way whether a command is missing or your `.env` is:

```bash
test -f $FUSION/Fusion_System_Administrator/Backend/backend/iam/management/commands/seed_iam_roles.py \
  && echo newer || echo older
```

---

## Where the data lives

Three databases, and a module adds tables to exactly one of them.

| Database | Owns | A new module |
|---|---|---|
| `fusionlab` | every person, grade and registration, and which designations each person holds | never writes it |
| `fusion_system_db` | **all access control** — designation → permission, designation → module, and the sidebar | adds rows through its manifest, never tables |
| `fusion_integrated` | the modules' own business tables | adds its tables here |

Two things follow, and they are the two people get wrong:

- **A module never creates an access-control table.** No `Role`, no `Permission`,
  no `UserRole`. Those exist once, in the IAM, and a module declares permissions in
  its `registry.py` that `seed_iam_permissions` turns into rows there. Ten modules
  means more rows, never more tables.
- **Modules do not define roles.** Roles are institute-wide designations owned by
  the ERP and mirrored in by `sync_identity`. A module only says which existing
  designations hold which of its permissions. If a designation the spec needs does
  not exist, the academic office creates it in the ERP — a module cannot invent one.

The old portal and the new services **share `fusionlab`**: a grade saved by one
shows up in the other, because it is the same table. That is deliberate, and it is
why nothing copies it.

---

## About the main Fusion repository

The old monolith owns the database, and it is also the portal — the shell your
module renders inside. You need its code only for [Step 6](#step-6--the-portal-optional-until-you-need-the-real-sidebar);
Steps 1–5 need the database alone.

**Do not modify that repository beyond the portal work.** Only five of its apps are
still in production, and the rest are legacy awaiting deletion.

---

## For a real deployment

1. **`sync_identity` runs on a timer you install** — nothing in the service
   schedules it. Units ship in `Backend/ops/systemd/`; see the IAM's
   `DEPLOYMENT.md` §3b. Install the freshness timer alongside it, or a stopped
   sync is silent until somebody complains that their new post does nothing.
2. **Service tokens are mandatory** — production reads `IAM_SERVICE_TOKEN` with
   `os.environ[...]` and will not start without it.
3. **`DEBUG=False`**, and every `.env` at mode `600`.
4. **Leave `IAM_ENFORCE_ROLE_POLICY` off at first.** Let it record problems, read
   them, then switch it on. Enforcing on day one revokes access from anybody the
   catalogue is wrong about, silently.
5. **`X_FRAME_OPTIONS` is `SAMEORIGIN`, not `DENY`**, or the portal cannot frame a
   module's screens. This breaks in production only — the development server sends
   no such header, so it passes locally either way.
6. **Re-run `seed_iam_permissions` on every release** that changed a `registry.py`.
   Regenerating the manifest does not grant anything; seeding it does.
