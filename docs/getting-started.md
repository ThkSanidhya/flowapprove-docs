# Getting Started

## Prerequisites

- **Python 3.11+**
- **Node.js 20+** and npm
- **MySQL 8+** running locally (or reachable via env vars)

## 1. Clone the sibling repos

```bash
mkdir flowapprove && cd flowapprove
git clone <backend-repo-url>   flowapprove-backend
git clone <frontend-repo-url>  flowapprove-frontend
git clone <docs-repo-url>      flowapprove-docs
```

## 2. Backend setup

```bash
cd flowapprove-backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Create a MySQL database and export env vars (see [Deployment](deployment.md) for the full list):

```bash
export DB_NAME=flowapprove
export DB_USER=root
export DB_PASSWORD=yourpassword
export DJANGO_SECRET_KEY="change-me"
```

Run migrations and start the dev server:

```bash
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver    # http://127.0.0.1:8000
```

## 3. Frontend setup

```bash
cd ../flowapprove-frontend/frontend
npm install
npm run dev                   # http://127.0.0.1:5173
```

The Vite dev server talks to the backend API via the base URL configured in `src/services/api.js`.

## 4. First login

1. Open the frontend in your browser.
2. **Register** — this creates an `Organization` plus an `ADMIN` user.
3. Create additional users, build a workflow, upload a document, and walk it through the approval chain.

## Troubleshooting

- **`django.db.utils.OperationalError`** — MySQL not reachable or credentials wrong; check env vars.
- **CORS errors in browser** — backend has `CORS_ALLOW_ALL_ORIGINS=True` in dev; confirm `DEBUG=True`.
- **JWT 401 on every request** — token expired (1 day lifetime); log in again.
