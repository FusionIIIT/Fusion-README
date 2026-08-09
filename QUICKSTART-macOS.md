# Quickstart: macOS

Paste each block in order. Nothing to decide, nothing to skip.

On Ubuntu, WSL or Windows use [QUICKSTART.md](QUICKSTART.md) instead.

Each block is followed by what it should print. If yours differs, stop there
and check [When something goes wrong](Fusion_System_Setup.md#when-something-goes-wrong).

---

### 1. Set two variables

```bash
cd ~
echo 'export FUSION=$HOME/Documents/Fusion' >> ~/.zshrc
echo "export PGPASSWORD='hello123'" >> ~/.zshrc
source ~/.zshrc
echo "FUSION=$FUSION"
```

Shows:

```
FUSION=/Users/yourname/Documents/Fusion
```

Writing them to `~/.zshrc` means every Terminal window gets them automatically.
Pick your own password if you prefer, but use the same one everywhere.

> `PGPASSWORD` in `~/.zshrc` is fine for a throwaway local database. Remove it
> when you are done: `sed -i '' '/PGPASSWORD/d' ~/.zshrc`

---

### 2. Install the tools

Homebrew first, if you do not have it:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

It prints instructions about your `PATH` at the end — **follow them**, then open
a new Terminal window.

```bash
brew install git python@3.12 node uv postgresql@14
xcode-select --install
```

`xcode-select` opens a dialog; accept it. If it says the tools are already
installed, carry on.

```bash
git --version; python3 --version; uv --version; node --version; npm --version
```

Shows something like:

```
git version 2.39.5
Python 3.12.x
uv 0.12.x
v22.x
10.x
```

Any version of Python from 3.12 up is fine, and any Node from 20 up.

---

### 3. Start PostgreSQL and create the login

```bash
brew services start postgresql@14
echo 'export PATH="/opt/homebrew/opt/postgresql@14/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
pg_isready
```

Shows:

```
/tmp:5432 - accepting connections
```

On an Intel Mac replace `/opt/homebrew` with `/usr/local`.

```bash
test -n "$PGPASSWORD" || echo "STOP: PGPASSWORD is empty — go back and paste step 1"
psql postgres -c "CREATE ROLE fusion_admin LOGIN SUPERUSER;" 2>/dev/null
psql postgres -c "ALTER ROLE fusion_admin WITH LOGIN SUPERUSER PASSWORD '$PGPASSWORD';"
```

Shows:

```
ALTER ROLE
```

Safe to repeat, and safe after a failed attempt: the `ALTER ROLE` always runs
and sets the password.

**If instead you get `password authentication failed for user <your name>`,**
your PostgreSQL did not come from Homebrew — it is the EnterpriseDB installer
under `/Library/PostgreSQL`, where your macOS account is not a superuser. Use
the `postgres` account, which will ask for the password you chose during that
installer:

```bash
psql -U postgres -h 127.0.0.1 -c "CREATE ROLE fusion_admin LOGIN SUPERUSER;" 2>/dev/null
psql -U postgres -h 127.0.0.1 -c "ALTER ROLE fusion_admin WITH LOGIN SUPERUSER PASSWORD '$PGPASSWORD';"
```

Now check it either way:

```bash
psql -U fusion_admin -h 127.0.0.1 -d postgres -tAc "select current_user;"
```

Shows:

```
fusion_admin
```

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
gh auth login
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

**Open a new Terminal window.** It already has `FUSION` and `PGPASSWORD` from
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

**Open a third Terminal window**, then:

```bash
cd $FUSION/Fusion-Integrated/client
npm run dev
```

---

### 16. Log in

Open **http://localhost:5173**.

```bash
psql -U fusion_admin -h 127.0.0.1 -d fusionlab -tAc "select username from auth_user where username like 'stu%' limit 5;"
```

Shows five usernames like `stu00001`. Sign in as any of them, password
**`fusion123`**.

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

**Do not drop `--database system_db`.** Without it the account goes to the ERP
database, where the console never looks, and the login fails even though your
credentials are right.

```bash
cd $FUSION/Fusion_System_Administrator/client
sed -i '' 's|http://localhost:8000|http://localhost:8001|' vite.config.js
npm install
npm run dev -- --port 5175
```

Open **http://localhost:5175**. The `sed` is needed because the client proxies
to 8000 while your IAM runs on 8001. The `--port` avoids the clash with
placement.

Note the `sed -i ''`. macOS needs the empty argument; Linux does not.

---

## What you have running

| Window | Command | Address |
|---|---|---|
| 1 | `manage.py runserver 127.0.0.1:8001` | 8001 |
| 2 | `make dev` | 8002 |
| 3 | `npm run dev` | **5173 ← the app** |
| 4 | `npm run dev -- --port 5175` | 5175, console only |

Window 1 must start first. New windows pick up `FUSION` and `PGPASSWORD` from
`~/.zshrc` on their own.

After a restart PostgreSQL comes back by itself — `brew services` starts it at
login. Windows 1, 2 and 3 you start again.

---

## If a step fails

Paste the command and the full error text. Do not summarise it.
