# FlowApprove Docs

> A full-stack, multi-tenant **document approval workflow system** that replaces email / WhatsApp / Excel tracking with a structured, auditable pipeline.

Built with **Django REST** (backend) and **React + Vite** (frontend).

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

## Main features

- **Dashboard** — stat cards (Total, My Turn, In Progress, Approved by Me, Sent Back, Completed) + filterable document table.
- **Document detail** — tabs for preview (download / open), workflow timeline, comments, full history, version list.
- **Version control** — available only after rejection; new version replaces current file, older versions are archived for comparison.
- **Page-anchored comments** — comments can reference a specific PDF page.
- **Email notifications** — SMTP-configurable; next approver (or creator) is pinged on status changes.
- **Audit trail** — every upload, approve, send-back, comment, and version is immutably logged in document history.

## Tech stack

| Layer | Stack |
|---|---|
| Backend | Python, Django, Django REST Framework, MySQL, JWT (SimpleJWT), SMTP email |
| Frontend | React 19, Vite, Axios, React Router v7, TanStack Query, react-pdf, react-dropzone, custom CSS |

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

## Running these docs locally

```bash
pip install mkdocs mkdocs-material
mkdocs serve        # http://127.0.0.1:8000
mkdocs build        # static site in ./site
```

Or just open any `.md` file in your editor — they render on GitHub as-is.
