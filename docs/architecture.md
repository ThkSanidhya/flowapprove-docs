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
                         approve              approve              approve
  upload  ──▶  step 1 ────────────▶ step 2 ────────────▶ step 3 ────────────▶ APPROVED
               │                    │                    │
               │ recall             │ reject             │ sendback
               ▼                    ▼                    ▼
           CANCELLED             REJECTED            step 1 or 2 (policy-gated)
           (creator)             │                    │
                                 │ upload-version     │ upload-version
                                 │ (creator, full     │ (current step user,
                                 │  reset to step 1)  │  partial resume)
                                 ▼                    ▼
                               step 1 PENDING      resume forward
```

- **approve** — only the user assigned to `current_step` may approve. Advances `current_step` or finalizes `APPROVED`. Wrapped in `transaction.atomic()` + `select_for_update()`.
- **reject** — only the current-step assignee. Sets `status = REJECTED` and resets every sibling approval to PENDING so the creator can upload a revised version that restarts the workflow cleanly.
- **sendback** — current-step assignee sends the document back to any previous step (subject to the workflow's `sendback_type` policy). Partial reset: approvals from the target step onward become PENDING, earlier approved steps stay APPROVED.
- **upload-version** — after rejection, the **creator** uploads a revised file; full reset to step 1. After sendback, the **current-step user** uploads; no further reset. Old file is archived as a `DocumentVersion` row.
- **recall** — the creator withdraws a PENDING document mid-flow. Status becomes `CANCELLED` and all approvals are cleared. Terminal — document can't be re-opened.

Every transition appends a `DocumentHistory` row. See [Data Model](data-model.md) for the full entity map.

## Versioning

Version control is **only available after a document is rejected**. When the creator uploads a new version:

- A new `DocumentVersion` row is written with an incremented `version_number`.
- The `Document.file` pointer switches to the new file; the old file is preserved and accessible from the version list.
- `current_step` resets to **1** and all `DocumentApproval` rows go back to `PENDING`.
- A `DocumentHistory` entry records the re-upload with its version number.

This lets reviewers compare the new submission against what they rejected without losing the audit trail.

## Email notifications

SMTP-based notifications (configured via `EMAIL_*` settings in `flowapprove_backend/settings.py`) fire on status transitions:

| Event | Recipient |
|---|---|
| Document uploaded | first-step approver |
| Step approved (not final) | next-step approver |
| Step approved (final) | document creator |
| Sent back | previous-step approver |
| Rejected | document creator |

Notification sending lives in `api/utils.py::send_email_notification` and is called from the approve / reject / sendback views. Failures are non-blocking — the workflow state still transitions even if the email can't be delivered.

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
