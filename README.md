# Fusion

Documentation for the Fusion ERP rebuild — the IAM, the placement platform, and
how the modules map onto the existing database.

## Setting up a machine

**Start here. Pick your operating system:**

| You are on | Follow |
|---|---|
| **Windows** | [QUICKSTART.md](QUICKSTART.md). Step 0 installs Ubuntu; the rest runs inside it |
| **Ubuntu, WSL or other Linux** | [QUICKSTART.md](QUICKSTART.md) |
| **macOS** | [QUICKSTART-macOS.md](QUICKSTART-macOS.md) |

Each is one linear list of commands, with the expected output under every block.
Paste top to bottom. Allow 2-3 hours the first time, mostly downloading.

You finish with placement running at `http://localhost:5173`, logged in as a
student.

### The other two setup documents

You should not need these to get running.

- **[Fusion_System_Setup.md](Fusion_System_Setup.md)**: the same setup as a
  reference, with every option and platform plus a troubleshooting table. Go
  here when a quickstart step misbehaves.
- **[Fusion_Database_Setup.md](Fusion_Database_Setup.md)**: PostgreSQL on its
  own, if you only need the database.

## Reporting problems

If a step fails, paste the command and the full error text. Do not summarise it.
The exact wording is usually what identifies the cause.

## Everything else

[00_A_INDEX.md](00_A_INDEX.md) — the module integration guides: which tables
each module owns, and how it syncs with academic.

## The repositories

| Repository | What it is |
|---|---|
| [Fusion_System_Administrator](https://github.com/FusionIIIT/Fusion_System_Administrator) | the IAM — identity, roles, permissions |
| [Fusion-Integrated](https://github.com/FusionIIIT/Fusion-Integrated) | placement |
| [Fusion](https://github.com/FusionIIIT/Fusion) | the old monolith. Database only; you never run its code |
| [Fusion-client](https://github.com/FusionIIIT/Fusion-client) | the old React frontend |
