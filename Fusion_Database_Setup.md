# PostgreSQL Setup

Installs PostgreSQL and creates the login the services use. **Loading the data
is not here** — that is Step 1 of
[Fusion_System_Setup.md](Fusion_System_Setup.md), which you should be following.
Come back there when this is done.

Requires **PostgreSQL 14 or newer** on port 5432.

---

## Pick a password first

You will type it several times. Choose one now and keep it for the whole setup:

```bash
export PGPASSWORD='choose-something-here'
```

Paste that into every new terminal. Without it `psql` stops and waits for input
without printing a prompt, which reads as a hang.

> Older notes in this project used a shared password. Do not reuse it — this
> role is a superuser on your machine.

---

## Install it

<details open>
<summary><b>macOS</b></summary>

```bash
brew install postgresql@14
brew services start postgresql@14

echo 'export PATH="/opt/homebrew/opt/postgresql@14/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

On an Intel Mac replace `/opt/homebrew` with `/usr/local`.
</details>

<details open>
<summary><b>Windows — install it inside WSL</b></summary>

You should already be in WSL from Step 0 of the setup guide. **Install
PostgreSQL inside WSL, not on Windows.** A Windows-side PostgreSQL with the
services running in Linux works, but the two disagree about `localhost` and file
paths often enough that it is not worth the afternoon.

Open the Ubuntu window and follow the **Linux** block below.
</details>

<details>
<summary><b>Linux (Ubuntu / Debian)</b></summary>

```bash
sudo apt update
sudo apt install -y postgresql postgresql-client
sudo service postgresql start
```

Unversioned on purpose: **Ubuntu 24.04 has no `postgresql-14` package** and
asking for one fails. You get 16 on 24.04 and 14 on 22.04, and anything from 14
up is fine.

On a systemd machine `sudo systemctl enable --now postgresql` starts it with the
computer instead. WSL does not run systemd by default — there, run
`sudo service postgresql start` once per session.
</details>

**Check it is up:**

```bash
pg_isready
```

```
/tmp:5432 - accepting connections
```

---

## Create the `fusion_admin` role

This is the login every service connects as.

<details open>
<summary><b>macOS</b></summary>

```bash
psql postgres -c "CREATE ROLE fusion_admin WITH LOGIN SUPERUSER PASSWORD '$PGPASSWORD';"
```

**If that fails with `password authentication failed for user <you>`,** your
PostgreSQL came from the EnterpriseDB installer rather than Homebrew, and your
macOS account is not a superuser in it. Use the `postgres` account instead —
it will ask for the password you set during that installer:

```bash
psql -U postgres -h 127.0.0.1 -c "CREATE ROLE fusion_admin WITH LOGIN SUPERUSER PASSWORD '$PGPASSWORD';"
```
</details>

<details>
<summary><b>Linux and WSL</b></summary>

```bash
test -n "$PGPASSWORD" || echo "STOP: PGPASSWORD is empty — set it first"
sudo -u postgres psql -c "CREATE ROLE fusion_admin LOGIN SUPERUSER;" 2>/dev/null
sudo -u postgres psql -c "ALTER ROLE fusion_admin WITH LOGIN SUPERUSER PASSWORD '$PGPASSWORD';"
```

Two statements rather than one so this works whether or not the role already
exists: the create is allowed to fail, the alter always runs and is what sets
the password.

Expect `CREATE ROLE` and nothing else. `NOTICE: empty string is not a valid
password, clearing password` means `$PGPASSWORD` was empty — usually a terminal
that never got the export. The role is then created **with no password at all**,
which still says `CREATE ROLE`, and every later connection fails. Set the
variable and repair it with
`sudo -u postgres psql -c "ALTER ROLE fusion_admin PASSWORD '$PGPASSWORD';"`.

`sudo` asks for your own password and shows nothing as you type. That is normal.
</details>

**Check it works:**

```bash
psql -U fusion_admin -h 127.0.0.1 -d postgres -tAc "select current_user, version();"
```

```
fusion_admin|PostgreSQL 14.23 ...
```

That is everything. **Go back to Step 1 of
[Fusion_System_Setup.md](Fusion_System_Setup.md)** to create the databases and
restore the data.

---

## pgAdmin — optional

A graphical client. Nothing in this project needs it; `psql` does everything the
guides ask for.

```bash
brew install --cask pgadmin4        # macOS
sudo apt install -y pgadmin4-desktop # Linux, after adding their apt repository
```

---

## When something goes wrong

| What you see | Fix |
|---|---|
| `NOTICE: empty string is not a valid password` | `$PGPASSWORD` was empty. The role has no password — set the variable, then `ALTER ROLE fusion_admin PASSWORD '...'` |
| `psql` prints nothing and waits | No `PGPASSWORD`. Export it |
| `psql: command not found` | PostgreSQL is installed but not on `PATH` — redo the `export PATH` line, open a new terminal |
| `could not connect to server` | The server is not running: `brew services start postgresql@14`, or `sudo service postgresql start` |
| `password authentication failed for user <your name>` | You are not a superuser in this install. Use the `-U postgres` form above |
| `role "fusion_admin" already exists` | Harmless — the `ALTER ROLE` line after it still sets the password |
| `FATAL: database "postgres" does not exist` | The cluster was never initialised — reinstall, or `initdb` it |
| Port 5432 already in use | Another PostgreSQL is running. `lsof -i :5432` to find it |
