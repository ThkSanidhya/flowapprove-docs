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

For `recall` and `upload-version` (when status is `REJECTED` or `CANCELLED`), the caller must be the document's `created_by`.

For `reassign`, the caller must be `ADMIN` *and* both the document and the new approver must belong to the caller's organization. The step-swap is the only admin override of the step-assignee rule and is always audited via a `REASSIGNED` `DocumentHistory` entry.

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

## Production security headers

When `DEBUG=False`, the backend automatically enables:

- `SECURE_SSL_REDIRECT` (override with `SECURE_SSL_REDIRECT=False` if TLS is terminated upstream)
- `SESSION_COOKIE_SECURE` + `CSRF_COOKIE_SECURE`
- HSTS: `SECURE_HSTS_SECONDS=31536000`, `SECURE_HSTS_INCLUDE_SUBDOMAINS`, `SECURE_HSTS_PRELOAD`
- `SECURE_PROXY_SSL_HEADER=('HTTP_X_FORWARDED_PROTO', 'https')`

Always on (dev and prod):

- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: same-origin`

## Observability

Every request passes through `api.middleware.RequestIDMiddleware`, which:

1. Reads an inbound `X-Request-ID` header or generates a 16-char UUID.
2. Attaches the ID to `request.request_id`.
3. Echoes it back on the response as `X-Request-ID`.
4. Logs the method + path + status code under logger `request` with the ID in `extra`.

This lets you grep backend logs by an ID the frontend (or your friend filing a bug report) sees.

## Known gaps

| Gap | Mitigation |
|---|---|
| JWT stored in `localStorage` (XSS-exfiltratable) | Consider httpOnly cookies + CSRF tokens |
| No rate limiting on `/auth/login` | Add `django-ratelimit` or an upstream proxy rule |
| No audit of admin actions beyond `DocumentHistory` | Extend the history table or ship a separate audit log |
| No virus scanning on uploads | Add ClamAV sidecar or VirusTotal integration |

## Reporting vulnerabilities

Email the maintainers privately — do not file a public issue for security bugs.
