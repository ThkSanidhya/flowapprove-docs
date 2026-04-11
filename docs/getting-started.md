# Getting Started

Three ways to run FlowApprove, in order of ease. **If you're on Windows, use Docker — it's painless.**

- [1. Docker (recommended — Windows / macOS / Linux)](#1-docker-recommended)
- [2. Native on Linux / macOS](#2-native-on-linux--macos)
- [3. Native on Windows (without Docker)](#3-native-on-windows-without-docker)
- [First login](#first-login)
- [Troubleshooting](#troubleshooting)

---

## 1. Docker (recommended)

You only need **Git** and **[Docker Desktop](https://www.docker.com/products/docker-desktop/)**. No Python, Node, or Postgres install required.

### Install Docker Desktop

- **Windows**: <https://www.docker.com/products/docker-desktop/> → download "Docker Desktop for Windows" → install. Launch Docker Desktop from the Start menu and wait for the whale icon to stop animating (it's ready when it's solid).
- **macOS**: same URL, "Docker Desktop for Mac".
- **Linux**: install `docker` + the `docker compose` plugin via your package manager.

### Clone and start

```bash
git clone --recurse-submodules https://github.com/ThkSanidhya/flowapprove.git
cd flowapprove
docker compose -f flowapprove-backend/docker-compose.yml up --build
```

> **Windows**: run these commands in **PowerShell** or **Command Prompt** — not WSL — so Docker Desktop picks them up. The commands themselves are identical.

First build takes 3–5 minutes. When you see `gunicorn` log lines, it's ready.

| Service   | URL                                  |
|-----------|--------------------------------------|
| Frontend  | <http://localhost:5173>              |
| Backend   | <http://localhost:8000/api>          |
| API docs  | <http://localhost:8000/api/docs/>    |
| Postgres  | localhost:5432 (`flowapprove` / `flowapprove`) |

### Stop everything

Press **Ctrl+C** in the terminal, then:

```bash
docker compose -f flowapprove-backend/docker-compose.yml down
```

Add `-v` to also drop the database volume (clean slate).

Jump to [First login](#first-login).

---

## 2. Native on Linux / macOS

### Prerequisites

- **Python 3.12+**
- **Node.js 20+** and npm
- **PostgreSQL 14+** running locally

### Clone the three repos (or the meta-repo)

```bash
mkdir flowapprove && cd flowapprove
git clone https://github.com/ThkSanidhya/flowapprove-backend.git
git clone https://github.com/ThkSanidhya/flowapprove-frontend.git
git clone https://github.com/ThkSanidhya/flowapprove-docs.git
```

Or clone the meta-repo with submodules:
```bash
git clone --recurse-submodules https://github.com/ThkSanidhya/flowapprove.git
cd flowapprove
```

### Backend

```bash
cd flowapprove-backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Create a Postgres user and database:
```bash
sudo -u postgres createuser --pwprompt flowapprove   # remember the password
sudo -u postgres createdb -O flowapprove flowapprove
```
(macOS Homebrew: drop the `sudo -u postgres` prefix.)

Set env vars:
```bash
cp .env.example .env
# edit .env — set DB_PASSWORD and DJANGO_SECRET_KEY
export $(grep -v '^#' .env | xargs)
```

Migrate and start:
```bash
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

### Frontend (new terminal)

```bash
cd flowapprove-frontend/frontend
npm install
cp .env.example .env     # defaults to http://localhost:8000/api — fine
npm run dev
```

Jump to [First login](#first-login).

---

## 3. Native on Windows (without Docker)

Only do this if you actively prefer not to use Docker — it's strictly more work.

### Prerequisites

- **Python 3.12+** — <https://www.python.org/downloads/> → during install, tick **"Add python.exe to PATH"**.
- **Node.js 20+** — <https://nodejs.org/> → pick the "LTS" Windows installer.
- **PostgreSQL 16** — <https://www.postgresql.org/download/windows/> → the EDB installer bundles the server and pgAdmin. Remember the `postgres` superuser password you set.
- **Git for Windows** — <https://git-scm.com/download/win>.

No Microsoft C++ Build Tools needed — `psycopg2-binary` ships pre-compiled wheels for Windows.

### Clone (PowerShell)

```powershell
cd $HOME\Desktop
git clone --recurse-submodules https://github.com/ThkSanidhya/flowapprove.git
cd flowapprove
```

### Backend

Open a new **PowerShell** window:

```powershell
cd flowapprove-backend
py -m venv venv
.\venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

If PowerShell refuses to run the activate script:
```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

Create the database. Open **pgAdmin** (installed with Postgres), right-click **Databases** → Create → Database. Or from **SQL Shell (psql)** in the Start menu:
```sql
CREATE USER flowapprove WITH PASSWORD 'your-password';
CREATE DATABASE flowapprove OWNER flowapprove;
```

Set environment variables for this PowerShell session:
```powershell
$env:DJANGO_SECRET_KEY = "dev-only-secret"
$env:DJANGO_DEBUG      = "True"
$env:DB_NAME           = "flowapprove"
$env:DB_USER           = "flowapprove"
$env:DB_PASSWORD       = "your-password"
$env:DB_HOST           = "127.0.0.1"
$env:DB_PORT           = "5432"
```

*(In **Command Prompt**, use `set DB_PASSWORD=your-password` — one per line, no `$` prefix.)*

Migrate and run:
```powershell
py manage.py migrate
py manage.py createsuperuser
py manage.py runserver
```

Leave this window open — the backend is now running.

### Frontend

Open a **second** PowerShell window:

```powershell
cd flowapprove-frontend\frontend
npm install
Copy-Item .env.example .env
npm run dev
```

Jump to [First login](#first-login).

---

## First login

1. Open **<http://localhost:5173>** in your browser.
2. Click **"Register here"** and create an account — this bootstraps an Organization and makes you an `ADMIN`.
3. **Add team members** at `/users` so there's someone to approve documents.
4. **Create a workflow** at `/workflows/create` — name it, pick an ordered list of approvers, set the send-back policy.
5. **Upload a document** at `/documents/upload` — pick the workflow. Approval records are created for every step.
6. **Log in as the first approver** in another browser (or incognito window) and walk it through approve / sendback / reject / version-upload.

---

## Troubleshooting

### Docker

**"docker: command not found"** — Docker Desktop isn't running (Windows/macOS) or the `docker` CLI isn't installed (Linux). Open Docker Desktop and wait for the whale icon to go steady.

**"port is already allocated" for 5432 / 8000 / 5173** — something else is using that port. Stop it, or edit `flowapprove-backend/docker-compose.yml` and change the `host:container` mapping, e.g. `5433:5432`.

**Build hangs on "downloading"** — your network is slow. First build is 3–5 minutes. Subsequent `up` commands are instant.

### Native backend

**`django.db.utils.OperationalError: connection to server at "localhost"`** — Postgres isn't running, or `DB_HOST` / `DB_PORT` are wrong. Check `pg_isready -h 127.0.0.1 -p 5432` (Linux/macOS) or open **Services** and confirm `postgresql-x64-16` is running (Windows).

**`FATAL: password authentication failed for user "flowapprove"`** — the `DB_PASSWORD` doesn't match what you set when creating the user. Try `PGPASSWORD=... psql -h 127.0.0.1 -U flowapprove flowapprove` to verify credentials outside Django.

**Workflows page throws "Column 'organization_id' cannot be null"** — you're logged in as a superuser created via `createsuperuser`, which doesn't set an organization. Register via the frontend instead, or assign the user an Organization in the Django admin.

### Native frontend

**CORS errors in the browser console** — the backend isn't running at `VITE_API_URL`. Visit <http://localhost:8000/api/healthz> in a browser; if it doesn't show `{"status":"ok"}`, fix the backend first.

**JWT 401 on every request** — your access token expired (60 min in dev, 15 in prod). Log out and back in.

**"Port 5173 already in use"** — another Vite instance is running. Either kill it or run `npm run dev -- --port 5174`.

### Everything else

Ask in the project repo's **Issues** tab.
