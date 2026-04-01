# Django Admin as System UI — Model & Action Plan

This document lists every model that needs to be registered in Django admin and the
actions/configuration required to use admin as the primary UI for operating VaultShare.

---

## 1. accounts — `User`

**Admin class:** `UserAdmin` (extend `django.contrib.auth.admin.UserAdmin`)

### Fields to display

| List display | Detail (fieldsets) |
|---|---|
| `email`, `username`, `is_active`, `is_staff`, `storage_quota_bytes`, `storage_used_bytes`, `created_at` | **Account:** email, username, is_active, is_staff, is_superuser · **Storage:** storage_quota_bytes, storage_used_bytes (read-only) · **Dates:** created_at, updated_at, last_login (all read-only) |

### Filters & search
- **Search:** email, username
- **Filters:** is_active, is_staff

### Actions
| Action | Description | Service layer hook |
|---|---|---|
| Create user | Form with email + password | Django's built-in `UserAdmin` creation form (adapt for email-based login) |
| Edit user | Change email, username, is_active, quota | Standard save |
| Reset password | Set new password via admin | Django's built-in password change form |
| Deactivate | Toggle `is_active` | Standard save |

### Notes
- `storage_used_bytes` should be **read-only** — it is derived from file versions.
- Adapt `UserAdmin` add/change forms since `USERNAME_FIELD` is `email`.

---

## 2. storage — `Folder`

**Admin class:** `FolderAdmin`

### Fields to display

| List display | Detail |
|---|---|
| `name`, `owner`, `parent`, `deleted_at`, `created_at` | name, owner, parent, deleted_at |

### Filters & search
- **Search:** name, owner email
- **Filters:** deleted_at (null = active), owner

### Actions
| Action | Description | Service layer hook |
|---|---|---|
| Create folder | Pick owner + parent + name | Standard save |
| Edit folder | Rename, move (change parent) | Standard save |
| Soft-delete | Set `deleted_at` to now | Standard save (or custom action) |
| Restore | Clear `deleted_at` | Standard save (or custom action) |

### Notes
- Consider a tree-style display or breadcrumb in the detail view (`parent` chain).

---

## 3. storage — `File`

**Admin class:** `FileAdmin`

### Fields to display

| List display | Detail |
|---|---|
| `name`, `owner`, `folder`, `mime_type`, `current_version`, `deleted_at`, `created_at` | name, owner, folder, mime_type, current_version (read-only link), deleted_at |

### Inline
- `FileVersionInline` — read-only table of all versions for this file.

### Filters & search
- **Search:** name, owner email
- **Filters:** mime_type, deleted_at (null = active), folder

### Actions
| Action | Description | Service layer hook |
|---|---|---|
| Upload new file | Custom form: pick owner, folder, attach file | `FileService.upload()` — creates File + FileVersion, writes to storage backend, logs AuditLog |
| Upload new version | Custom action on existing file: attach file | `FileService.upload()` (existing file path) |
| Soft-delete | Set `deleted_at` to now | `FileService.soft_delete()` |
| Restore | Clear `deleted_at` | Standard save |

### Notes
- **Upload must go through `FileService.upload()`** to ensure: storage backend write, FileVersion creation, `current_version` pointer update, and AuditLog entry.
- Override `save_model()` to call the service layer instead of raw ORM save.
- The admin add form needs a `file` upload widget (not stored on the model — passed to the service).

---

## 4. storage — `FileVersion`

**Admin class:** `FileVersionAdmin` (read-only)

### Fields to display

| List display | Detail |
|---|---|
| `file`, `version_number`, `size_bytes`, `storage_key`, `created_by`, `created_at` | All fields, all read-only |

### Filters & search
- **Search:** file name, created_by email
- **Filters:** created_at

### Actions
None — **read-only**. Versions are created only through `FileService.upload()`.

### Permissions
- `has_add_permission` → False
- `has_change_permission` → False
- `has_delete_permission` → False

---

## 5. sharing — `FileShare`

**Admin class:** `FileShareAdmin`

### Fields to display

