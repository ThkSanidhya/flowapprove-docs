# Development

See [Getting Started](getting-started.md) for first-time setup on Windows, macOS, or Linux (including the Docker shortcut). This page is the day-to-day command reference.

## Backend (`flowapprove-backend/`)

```bash
python manage.py migrate                 # apply migrations
python manage.py makemigrations api      # after model changes
python manage.py runserver                # dev server :8000
python manage.py createsuperuser
python manage.py test api --settings=flowapprove_backend.settings_test   # full suite (SQLite, no Postgres needed)
python manage.py test api.tests.RecallDocumentTests.test_creator_can_recall_pending_document --settings=flowapprove_backend.settings_test   # single test
python manage.py spectacular --file schema.yml    # export OpenAPI schema
```

On Windows, replace `python` with `py` if Python was installed via the python.org installer.

### Adding an endpoint

1. Add the view in `api/views.py` (function-based, decorated with `@api_view` + `@permission_classes([IsAuthenticated])`).
2. **Scope every query by `request.user.organization`** — see [Security](security.md).
3. Register the route in `api/urls.py`.
4. Add/extend a serializer in `api/serializers.py` if needed.
5. Mirror the change in `flowapprove-frontend/frontend/src/services/*Service.js`. There is no shared schema — backend and frontend must be updated together.

### Modifying the approval flow

Any change to approve / reject / sendback must keep `Document.current_step`, `DocumentApproval.status`, and `DocumentHistory` consistent **inside a single `transaction.atomic()`**. Use `select_for_update()` when reading rows you intend to mutate.

## Frontend (`flowapprove-frontend/frontend/`)

```bash
npm install
npm run dev        # Vite :5173
npm run build
npm run preview
npm run lint       # flat config in eslint.config.js
```

### Adding a service call

```js
// src/services/documentService.js
import api from './api';

export const approveDocument = (id, comment) =>
  api.post(`/documents/${id}/approve`, { comment }).then(r => r.data);
```

Then use TanStack Query in the component:

```js
const mutation = useMutation({
  mutationFn: ({ id, comment }) => approveDocument(id, comment),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['documents'] }),
});
```

Remember to invalidate both the list and detail queries after a mutation.

## Docs (`flowapprove-docs/`)

```bash
pip install mkdocs mkdocs-material
mkdocs serve       # http://127.0.0.1:8000 (use a different port if backend is also running)
mkdocs build
```

## Commit style

Conventional prefixes: `fix:`, `feat:`, `refactor:`, `docs:`, `test:`, `chore:`. Keep the subject under 72 characters and describe **why** in the body.
