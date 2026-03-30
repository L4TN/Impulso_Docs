# Impulso Hub (Chatwoot fork) — Full Setup Guide (DEV, WSL/Ubuntu)

**Last updated:** 2025-09-13  
**Target environment:** WSL2 (Ubuntu 24.04) or native Ubuntu 22.04/24.04  
**Audience:** Developers running the app locally for development

> This guide covers end‑to‑end setup from OS packages to running the app, plus a failure catalog with concrete fixes. It assumes your repo lives at `~/Impulso_Hub` (a fork of Chatwoot).

---

## Table of Contents

1. [Overview](#overview)  
2. [System Prep](#system-prep)  
   2.1. [WSL and VS Code](#wsl-and-vs-code)  
   2.2. [Essential OS Packages](#essential-os-packages)  
3. [Language & Toolchain](#language--toolchain)  
   3.1. [Ruby via RVM (3.4.4)](#ruby-via-rvm-344)  
   3.2. [Bundler (2.5.16)](#bundler-2516)  
   3.3. [Node & pnpm](#node--pnpm)  
4. [Services](#services)  
   4.1. [PostgreSQL + pgvector](#postgresql--pgvector)  
   4.2. [Redis](#redis)  
5. [Project Setup](#project-setup)  
   5.1. [Clone / Enter Repo](#clone--enter-repo)  
   5.2. [`.env` (Development)](#env-development)  
   5.3. [Install Deps](#install-deps)  
   5.4. [DB Prepare & Seed](#db-prepare--seed)  
   5.5. [Run App (Foreman / Overmind)](#run-app-foreman--overmind)  
6. [🚀 Start Commands (TL;DR)](#-start-commands-tldr)  
7. [🧰 Make Targets (cheat sheet)](#-make-targets-cheat-sheet)  
8. [Quickstart Checklist](#quickstart-checklist)  
9. [Default Login (DEV)](#default-login-dev)  
10. [Failure Catalog (Errors & Fixes)](#failure-catalog-errors--fixes)  
    10.1. [Ruby/Bundler Issues](#rubybundler-issues)  
    10.2. [Node/pnpm/Vite Issues](#nodepnpmvite-issues)  
    10.3. [PostgreSQL/pgvector Issues](#postgresqlpgvector-issues)  
    10.4. [Redis Issues](#redis-issues)  
    10.5. [`.env` / dotenv Issues](#env--dotenv-issues)  
    10.6. [WSL / VS Code / Interop](#wsl--vs-code--interop)  
    10.7. [Misc (Ports, Seeds, Assets)](#misc-ports-seeds-assets)  
11. [Diagnostics Commands](#diagnostics-commands)  
12. [Appendix: Full `.env` for DEV](#appendix-full-env-for-dev)

---

## Overview

Impulso Hub is a fork of Chatwoot. DEV setup consists of:

- Ruby 3.4.4 via RVM + Bundler 2.5.16  
- Node (20.x or 23.x) + pnpm  
- PostgreSQL (local) with optional **pgvector** extension (for v4+ features)  
- Redis (local)  
- Project dependencies (`bundle install`, `pnpm install`)  
- Per‑repo **`.env`** configuration  
- Database prepare & optional seed  
- Run via **Foreman** (`Procfile.dev`) or **Overmind**

---

## System Prep

### WSL and VS Code

- Install **WSL2** and **Ubuntu-24.04**.  
- In VS Code, install **Remote – WSL** extension.  
- Open the repo *from inside WSL*: `cd ~/Impulso_Hub && code .`  
- To avoid CRLF issues, set VS Code workspace `files.eol = "\n"` and Git `core.autocrlf=input`.

### Essential OS Packages

```bash
sudo apt-get update
sudo apt-get install -y \
  git curl build-essential software-properties-common \
  postgresql postgresql-contrib libpq-dev \
  redis-server imagemagick tmux
```

> If you are in WSL, enable **systemd** in `/etc/wsl.conf`:
>
> ```ini
> [interop]
> enabled = true
> appendWindowsPath = true
>
> [boot]
> systemd = true
> ```
> Then from Windows: `wsl --shutdown`, reopen Ubuntu.

---

## Language & Toolchain

### Ruby via RVM (3.4.4)

```bash
# load RVM (if already installed)
source /etc/profile.d/rvm.sh 2>/dev/null || source ~/.rvm/scripts/rvm

# install/select Ruby 3.4.4
rvm install 3.4.4
rvm use 3.4.4 --default

ruby -v
```

### Bundler (2.5.16)

```bash
gem install bundler -v 2.5.16
bundle -v
```

> Do **not** use `sudo` for `bundle`.

### Node & pnpm

Prefer Node 20.x (stable for dev). Node 23.x is fine if your toolchain aligns.

```bash
# Node 20 (via NodeSource or nvm if you prefer)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v

# pnpm via corepack
corepack enable
corepack prepare pnpm@10 --activate
pnpm -v
```

---

## Services

### PostgreSQL + pgvector

Create a dev user and DB (idempotent):

```bash
sudo systemctl enable --now postgresql

sudo -u postgres psql -c "DO $$BEGIN IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname='dev') THEN CREATE ROLE dev LOGIN SUPERUSER PASSWORD 'devpass'; END IF; END$$;"
sudo -u postgres createdb -O dev chatwoot_development 2>/dev/null || true

# test
PGPASSWORD=devpass psql -h 127.0.0.1 -U dev -d chatwoot_development -c "select current_user, current_database();"
```

Enable **pgvector** only if migrations require it:

```bash
# choose package matching your PG version (15/16)
sudo apt-get install -y postgresql-16-pgvector || sudo apt-get install -y postgresql-15-pgvector
PGPASSWORD=devpass psql -h 127.0.0.1 -U dev -d chatwoot_development -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### Redis

```bash
sudo systemctl enable --now redis-server
redis-cli ping  # expect PONG
```

---

## Project Setup

### Clone / Enter Repo

```bash
cd ~
git clone https://github.com/ImpulsoCoreIA/Impulso_Hub.git
cd ~/Impulso_Hub
```

### `.env` (Development)

Create `~/Impulso_Hub/.env` (LF endings, no BOM):

```dotenv
RAILS_ENV=development
FRONTEND_URL=http://localhost:3000
SECRET_KEY_BASE=REPLACE_ME_WITH_secure_hex_128chars

# Postgres
POSTGRES_HOST=127.0.0.1
POSTGRES_PORT=5432
POSTGRES_DATABASE=chatwoot_development
POSTGRES_USERNAME=dev
POSTGRES_PASSWORD=devpass

# Redis
REDIS_URL=redis://127.0.0.1:6379

# Email (DEV)
MAILER_SENDER_EMAIL="Impulso Hub <no-reply@localhost>"
LETTER_OPENER=true

# Useful in DEV
FORCE_SSL=false
ENABLE_ACCOUNT_SIGNUP=true
RAILS_LOG_TO_STDOUT=true
LOG_LEVEL=debug
LOG_SIZE=500
```

Generate a secure secret:

```bash
ruby -e 'require "securerandom"; puts SecureRandom.hex(64)'
```

### Install Deps

```bash
bundle install
pnpm install
# or: make burn
```

### DB Prepare & Seed

```bash
RAILS_ENV=development bundle exec rails db:chatwoot_prepare
# optional sample data:
RAILS_ENV=development bundle exec rails db:seed
```

If `.env` is not being read, force via `DATABASE_URL`:
```bash
RAILS_ENV=development \
DATABASE_URL=postgres://dev:devpass@127.0.0.1:5432/chatwoot_development \
bundle exec rails db:chatwoot_prepare
```

### Run App (Foreman / Overmind)

**Foreman (simple):**
```bash
gem install foreman
foreman start -f Procfile.dev
```

**Overmind (advanced, requires binary installed + tmux):**
```bash
sudo apt-get install -y tmux
# assuming overmind is installed in PATH:
overmind s -f Procfile.dev
# or project make target:
# make run
```

Open <http://localhost:3000>.

---

## 🚀 Start Commands (TL;DR)

Pick **one** of the flows below.

### A) One‑shot (recommended): Foreman (Procfile.dev)
```bash
# from ~/Impulso_Hub
gem install foreman
foreman start -f Procfile.dev
```
- Starts **backend** (Rails), **worker** (Sidekiq) and **Vite** (front-end) together.
- Open: <http://localhost:3000>

### B) With Overmind (if you prefer)
```bash
sudo apt-get install -y tmux
overmind s -f Procfile.dev
# or, if your Makefile has these targets:
make run        # start
make force_run  # start (ignores socket-in-use)
```
> Overmind is great for managing processes; requires overmind binary in PATH.

### C) Minimal smoke test (backend only)
```bash
RAILS_ENV=development bundle exec rails s -p 3000 -b 0.0.0.0
```
In a **second terminal**, run Vite to serve the UI:
```bash
# Either via script, if defined:
pnpm run dev
# Or directly with the binary:
pnpm vite
```
Vite prints a URL (often `http://localhost:3036/vite-dev/`). The Rails UI will still be at `http://localhost:3000`.

---

## 🧰 Make Targets (cheat sheet)

> Available targets depend on your repo’s `Makefile`. Common ones in this fork:
- `make burn` → `bundle install && pnpm install`
- `make db` → `RAILS_ENV=development bundle exec rails db:chatwoot_prepare`
- `make db_seed` → `RAILS_ENV=development bundle exec rails db:seed`
- `make run` → run app (often via Overmind) with `-f Procfile.dev`
- `make force_run` → like `run`, but ignoring stale socket

If a target returns “No rule to make target”, check your `Makefile` or use the raw commands shown above.

---

## Quickstart Checklist

1. `rvm use 3.4.4 --default` → `ruby -v` shows 3.4.4  
2. `gem install bundler -v 2.5.16` → `bundle -v` shows 2.5.16  
3. `sudo systemctl enable --now postgresql redis-server`  
4. Create `dev/devpass` + DB → `select 1;` succeeds  
5. `.env` exists in `~/Impulso_Hub` with correct values (LF)  
6. `bundle install && pnpm install`  
7. `rails db:chatwoot_prepare` (or with `DATABASE_URL` forced)  
8. `foreman start -f Procfile.dev`

---

## Default Login (DEV)

- **Email:** `john@acme.inc`  
- **Password:** `Password1!`

If missing, run: `RAILS_ENV=development bundle exec rails db:seed` or enable sign‑up (`ENABLE_ACCOUNT_SIGNUP=true`).

---

## Failure Catalog (Errors & Fixes)

### Ruby/Bundler Issues

**`bundle: not found`**  
- RVM not loaded or bundler not installed.  
- Fix:
  ```bash
  source /etc/profile.d/rvm.sh 2>/dev/null || source ~/.rvm/scripts/rvm
  rvm use 3.4.4 --default
  gem install bundler -v 2.5.16
  which bundle && bundle -v
  ```

**`Your Ruby version is 3.3.3, but your Gemfile specified 3.4.4`**  
- Use correct Ruby:
  ```bash
  rvm install 3.4.4 && rvm use 3.4.4 --default
  bundle install
  ```

**`bundler: command not found: rails`**  
- `bundle install` didn’t finish. Run it again with the right Ruby.  
- Confirm `Gem.bindir` and PATH:
  ```bash
  ruby -e 'puts Gem.bindir'
  echo $PATH
  ```

### Node/pnpm/Vite Issues

**Engine warnings (e.g., wants Node 23.x)**  
- Usually safe to ignore in dev. Prefer Node 20.x unless toolchain requires 23.x.

**Browserslist outdated**  
```bash
pnpm dlx update-browserslist-db@latest
```

**Vite CJS deprecation note**  
- Informational. No action required for dev.

### PostgreSQL/pgvector Issues

**`fe_sendauth: no password supplied`**  
- `.env` missing or unread. Ensure:
  - `.env` exists at `~/Impulso_Hub/.env`
  - Contains `POSTGRES_*` vars and `RAILS_ENV=development`
  - Has LF line endings (no CRLF) and no BOM  
- Test:
  ```bash
  set -a; source .env; set +a
  PGPASSWORD="$POSTGRES_PASSWORD" psql -h "$POSTGRES_HOST" -U "$POSTGRES_USERNAME" -d "$POSTGRES_DATABASE" -c "select 1;"
  ```
- Force:
  ```bash
  RAILS_ENV=development DATABASE_URL="postgres://dev:devpass@127.0.0.1:5432/chatwoot_development" bundle exec rails db:chatwoot_prepare
  ```

**`pgvector` extension missing**  
```bash
sudo apt-get install -y postgresql-16-pgvector || sudo apt-get install -y postgresql-15-pgvector
psql ... -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

**Postgres not running**  
```bash
sudo systemctl enable --now postgresql
ss -lntp | grep 5432
```

### Redis Issues

**Cannot connect / not running**  
```bash
sudo systemctl enable --now redis-server
redis-cli ping
```

### `.env` / dotenv Issues

**`dotenv: No such file or directory @ rb_sysopen - /home/.../.env`**  
- Ensure file exists in repo dir; normalize EOL:
  ```bash
  dos2unix .env || sed -i 's/\r$//' .env
  sed -i '1s/^\xEF\xBB\xBF//' .env
  chmod 600 .env
  ```

**Spaces in values** (e.g., `MAILER_SENDER_EMAIL`)  
- Quote them: `MAILER_SENDER_EMAIL="Impulso Hub <no-reply@localhost>"`

### WSL / VS Code / Interop

**`Exec format error` running `code .` or `.exe`**  
- Enable interop + systemd in `/etc/wsl.conf` and restart WSL (`wsl --shutdown`).  
- Install **Remote – WSL** in VS Code.

**Windows file created with CRLF**  
- Convert: `dos2unix .env` (or `sed -i 's/\r$//' .env`).

### Misc (Ports, Seeds, Assets)

**Port 3000 in use**  
```bash
ss -lntp | grep :3000
kill -9 <PID>
```

**Missing default login**  
```bash
RAILS_ENV=development bundle exec rails db:seed
```

**Assets issues**  
```bash
bundle exec rails assets:precompile
```

---

## Diagnostics Commands

```bash
# Ruby/Bundler
which ruby; ruby -v
which bundle; bundle -v
ruby -e 'puts Gem.bindir'
echo "$PATH"

# Node/pnpm
node -v; pnpm -v

# Repo & env
pwd; ls -l .env
grep -nE 'RAILS_ENV|FRONTEND_URL|POSTGRES_|REDIS_URL|SECRET_KEY_BASE' .env || true

# Postgres
sudo systemctl status postgresql --no-pager
PGPASSWORD=devpass psql -h 127.0.0.1 -U dev -d chatwoot_development -c "select 1;"
ss -lntp | grep 5432

# Redis
sudo systemctl status redis-server --no-pager
redis-cli ping

# App
RAILS_ENV=development bundle exec rails -v
```

---

## Appendix: Full `.env` for DEV

```dotenv
RAILS_ENV=development
FRONTEND_URL=http://localhost:3000
SECRET_KEY_BASE=REPLACE_ME_WITH_secure_hex_128chars

POSTGRES_HOST=127.0.0.1
POSTGRES_PORT=5432
POSTGRES_DATABASE=chatwoot_development
POSTGRES_USERNAME=dev
POSTGRES_PASSWORD=devpass

REDIS_URL=redis://127.0.0.1:6379

MAILER_SENDER_EMAIL="Impulso Hub <no-reply@localhost>"
LETTER_OPENER=true

FORCE_SSL=false
ENABLE_ACCOUNT_SIGNUP=true
RAILS_LOG_TO_STDOUT=true
LOG_LEVEL=debug
LOG_SIZE=500
```
