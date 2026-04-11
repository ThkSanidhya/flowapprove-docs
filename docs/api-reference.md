# API Reference

Base URL: `http://localhost:8000/api/` (configurable via `flowapprove_backend/urls.py`).

All endpoints except `register` and `login` require a JWT:

```
Authorization: Bearer <access_token>
```

## Auth

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/auth/register` | `{name, email, password, organization_name}` | `{token, user}` — also bootstraps an `Organization` and an `ADMIN` user |
| POST | `/auth/login` | `{email, password}` | `{token, user}` |
| GET  | `/auth/me` | — | current user profile |

## Users (admin only)

| Method | Path | Purpose |
|---|---|---|
| GET  | `/users` | list users in caller's org |
| POST | `/users` | create user (body: `{name, email, password, role}`) |
| PUT  | `/users/<id>` | update name / email / role / password |
| DELETE | `/users/<id>` | delete user (cannot delete self) |

## Workflows

| Method | Path | Purpose |
|---|---|---|
| GET  | `/workflows` | list workflows in caller's org |
| POST | `/workflows` | create; body: `{name, sendbackType?, steps: [{userId}, ...]}` — each `userId` must belong to caller's org. `sendbackType` is `PREVIOUS_ONLY` (default) or `ANY_PREVIOUS`. |
| GET  | `/workflows/<id>` | detail (admin only) |
| PUT  | `/workflows/<id>` | replace name + steps (admin only) |
| DELETE | `/workflows/<id>` | delete (admin only) |

## Documents

| Method | Path | Purpose |
|---|---|---|
| POST | `/documents/upload` | multipart: `file`, `title`, `description`, `workflowId`. Enforces **50 MB** max and MIME whitelist (`pdf`, `jpeg`, `png`, `doc`, `docx`). |
| GET  | `/documents` | list (admins: all in org; users: created-by or assigned-to) |
| GET  | `/documents/<id>` | detail — includes `timeline`, `canApprove`, `progress`, `comments`, `history`, `versions` |
| POST | `/documents/<id>/approve` | body: `{comment}` — only the current-step assignee |
| POST | `/documents/<id>/reject` | body: `{comment}` — only the current-step assignee. Sets status=`REJECTED` and resets all sibling approvals so the creator can re-upload. |
| POST | `/documents/<id>/sendback` | body: `{reason, target_step?}` — only the current-step assignee. `target_step` is optional and defaults to `current_step - 1`. Workflows with `sendback_type='PREVIOUS_ONLY'` reject non-adjacent targets. Approvals from `target_step` onward are reset to PENDING; earlier approved steps stay APPROVED (partial reset). |
| POST | `/documents/<id>/upload-version` | multipart: `file`, optional `version_note`. Archives the current file to `DocumentVersion` and replaces it. Auth: creator only (if status=`REJECTED`, full reset back to step 1) or current-step assignee (if status=`PENDING` post-sendback, no additional reset). |
| POST | `/documents/<id>/recall` | body: `{reason}` — only the creator, only while status=`PENDING`. Sets status=`CANCELLED` and clears all approvals. |

## Dashboard

| Method | Path | Purpose |
|---|---|---|
| GET | `/dashboard/stats` | `{totalDocuments, inProgress, pendingMyAction, approvedByMe, sentBack, completed}` |
| GET | `/dashboard/documents` | paginated doc list. Query: `page`, `limit`, `status`, `search` |

## Interactive docs & OpenAPI schema

The backend publishes a live OpenAPI 3 schema via **drf-spectacular**:

| Path | What |
|---|---|
| `/api/schema/` | raw OpenAPI YAML |
| `/api/docs/` | Swagger UI (interactive) |
| `/api/redoc/` | Redoc (reference-style rendering) |

Every response includes an `X-Request-ID` header — quote it in bug reports so backend logs can be correlated.

## Health check

| Method | Path | Purpose |
|---|---|---|
| GET | `/healthz` | Liveness + DB reachability. Returns `{"status":"ok"}` (200) or `{"status":"fail"}` (503). No auth. Point Docker / Kubernetes / load balancer probes here. |

## Error shape

```json
{ "error": "Human readable message" }
```

Common status codes:

| code | meaning |
|---|---|
| 400 | validation error or invalid state transition |
| 401 | missing/expired JWT |
| 403 | authenticated but not authorized (wrong role or not the step assignee) |
| 404 | resource not found **or not in your organization** |
| 413 | upload exceeds 50 MB |
