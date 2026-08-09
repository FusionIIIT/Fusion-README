# Quickstart: Ubuntu and WSL

Paste each block in order. Nothing to decide, nothing to skip.

- **Windows:** do step 0, then run everything in the **Ubuntu** window.
- **Ubuntu / WSL:** start at step 1.
- **macOS:** use [QUICKSTART-macOS.md](QUICKSTART-macOS.md) instead.
- **Anything else:** [Fusion_System_Setup.md](Fusion_System_Setup.md).

Each block is followed by what it should print. If yours differs, stop there
and check [When something goes wrong](Fusion_System_Setup.md#when-something-goes-wrong).

---

### 0. Windows only: install Ubuntu

WSL is Ubuntu running inside Windows. Not a virtual machine to manage, not a
dual boot, nothing removed. One extra app in the Start menu that opens a
terminal; Windows and your files are untouched.

It is needed because `make`, which installs and runs both services, does not
exist on Windows, and the build file points at Linux paths. Inside WSL you run
what everyone else runs.

In **PowerShell as Administrator**:

```powershell
wsl --install -d Ubuntu-24.04
```

Reboot. Ubuntu opens and asks you to invent a username and password. Everything
below goes in that Ubuntu window. The prompt looks like `you@machine:~$`.

**Turn on copy-paste before you go further.** Right-click the Ubuntu window's
title bar → **Properties** → **Options** tab → tick **Use Ctrl+Shift+C/V as
Copy/Paste** → OK. Then `Ctrl+Shift+V` pastes and `Ctrl+Shift+C` copies.

Without it `Ctrl+V` does nothing in that window and you will be retyping every
block on this page by hand. If there is no Properties entry you are on Windows
Terminal, where it already works.

---

### 1. Set two variables

```bash
cd ~
echo 'export FUSION=$HOME/Documents/Fusion' >> ~/.bashrc
echo "export PGPASSWORD='hello123'" >> ~/.bashrc
source ~/.bashrc
echo "FUSION=$FUSION"
```

Shows:

```
FUSION=/home/yourname/Documents/Fusion
```

Writing them to `~/.bashrc` means every Ubuntu window gets them automatically,
including the two more you open later. Pick your own password if you prefer, but
use the same one everywhere.

If `FUSION=` comes back empty later, you are in a window opened before this
step. Run `source ~/.bashrc` in it.

> `PGPASSWORD` in `~/.bashrc` is fine for a throwaway local database. Remove it
> when you are done: `sed -i '/PGPASSWORD/d' ~/.bashrc`

---

### 2. Install the tools

```bash
sudo apt update
sudo apt install -y git curl build-essential python3.12 python3.12-venv python3.12-dev
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs postgresql postgresql-client
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
```

`sudo` asks for your Ubuntu password and shows nothing as you type.

```bash
git --version; python3 --version; uv --version; node --version; npm --version
```

Shows:

```
git version 2.43.0
Python 3.12.3
uv 0.12.3 (x86_64-unknown-linux-gnu)
v20.20.2
10.8.2
```

---

### 3. Start PostgreSQL and create the login

```bash
test -n "$PGPASSWORD" || echo "STOP: PGPASSWORD is empty — go back and paste step 1"
sudo service postgresql start
sudo -u postgres psql -c "CREATE ROLE fusion_admin LOGIN SUPERUSER;" 2>/dev/null
sudo -u postgres psql -c "ALTER ROLE fusion_admin WITH LOGIN SUPERUSER PASSWORD '$PGPASSWORD';"
```

Safe to repeat, and safe after a failed attempt: the `ALTER ROLE` always runs
and sets the password. If the first line printed **STOP**, paste step 1 and run
this block again before continuing.

Now check it:

```bash
psql -U fusion_admin -h 127.0.0.1 -d postgres -tAc "select current_user;"
```

Shows:

```
fusion_admin
```

If that asks for a password or refuses you, the role's password is not what
`$PGPASSWORD` holds. Run the `ALTER ROLE` above.

On WSL, `sudo service postgresql start` must be re-run each time you reboot.

---

### 4. Load the data

```bash
mkdir -p $FUSION && cd $FUSION
curl -L -o fusion-dev.dump https://github.com/FusionIIIT/Fusion-README/raw/main/fusion-dev.dump
psql -U fusion_admin -h 127.0.0.1 -d postgres -c "CREATE DATABASE fusionlab;"
pg_restore -U fusion_admin -h 127.0.0.1 -d fusionlab --no-owner fusion-dev.dump
psql -U fusion_admin -h 127.0.0.1 -d postgres -c "CREATE DATABASE fusion_system_db;"
psql -U fusion_admin -h 127.0.0.1 -d postgres -c "CREATE DATABASE fusion_integrated;"
```

```bash
psql -U fusion_admin -h 127.0.0.1 -d fusionlab -tAc "select count(*) from auth_user;"
```

Shows:

```
3277
```

---

### 5. Get the code

```bash
cd $FUSION
gh auth login       # skip if you have no gh; use the git clone lines below
gh repo fork FusionIIIT/Fusion_System_Administrator --clone
gh repo fork FusionIIIT/Fusion-Integrated --clone
```

No `gh`? Fork both on GitHub, then:

```bash
cd $FUSION
git clone https://github.com/<your-username>/Fusion_System_Administrator.git
git clone https://github.com/<your-username>/Fusion-Integrated.git
```

---

### 6. Install the IAM

```bash
cd $FUSION/Fusion_System_Administrator/Backend
python3 -m venv venv
./venv/bin/pip install -r requirements.txt
```

---

### 7. Configure the IAM

```bash
cd $FUSION/Fusion_System_Administrator
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
```

```bash
cd Backend/backend
./../venv/bin/python manage.py check
```

Shows:

```
System check identified no issues (0 silenced).
```

---

### 8. Build the IAM's tables

```bash
./../venv/bin/python manage.py migrate
./../venv/bin/python manage.py migrate --database system_db
./../venv/bin/python manage.py sync_identity
```

Shows:

```
users written      3277
academic standings 3004
succeeded
```

---

### 9. Issue a service token

```bash
./../venv/bin/python manage.py service_token --issue fusion-integrated
```

Shows:

```
IAM_SERVICE_TOKEN=fsvc_...
```

Copy the `fsvc_...` value into the next line and run it:

```bash
export TOKEN=fsvc_paste-it-here
```

---

### 10. Start the IAM (leave this window open)

```bash
./../venv/bin/python manage.py runserver 127.0.0.1:8001
```

**Open a new Ubuntu window.** It already has `FUSION` and `PGPASSWORD` from
step 1; add your token with `export TOKEN=fsvc_...`. Then:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8001/api/iam/v1/me
```

Shows:

```
401
```

`401` is correct. The server is up and refusing you because you are not logged
in.

---

### 11. Install placement

```bash
cd $FUSION/Fusion-Integrated
make install
cd client && npm install && cd ..
```

---

### 12. Configure placement

```bash
cd $FUSION/Fusion-Integrated
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

---

### 13. Build placement's tables

```bash
make migrate
.venv/bin/python manage.py seed_modules
```

Shows:

```
1 module(s) registered
```

---

### 14. Load the permissions

```bash
make permissions
cd $FUSION/Fusion_System_Administrator/Backend/backend
./../venv/bin/python manage.py seed_iam_permissions --manifest $FUSION/Fusion-Integrated/registry/permissions.json
```

Shows:

```
56 added, 0 revoked; 45 total
```

Skip this and every screen says forbidden.

---

### 15. Start placement (leave this window open)

```bash
cd $FUSION/Fusion-Integrated
make dev
```

**Open a third Ubuntu window**, then:

```bash
cd $FUSION/Fusion-Integrated/client
npm run dev
```

---

### 16. Log in

Open **http://localhost:5173** in your normal browser.

```bash
psql -U fusion_admin -h 127.0.0.1 -d fusionlab -tAc "select username from auth_user where username like 'stu%' limit 5;"
```

Shows:

```
stu00001, stu00002, ...
```

Sign in as any of those, password **`fusion123`**.

**You should get a sidebar with Placement in it.** That is the whole system
working. Done.

---

### 17. The admin console (only if you work on the administrator)

A different login from the app. Placement signs in students and faculty from
`fusionlab`. The console signs in operators, who live in `fusion_system_db` and
do not exist until you make one. `stu00001` will not work here.

```bash
cd $FUSION/Fusion_System_Administrator/Backend/backend
./../venv/bin/python manage.py createsuperuser --database system_db
```

It asks for a username, email and password. **Do not drop `--database system_db`.** Without it the account goes to the ERP
database, where the console never looks, and the login fails even though your
credentials are right.

Its web client proxies to port 8000, but your IAM is on 8001, so point it at the
right one and start it on a free port:

```bash
cd $FUSION/Fusion_System_Administrator/client
sed -i 's|http://localhost:8000|http://localhost:8001|' vite.config.js
npm install
npm run dev -- --port 5175
```

Open **http://localhost:5175** and sign in with the operator account you just
made. Skip the `sed` and the page loads but every request 404s; skip `--port`
and it collides with placement on 5173.

---

## What you have running

| Window | Command | Address |
|---|---|---|
| 1 | `manage.py runserver 127.0.0.1:8001` | 8001 |
| 2 | `make dev` | 8002 |
| 3 | `npm run dev` | **5173 ← the app** |
| 4 | `npm run dev -- --port 5175` | 5175, console only |

Window 1 must start first. New windows pick up `FUSION` and `PGPASSWORD` from
`~/.bashrc` on their own.

After a reboot: `sudo service postgresql start`, then windows 1, 2, 3 again.

---

## If a step fails

Paste the command and the full error text. Do not summarise it.
