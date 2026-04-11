# FlowApprove Docs

> A full-stack, multi-tenant **document approval workflow system** that replaces email / WhatsApp / Excel tracking with a structured, auditable pipeline.

Built with **Django REST** (backend) and **React + Vite** (frontend).

**Running the app?** Start at the [meta-repo README](https://github.com/ThkSanidhya/flowapprove) — the easiest path is `docker compose -f flowapprove-backend/docker-compose.yml up --build`. Platform-specific instructions for Windows, macOS, and Linux are in [Getting Started](docs/getting-started.md).

## What it is

FlowApprove turns "please approve this file" chaos into an explicit, ordered workflow. Every document flows through a named sequence of approvers, every action is logged, and nothing falls through the cracks.

## Key concepts

- **Organizations** — each sign-up creates a separate company workspace (multi-tenant).
- **Users** — two roles: **Admin** (full control) and **User** (upload documents, approve only steps assigned to them).
- **Workflows** — admin-defined named sequence of steps, each bound to a specific user.
- **Documents** — any file (PDF, image, doc) uploaded by a user, optionally linked to a workflow.

## Typical flow

1. **Admin** creates a workflow, e.g. *Step 1 → User A*, *Step 2 → User B*.
2. **User** uploads a document and picks the workflow → system creates one approval record per step.
3. First approver sees **"My Turn"** on their dashboard. They can:
   - **Approve** → advances to next step.
   - **Send back** (requires reason) → returns to the **previous** step, not to the creator.
   - **Comment** (optionally anchored to a PDF page).
4. Repeats until the final step → document becomes **Approved**.
5. If **rejected** at any step, the creator can upload a **new version**. This resets the workflow to step 1 and keeps old versions for comparison.

## Typical flow (updated)

1. **Admin** creates a workflow, e.g. *Step 1 → Alice*, *Step 2 → Bob*, *Step 3 → Carol*. Admin picks a **send-back policy** (previous step only, or any previous step).
2. **User** uploads a document and picks the workflow → system creates one approval record per step.
3. First approver sees **"My Turn"** on their dashboard. They can:
   - **Approve** → advances to next step.
   - **Send back** — to any previous step (if the workflow allows). Partial reset: only approvals from the target step onward become PENDING. The user sent back to can then **upload a new version** and re-approve.
   - **Reject** — terminal. The creator can upload a new version to restart the workflow from step 1 (full reset).
   - **Comment** (optionally anchored to a PDF page).
4. **Creator** can **Recall** a pending document mid-flow → status becomes `CANCELLED`.
5. All old versions are archived under the Versions tab.

## Main features

- **Multi-tenant** — every organization's data is isolated; users can only see their own org's documents, workflows, and users.
- **Inline document viewer** — PDFs, images, and `.docx` files render directly in the browser (no download needed).
- **Dashboard** — stat cards (Total, My Turn, In Progress, Approved by Me, Sent Back, Completed, Cancelled) + filterable document table.
- **Document detail** — tabs for inline preview, workflow timeline, activity history, comments, version list.
- **Version control** — after sendback or rejection; archives the previous file, resets the workflow appropriately (partial for sendback, full for reject).
- **Configurable sendback policy** — admins choose whether approvers can skip steps.
- **Creator recall** — withdraw a pending document mid-flow.
- **Page-anchored comments** — comments can reference a specific PDF page.
- **Email notifications** — SMTP-configurable; next approver (or creator) is pinged on status changes.
- **Audit trail** — every upload, approve, send-back, reject, recall, version upload, and comment is logged in `DocumentHistory`.
- **OpenAPI schema** — live Swagger UI at `/api/docs/`, Redoc at `/api/redoc/`.

## Tech stack

| Layer    | Stack |
|----------|-------|
| Backend  | Python 3.12, Django 5, Django REST Framework, SimpleJWT, PostgreSQL 16, drf-spectacular, gunicorn |
| Frontend | React 19, Vite, Axios, React Router v7, TanStack Query, react-pdf, docx-preview, react-dropzone |
| Docs     | MkDocs + Material theme |
| Infra    | Docker Compose, GitHub Actions CI |

## Repos

- `flowapprove-backend/` — Django + DRF API
- `flowapprove-frontend/frontend/` — React 19 SPA
- `flowapprove-docs/` — this documentation

## Contents

- [Getting Started](docs/getting-started.md) — prerequisites, install, first run
- [Architecture](docs/architecture.md) — system overview and approval flow
- [Data Model](docs/data-model.md) — entities and relationships
- [API Reference](docs/api-reference.md) — HTTP endpoints
- [Development](docs/development.md) — day-to-day commands
- [Deployment](docs/deployment.md) — environment variables and production notes
- [Security](docs/security.md) — auth, multi-tenant scoping, upload rules
- [Changelog](docs/changelog.md) — notable changes across all three repos

## Running these docs locally

```bash
pip install mkdocs mkdocs-material
mkdocs serve        # http://127.0.0.1:8000
mkdocs build        # static site in ./site
```

Or just open any `.md` file in your editor — they render on GitHub as-is.
