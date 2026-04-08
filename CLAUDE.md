# CLAUDE.md — flowapprove-docs

Guidance for Claude Code when working in this repository.

## What this is

The Markdown documentation site for **FlowApprove**, a multi-tenant document approval workflow system. Sibling repos: `flowapprove-backend/` (Django API) and `flowapprove-frontend/` (React SPA).

## Structure

```
flowapprove-docs/
├── README.md          # landing page + product overview
├── mkdocs.yml         # MkDocs Material config
└── docs/
    ├── getting-started.md
    ├── architecture.md
    ├── data-model.md
    ├── api-reference.md
    ├── development.md
    ├── deployment.md
    └── security.md
```

## Commands

```bash
pip install mkdocs mkdocs-material
mkdocs serve        # http://127.0.0.1:8000
mkdocs build        # static site → ./site (gitignored)
```

All pages are plain Markdown and also render as-is on GitHub.

## Golden rules

1. **Keep docs in sync with code.** When the backend or frontend changes an API, a model, an env var, or the approval flow, the corresponding page here must be updated in the same pass.
2. **Navigation is declared in `mkdocs.yml`.** Any new page must be added to the `nav:` tree or it won't appear in the built site.
3. **Use admonitions for warnings** (`!!! warning`, `!!! note`) — MkDocs Material renders them nicely. The `admonition` and `pymdownx.superfences` extensions are already enabled.
4. **Link with relative paths** (`docs/foo.md` from the root, `foo.md` from within `docs/`) so links work both on GitHub and in the built site.
5. **Don't paste secrets** into examples. Use placeholders like `change-me` or `<your-secret>`.

## Content ownership

| Page | Source of truth |
|---|---|
| `getting-started.md` | manual — update when setup steps change |
| `architecture.md` | code (`api/views.py`, `api/models.py`) |
| `data-model.md` | `api/models.py` |
| `api-reference.md` | `api/urls.py` + `api/views.py` |
| `development.md` | `package.json`, `manage.py` commands |
| `deployment.md` | `settings.py` env vars |
| `security.md` | the full app — keep in sync with any auth / scoping change |

When in doubt, read the referenced source file before editing the doc.
