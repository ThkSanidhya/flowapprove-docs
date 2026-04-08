# FlowApprove Docs

Documentation for **FlowApprove** — a multi-tenant document approval workflow platform.

- `flowapprove-backend/` — Django + DRF + SimpleJWT API
- `flowapprove-frontend/frontend/` — React 19 + Vite SPA
- `flowapprove-docs/` — this site

## Contents

- [Getting Started](docs/getting-started.md) — prerequisites, install, first run
- [Architecture](docs/architecture.md) — system overview and approval flow
- [Data Model](docs/data-model.md) — entities and relationships
- [API Reference](docs/api-reference.md) — HTTP endpoints
- [Development](docs/development.md) — day-to-day commands
- [Deployment](docs/deployment.md) — environment variables and production notes
- [Security](docs/security.md) — auth, multi-tenant scoping, upload rules

## Running these docs

The docs are plain Markdown and render as-is on GitHub. To browse them locally as a site, use **MkDocs**:

```bash
pip install mkdocs mkdocs-material
mkdocs serve        # http://127.0.0.1:8000
mkdocs build        # static site in ./site
```

Or just open any `.md` file in your editor / GitHub.
