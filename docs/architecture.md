# Architecture

## High-level

```
┌────────────────────┐      HTTPS/JSON       ┌───────────────────────┐
│  React 19 SPA      │ ───────────────────▶  │  Django + DRF API     │
│  (Vite, RR v7)     │    JWT (Bearer)       │  SimpleJWT auth       │
│  TanStack Query    │ ◀───────────────────  │  Single app: api/     │
└────────────────────┘                       └──────────┬────────────┘
                                                        │
                                                        ▼
                                              ┌───────────────────┐
                                              │   MySQL 8         │
                                              │   + MEDIA_ROOT/   │
                                              └───────────────────┘
```

## Key decisions

- **Single Django app (`api/`)** — all models, views, serializers, and URLs live in one app. Business logic is inlined in function-based DRF views.
- **Custom user model (`api.User`)** — `USERNAME_FIELD = 'email'`, carries a `role` (`ADMIN`/`USER`) and an `Organization` FK.
- **Multi-tenant by `Organization`** — every domain row (`Workflow`, `Document`, `User`) is scoped by `organization`. **Every query must filter by `request.user.organization`.** See [Security](security.md).
- **JWT (SimpleJWT)** — stateless auth; the frontend stores the access token and attaches it via `Authorization: Bearer …` in `services/api.js`.
- **File storage on local disk** — uploads go to `MEDIA_ROOT/documents/` and `MEDIA_ROOT/versions/`. Swap for S3 by changing `DEFAULT_FILE_STORAGE`.

## The approval flow

A `Workflow` is an ordered sequence of `WorkflowStep`s, each assigned to a specific `User`. A `Document` references one workflow and tracks:

- `status`: `PENDING` | `APPROVED` | `REJECTED`
- `current_step`: 1-indexed pointer into the workflow's steps

On upload, a `DocumentApproval` row is created for every step (all `PENDING`). From there:

```
     upload                approve                 approve                 approve
PENDING  ──▶  step 1 PENDING  ──▶  step 2 PENDING  ──▶  step 3 PENDING  ──▶  APPROVED
                    │                  │                  │
                    │ reject           │ reject           │ reject
                    ▼                  ▼                  ▼
                REJECTED           REJECTED           REJECTED

                    │ sendback (step N → N-1, resets prev approval)
                    ▼
             step N-1 PENDING
```

- **approve** — only the user assigned to `current_step` may approve. Advances `current_step` or finalizes `APPROVED`. Wrapped in `transaction.atomic()` + `select_for_update()` to serialize concurrent requests.
- **reject** — only the current-step assignee. Sets `status = REJECTED`.
- **sendback** — bounces one step back, marks the current approval `REJECTED`, resets the previous step to `PENDING`, and logs the reason.

Every transition appends a `DocumentHistory` row. See [Data Model](data-model.md) for the full entity map.

## Frontend structure

Components are grouped by feature:

```
src/
├── context/AuthContext.jsx      # login/logout, current user
├── services/
│   ├── api.js                   # axios instance + JWT header
│   ├── authService.js
│   ├── documentService.js
│   └── workflowService.js
├── components/
│   ├── Auth/
│   ├── Dashboard/
│   ├── Documents/               # react-pdf viewer, react-dropzone upload
│   ├── Layout/
│   ├── Users/
│   └── Workflows/
└── App.jsx / main.jsx
```

TanStack Query owns server state (lists, detail, dashboard stats). When adding endpoints, extend the matching `services/*Service.js` rather than calling axios directly from components.
