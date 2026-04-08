# Security

## Auth

- JWT via `rest_framework_simplejwt`. Access token lifetime: 1 day; refresh: 7 days.
- Frontend stores the access token and attaches `Authorization: Bearer …` in `services/api.js`.
- `register` bootstraps a new `Organization` plus an `ADMIN` user. All further users are created by admins inside their org.

## Multi-tenant isolation — the golden rule

Every read and every write MUST be scoped by `request.user.organization`:

```python
Document.objects.get(id=id, organization=request.user.organization)
```

Forgetting this filter is an **IDOR** (Insecure Direct Object Reference): a user in Org A can approve/reject documents in Org B by guessing IDs. This bug existed in `approve_document`, `reject_document`, and `send_back_document` and was patched in the `fix: critical multi-tenant and security hardening` commit.

Cross-checks that must also hold:

- Workflow step assignment: the assignee's `organization` must equal the requester's.
- Document upload: the referenced `workflowId` must belong to the requester's organization.
- Role gate: admin-only endpoints explicitly check `request.user.role == 'ADMIN'`.

## Approval authorization

For `approve` / `reject` / `sendback`, the caller must be the user tied to the document's `current_step` `DocumentApproval`. The approve flow wraps its check+mutation in `transaction.atomic()` + `select_for_update()` to prevent concurrent double-advances.

## File upload

- Max size: **50 MB** (returns 413 on overflow).
- MIME whitelist: `application/pdf`, `image/jpeg`, `image/png`, `application/msword`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`.
- Stored filename is randomized with `uuid.uuid4().hex` — the original filename is kept only in `file_name` for display.
- Path traversal is avoided by never trusting the client-supplied filename for the on-disk path.

**TODO**: virus scanning (ClamAV / VirusTotal) is not implemented.

## Secrets

All sensitive config is env-driven — see [Deployment](deployment.md):

- `DJANGO_SECRET_KEY`
- `DB_PASSWORD`
- SMTP credentials (still hardcoded; migrate before production)

## Known gaps

| Gap | Mitigation |
|---|---|
| JWT stored in `localStorage` (XSS-exfiltratable) | Consider httpOnly cookies + CSRF tokens |
| `CORS_ALLOW_ALL_ORIGINS=True` in default settings | Replace with explicit allowlist in prod |
| No rate limiting on `/auth/login` | Add `django-ratelimit` or an upstream proxy rule |
| No audit of admin actions beyond `DocumentHistory` | Extend the history table or ship a separate audit log |
| SMTP creds hardcoded in `settings.py` | Move to env vars |

## Reporting vulnerabilities

Email the maintainers privately — do not file a public issue for security bugs.
