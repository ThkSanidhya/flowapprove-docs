# Deployment

!!! warning
    The default `settings.py` ships with `DEBUG=True` and `CORS_ALLOW_ALL_ORIGINS=True`. **Do not deploy as-is.** Override via environment variables below and tighten CORS before exposing the service.

## Required environment variables (backend)

See `flowapprove-backend/.env.example` for the canonical list.

| Variable | Purpose | Example |
|---|---|---|
| `DJANGO_SECRET_KEY` | Django signing key | long random string |
| `DJANGO_DEBUG` | `True` / `False` | `False` in prod |
| `DJANGO_ALLOWED_HOSTS` | comma-separated | `api.example.com,example.com` |
| `DB_NAME` / `DB_USER` / `DB_PASSWORD` / `DB_HOST` / `DB_PORT` | MySQL connection | — |
| `CORS_ORIGINS` | comma-separated allowed origins (prod only) | `https://app.example.com` |
| `CSRF_TRUSTED_ORIGINS` | comma-separated (prod only) | `https://app.example.com` |
| `JWT_ACCESS_MINUTES` | access token lifetime | `15` (prod default) / `60` (dev default) |
| `SECURE_SSL_REDIRECT` | force HTTPS at Django level | `True` unless TLS is terminated upstream |
| `EMAIL_BACKEND` | override auto-selected backend | `django.core.mail.backends.smtp.EmailBackend` |
| `EMAIL_HOST` / `EMAIL_PORT` / `EMAIL_USE_TLS` | SMTP server | — |
| `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` | SMTP creds | — |
| `DEFAULT_FROM_EMAIL` | From header | `noreply@flowapprove.local` |

In **dev** (`DJANGO_DEBUG=True`) the email backend defaults to the console backend so messages print to stdout. In **prod** it defaults to SMTP.

## Production checklist

- [x] `DEBUG=False` and a real `SECRET_KEY` (env-driven)
- [x] HSTS + secure cookies + `SECURE_PROXY_SSL_HEADER` (auto-enabled when `DEBUG=False`)
- [x] `X-Frame-Options=DENY`, content-type nosniff, referrer policy (always on)
- [x] `CORS_ALLOW_ALL_ORIGINS` is dev-only; prod reads `CORS_ORIGINS`
- [x] JWT access lifetime shortened to 15 min in prod
- [x] `/healthz` endpoint for liveness probes
- [x] Structured request logs with correlation ID
- [x] DB indexes on hot paths; `DocumentApproval` unique(document, step_order)
- [ ] Lock `ALLOWED_HOSTS` and `CORS_ORIGINS` to actual domains
- [ ] Run under `gunicorn` (the Dockerfile already does this)
- [ ] Collect static: `python manage.py collectstatic`
- [ ] Reverse-proxy `MEDIA_URL` through nginx or move storage to S3
- [ ] Back up MySQL + the media directory
- [ ] Run `python manage.py migrate --plan` on a staging copy before prod
- [ ] Wire Sentry / error tracking
- [ ] Rate-limit `/api/auth/login`

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
