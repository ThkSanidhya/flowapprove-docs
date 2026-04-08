# Changelog

All notable changes to FlowApprove. Entries are grouped by repo.

## Unreleased

### Backend — security & hardening

- **CRITICAL**: fixed multi-tenant IDOR on `approve`/`reject`/`sendback` — all queries now scope by `request.user.organization`.
- Workflow step user assignment validated against requester's organization; workflow create wrapped in `transaction.atomic()` so invalid steps don't leave orphan rows.
- `upload_document` now uses the validated `Workflow` instance instead of the raw `workflowId` from the request.
- File upload: 50 MB size cap + MIME whitelist (`pdf`, `jpeg`, `png`, `doc`, `docx`); randomized on-disk filenames.
- Approval flow wrapped in `select_for_update()` to prevent concurrent double-advances.
- `datetime.now()` → `timezone.now()` under `USE_TZ=True`.

### Backend — production readiness

- All secrets env-driven: `DJANGO_SECRET_KEY`, `DB_*`, `EMAIL_*`.
- Security headers auto-enabled in prod: HSTS, secure cookies, `SECURE_PROXY_SSL_HEADER`, `X-Frame-Options=DENY`, content-type nosniff, referrer policy.
- CORS tightened: `CORS_ALLOW_ALL_ORIGINS` is dev-only; prod reads `CORS_ORIGINS`.
- JWT access lifetime shortened to 15 min in prod (env-overridable).
- Email backend auto-selects console in dev, SMTP in prod.
- Structured logging with request-ID correlation middleware.
- `/healthz` endpoint for Docker / load balancer probes.
- `/api/schema/`, `/api/docs/` (Swagger UI), `/api/redoc/` via drf-spectacular.
- DB indexes on hot query paths (`Document.organization+status`, etc.) and `DocumentApproval.unique(document, step_order)`.
- N+1 fixes in `dashboard_stats`, `get_document_detail`, `get_user_documents`, `get_documents`.
- `Dockerfile` (gunicorn), `docker-compose.yml` (db + backend + frontend), `.env.example`.

### Backend — tests

- 13 regression tests covering multi-tenant isolation, non-assignee authorization, workflow step assignment, upload validation (cross-org, oversized, bad MIME).
- `settings_test.py` runs tests on SQLite in-memory — no MySQL required.
- GitHub Actions CI: `manage.py check` + test suite + OpenAPI schema validation.

### Frontend

- Top-level `ErrorBoundary` component so a broken child (e.g. `react-pdf` on a malformed file) no longer blanks the app.
- `services/api.js` cleaned up: removed debug `console.log` that leaked URLs and token presence; 401 redirect guarded against infinite loops on `/login`; uses `window.location.replace`.
- `.env.example` documents `VITE_API_URL`.
- Multi-stage Dockerfile (node build → nginx serve) with SPA fallback, hashed-asset caching, security headers.
- GitHub Actions CI: lint + build.

### Docs

- Full MkDocs site: getting started, architecture, data model, API reference, development, deployment, security.
- CLAUDE.md in every repo with golden rules and content-ownership map.
- Meta-repo (`flowapprove`) with git submodules for one-command clone.