| List display | Detail |
|---|---|
| `file`, `shared_with`, `shared_by`, `permission`, `revoked_at`, `created_at` | file, shared_with, shared_by (read-only after creation), permission, revoked_at (read-only) |

### Filters & search
- **Search:** file name, shared_with email, shared_by email
- **Filters:** permission, revoked_at (null = active)

### Actions
| Action | Description | Service layer hook |
|---|---|---|
| Create share | Pick file, recipient, permission level | `ShareService.share()` — validates recipient, creates record, logs AuditLog |
| Revoke share | Set `revoked_at` to now | `ShareService.revoke()` — timestamps and logs |

### Notes
- **Create must go through `ShareService.share()`** for validation (no self-sharing, duplicate check) and audit logging.
- **Revoke must go through `ShareService.revoke()`** for audit logging.
- `shared_by` should auto-populate with the admin user on creation.
- `revoked_at` should not be directly editable — use the revoke action instead.

---

## 6. sharing — `FolderShare`

**Admin class:** `FolderShareAdmin`

Same configuration as `FileShareAdmin` above, but for folders.

### Fields to display

| List display | Detail |
|---|---|
| `folder`, `shared_with`, `shared_by`, `permission`, `revoked_at`, `created_at` | folder, shared_with, shared_by (read-only after creation), permission, revoked_at (read-only) |

### Filters & search
- **Search:** folder name, shared_with email, shared_by email
- **Filters:** permission, revoked_at (null = active)

### Actions
| Action | Description | Service layer hook |
|---|---|---|
| Create share | Pick folder, recipient, permission level | `ShareService.share()` |
| Revoke share | Set `revoked_at` to now | `ShareService.revoke()` |

### Notes
- Same service-layer routing rules as FileShare.

---

## 7. audit — `AuditLog`

**Admin class:** `AuditLogAdmin` (read-only, already exists)

### Fields to display

| List display | Detail |
|---|---|
| `created_at`, `actor`, `action`, `target_type`, `target_name`, `ip_address` | All fields including `metadata` (JSON), all read-only |

### Filters & search
- **Search:** actor email, target_name, action
- **Filters:** action, target_type, created_at

### Actions
None — **read-only, append-only**. The model enforces this at the ORM level.

### Permissions
- `has_add_permission` → False
- `has_change_permission` → False
- `has_delete_permission` → False

---

## Summary: Service Layer Integration Points

The key implementation challenge is routing admin saves through the existing service
layer rather than doing raw ORM creates. These are the models that **must** use
service-layer hooks via `save_model()` or custom admin actions:

| Model | Operation | Service method |
|---|---|---|
| `File` | Create (upload) | `FileService.upload()` |
| `File` | Upload new version | `FileService.upload()` |
| `File` | Soft-delete | `FileService.soft_delete()` |
| `FileShare` | Create | `ShareService.share()` |
| `FileShare` | Revoke | `ShareService.revoke()` |
| `FolderShare` | Create | `ShareService.share()` |
| `FolderShare` | Revoke | `ShareService.revoke()` |

All other models (User, Folder) can use standard Django admin save behavior,
as they have no side effects beyond the database write.

---

## README Updates

After implementing the admin configuration, the following READMEs must be updated:

### `vaultshare-api/README.md`

- **Features section** — add "Django Admin UI" entry describing admin as the operator
  interface for managing users, files, folders, shares, and viewing audit logs.
- **API Endpoints section** — add a note that Django admin is available at `/admin/`.
- **Development section** — add `createsuperuser` instructions:
  ```
  docker compose exec api python manage.py createsuperuser
  ```
  If a `make createsuperuser` convenience target is added, document it alongside
  the other make commands.

### `vaultshare-infra/README.md`

- **Getting Started** — add a step between "Seed demo data" and "Verify features"
  for creating a superuser to access the admin panel.
- **Services table** — note that the `api` service also serves Django admin at
  `:8000/admin/`.

### `vaultshare-docs/README.md`

- **Contents table** — add a row for `django_admin_plan.md` (this document).
