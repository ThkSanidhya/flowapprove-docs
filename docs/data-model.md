# Data Model

All entities live in `flowapprove-backend/api/models.py`.

## Entity diagram

```
Organization ──┬──< User
               ├──< Workflow ──< WorkflowStep >── User
               └──< Document ──┬── Workflow (FK)
                               ├──< DocumentApproval >── User
                               ├──< DocumentComment  >── User
                               ├──< DocumentHistory  >── User
                               └──< DocumentVersion  >── User
```

## Tables

### Organization
| field | type | notes |
|---|---|---|
| `id` | PK | |
| `name` | varchar(255) | |
| `created_at` | datetime | auto |

### User (extends `AbstractUser`)
| field | type | notes |
|---|---|---|
| `email` | unique | `USERNAME_FIELD` |
| `name` | varchar(255) | |
| `role` | `ADMIN` \| `USER` | |
| `organization` | FK → Organization | nullable (during registration bootstrap) |

### Workflow
| field | type | notes |
|---|---|---|
| `name` | varchar(255) | |
| `organization` | FK → Organization | tenant scope |

### WorkflowStep
| field | type | notes |
|---|---|---|
| `workflow` | FK → Workflow | `related_name='steps'` |
| `order` | int | 1-indexed, ordered ascending |
| `user` | FK → User | must belong to same org |

### Document
| field | type | notes |
|---|---|---|
| `title` / `description` | text | |
| `file` / `file_name` / `file_url` / `file_type` / `file_size` | | stored under `MEDIA_ROOT/documents/` |
| `status` | `PENDING` \| `APPROVED` \| `REJECTED` | |
| `current_step` | int | 1-indexed pointer into workflow steps |
| `organization` | FK → Organization | tenant scope |
| `workflow` | FK → Workflow | nullable (SET_NULL on delete) |
| `created_by` | FK → User | |

### DocumentApproval
One row per step per document.

| field | type | notes |
|---|---|---|
| `document` | FK → Document | `related_name='approvals'` |
| `step_order` | int | matches `WorkflowStep.order` |
| `user` | FK → User | the assignee for that step |
| `status` | `PENDING` \| `APPROVED` \| `REJECTED` | |
| `comment` / `approved_at` | | |

### DocumentComment
Discussion thread, optionally anchored to a PDF page via `page_number`.

### DocumentHistory
Append-only audit log. Every approve/reject/sendback writes a row with `action` and `comment`.

### DocumentVersion
Re-uploads. `version_number` increments per document.

## Invariants

These must be preserved by any code touching the approval flow:

1. **Tenant isolation** — every query hits `.filter(organization=request.user.organization)`.
2. **`Document.current_step`, `DocumentApproval.status`, and `DocumentHistory`** are kept in sync inside a single `transaction.atomic()`.
3. **Only the user at `current_step`** may approve / reject / sendback.
4. On sendback, the previous step's approval is reset to `PENDING` (cleared `approved_at`, empty `comment`).
