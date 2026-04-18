# Changelog

All notable changes to FlowApprove. Entries are grouped by repo.

## Unreleased

### Core workflow improvements — safety, cancelled recovery, admin reassignment

Backend (`api/views.py`, `api/urls.py`):

- **Workflow edit/delete guard**: `PUT`/`DELETE /workflows/<id>` now returns **400** when any `PENDING` `Document` still references the workflow. Prevents rug-pulling live approval chains.
- **Cancelled → version recovery**: `upload-version` now also accepts documents with status `CANCELLED` (not only `REJECTED`). Creator uploads a new file → status resets to `PENDING`, `current_step` back to 1, prior approvals wiped. Closes the dead-end after a self-recall.
- **Admin step reassignment** (new endpoint): `POST /documents/<id>/reassign/` with `{stepOrder, newUserId}` — admin only, scoped to the caller's organization. Swaps the `DocumentApproval.user` on a single step without moving the workflow pointer, and appends a `REASSIGNED` `DocumentHistory` row. Useful when an approver is OOO.

Frontend (`flowapprove-frontend/frontend/`):

- **Admin "Reassign Step" card + modal** on the document detail page (`EnhancedDocumentDetail.jsx`) — visible only for `ADMIN` users while the document is `PENDING`. Modal lists pending steps with their current approver and an org-user dropdown (lazy-loaded from `userService.getAll()`).
- **Workflow list delete toast** now surfaces the backend "currently being used by N active document(s)" message instead of a generic failure.
- **Version-upload panel copy** now covers the `CANCELLED` case ("You recalled this document. Upload a revised file to restart the workflow from step 1.").
- New service method `documentService.reassign(id, stepOrder, newUserId)`.

### Switched from MySQL to PostgreSQL

- Backend database is now **PostgreSQL 16**. Switched for simpler free deployment (Render, Railway, Neon all offer free Postgres) and to avoid the Windows `mysqlclient` C++ build headaches.
- `requirements.txt`: `mysqlclient` → `psycopg2-binary` (pre-compiled wheels on all platforms, no system deps).
- `settings.py`: engine `django.db.backends.postgresql`, default port 5432. Added a `DATABASE_URL` parser so hosting providers (Render/Railway/Heroku/Fly.io/Neon) can inject a single URL and have it override the individual `DB_*` fields.
- `docker-compose.yml`: `mysql:8.0` → `postgres:16-alpine`, `pg_isready` healthcheck, `POSTGRES_*` env vars.
- `Dockerfile`: dropped `default-libmysqlclient-dev` + `build-essential` (no compilation needed); added `libpq5` for psycopg2 runtime; entrypoint now also runs `collectstatic` and honors the `PORT` env var so Render/Railway can set it.
- Django migrations are db-agnostic and applied cleanly to a fresh Postgres instance — no data loss for anyone running a dev database, just blow away the old MySQL DB and run `migrate`.
- All 31 regression tests still pass.


### Backend — approval workflow features (enterprise parity)

- **Version upload** (`POST /documents/<id>/upload-version`): revised file upload after sendback (current-step user, partial resume) or rejection (creator only, full reset to step 1). Archives old file as `DocumentVersion`. Same 50 MB + MIME whitelist as initial upload.
- **Configurable sendback targets**: new `Workflow.sendback_type` field (`PREVIOUS_ONLY` default, `ANY_PREVIOUS` alternative). `sendback` endpoint accepts optional `target_step`; partial reset clears only approvals from target step onward.
- **Creator recall** (`POST /documents/<id>/recall`): creator can withdraw a PENDING document, setting status to new `CANCELLED` value.
- `reject_document` now runs under `transaction.atomic()` + `select_for_update()` and resets sibling approvals so a version upload after rejection restarts cleanly.
- `get_document_detail` exposes new flags: `canRecall`, `canUploadVersion`, `sendbackType`.
- `DocumentVersion` gained a `file_type` column (previously missing).
- Migration `0003` adds all the above schema changes.
- **18 new regression tests** — 31 total, all passing.

### Frontend — new UI surfaces

- Version upload dropzone in the Versions tab, shown when `canUploadVersion` is true (copy adapts to rejected vs sent-back state).
- Recall card + confirmation modal shown when `canRecall` is true.
- Send-back modal gains a step selector when the workflow is `ANY_PREVIOUS`.
- Workflow form has a new send-back policy dropdown.
- `CANCELLED` status rendered in grey.


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
- `settings_test.py` runs tests on SQLite in-memory — no Postgres required.
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
