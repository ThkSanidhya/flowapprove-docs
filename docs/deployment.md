# Deployment

!!! warning
    The default `settings.py` ships with `DEBUG=True` and `CORS_ALLOW_ALL_ORIGINS=True`. **Do not deploy as-is.** Override via environment variables below and tighten CORS before exposing the service.

## Required environment variables (backend)

| Variable | Purpose | Example |
|---|---|---|
| `DJANGO_SECRET_KEY` | Django signing key | long random string |
| `DJANGO_DEBUG` | `True` / `False` | `False` in prod |
| `DJANGO_ALLOWED_HOSTS` | comma-separated | `api.example.com,example.com` |
| `DB_NAME` | MySQL database | `flowapprove` |
| `DB_USER` | MySQL user | `flowapprove` |
| `DB_PASSWORD` | MySQL password | — |
| `DB_HOST` | MySQL host | `db.internal` |
| `DB_PORT` | MySQL port | `3306` |

SMTP is still hardcoded in `settings.py` — move `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` to env vars before production use.

## Production checklist

- [ ] `DEBUG=False` and a real `SECRET_KEY`
- [ ] Lock `ALLOWED_HOSTS` and `CORS_ALLOWED_ORIGINS` (replace `CORS_ALLOW_ALL_ORIGINS`)
- [ ] Serve behind HTTPS; set `SECURE_PROXY_SSL_HEADER`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`
- [ ] Run under `gunicorn` / `uwsgi`, not `runserver`
- [ ] Collect static: `python manage.py collectstatic`
- [ ] Reverse-proxy `MEDIA_URL` through nginx or move storage to S3
- [ ] Rotate JWT signing key; shorten `ACCESS_TOKEN_LIFETIME` (currently 1 day)
- [ ] Back up MySQL + the media directory
- [ ] Run `python manage.py migrate --plan` on a staging copy before prod

## Frontend build

```bash
cd flowapprove-frontend/frontend
npm ci
npm run build                # output in dist/
```

Point your static host (nginx, Cloudflare Pages, S3+CloudFront) at `dist/`. Set the API base URL via a build-time env var (see `src/services/api.js`).

## Sample `.env`

```bash
DJANGO_SECRET_KEY=replace-with-64-random-chars
DJANGO_DEBUG=False
DJANGO_ALLOWED_HOSTS=api.example.com

DB_NAME=flowapprove
DB_USER=flowapprove
DB_PASSWORD=change-me
DB_HOST=db.internal
DB_PORT=3306
```

Load it with `direnv`, `systemd` `EnvironmentFile=`, or your container runtime.
