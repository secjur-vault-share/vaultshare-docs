# Phase 4 — Permissions & Sharing: Detailed Implementation Blueprint

This blueprint dictates the exact technical specifications for Phase 4 implementation.
It dictates the architectural boundaries, database schema, file structures, service
interfaces, and test contracts. No code will be written until this blueprint is
approved.

---

## 1. Scope & Deliverables

Phase 4 introduces the **permission and sharing boundary**. It delivers:

- `FileShare`, `FolderShare` models with permission levels
- `PermissionLevel` enum: `VIEWER`, `EDITOR`, `OWNER`
- `PermissionService`: `resolve(user, target)` → effective permission;
  `require(user, target, minimum_level)` → raises `PermissionDenied`
- `ShareService`: `share`, `revoke`, `list_shared_with_me`
- Endpoints: `POST /api/v1/shares/files/{id}/`, `POST /api/v1/shares/folders/{id}/`,
  `DELETE /api/v1/shares/{id}/`, `GET /api/v1/shared-with-me/`
- **All existing `FileService` methods retroactively gated by `PermissionService.require`**
  — the Phase 3 inline ownership checks (`if file.owner_id != user.pk`) are removed
- `FileService.list` expanded to include files shared with the user (not just owned)
- Audit entries for `SHARE` and `REVOKE` actions
- Updated `make seed` v2: 3 users with cross-user shares at varying permission levels
- `tests/acceptance/test_permissions.py` — 3 new acceptance scenarios (cumulative: 6)

**Working state at phase end:** `make up && make seed && make verify` → green on 6 scenarios.

---

## 2. Prerequisite Checks

Before writing any code, verify:

1. `apps.sharing` is already in `INSTALLED_APPS` in `config/settings/base.py`. If not,
   add it.
2. `AuditAction.SHARE` and `AuditAction.REVOKE` already exist in the `AuditAction` enum
   (defined in Phase 3). No migration needed for audit.
3. Phase 3 inline ownership checks exist at:
   - `apps/storage/services.py` line ~91 (`download` method): `if file.owner_id != user.pk`
   - `apps/storage/services.py` line ~121 (`soft_delete` method): `if file.owner_id != user.pk`
   - `apps/storage/views.py` line ~65 (`FileDetailView.get`): `if file.owner_id != request.user.pk`
   These must all be replaced — not supplemented — by `PermissionService.require` calls.
4. `apps/sharing/apps.py` exists with minimal `AppConfig`. No models, services, views, or
   URLs exist yet.

---

## 3. Strict File-by-File Technical Plan

### A. Permission Level Enum (`apps/sharing/constants.py`) — NEW FILE

```python
from enum import IntEnum

class PermissionLevel(IntEnum):
    """
    IntEnum so that comparison operators work for hierarchy checks.
    VIEWER < EDITOR < OWNER.
    """
    VIEWER = 10
    EDITOR = 20
    OWNER = 30
```

Using `IntEnum` with gapped values (10/20/30) enables natural comparison
(`PermissionLevel.EDITOR >= PermissionLevel.VIEWER`) and leaves room for future
intermediate levels without a migration. The numeric values are internal only — they
never appear in API responses or the database.

**Database storage:** The `permission` field on `FileShare`/`FolderShare` stores the
string name (`"VIEWER"`, `"EDITOR"`, `"OWNER"`), not the integer. Conversion between
the string and the enum uses `PermissionLevel[string_value]`.

---

### B. Sharing Models (`apps/sharing/models.py`) — NEW FILE

#### `FileShare`

```python
class FileShare(BaseModel):
    file = models.ForeignKey(
        'storage.File',
        on_delete=models.CASCADE,
        related_name='shares',
    )
    shared_with = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='received_file_shares',
    )
    shared_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='sent_file_shares',
    )
    permission = models.CharField(
        max_length=16,
        choices=[(level.name, level.name) for level in PermissionLevel],
    )
    revoked_at = models.DateTimeField(null=True, blank=True, default=None)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['file', 'shared_with'],
                condition=models.Q(revoked_at__isnull=True),
                name='unique_active_file_share_per_user',
            )
        ]
```

**Constraint rationale:** A user should have at most one active share per file. If the
owner revokes and re-shares, a new `FileShare` row is created (the old one has
`revoked_at` set). The partial unique constraint enforces this at the DB level.

#### `FolderShare`

```python
class FolderShare(BaseModel):
    folder = models.ForeignKey(
        'storage.Folder',
        on_delete=models.CASCADE,
        related_name='shares',
    )
    shared_with = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='received_folder_shares',
    )
    shared_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='sent_folder_shares',
    )
    permission = models.CharField(
        max_length=16,
        choices=[(level.name, level.name) for level in PermissionLevel],
    )
    revoked_at = models.DateTimeField(null=True, blank=True, default=None)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['folder', 'shared_with'],
                condition=models.Q(revoked_at__isnull=True),
                name='unique_active_folder_share_per_user',
            )
        ]
```

Mirrors `FileShare` exactly. Both models inherit from `BaseModel`.

---

### C. PermissionService (`apps/sharing/services.py`) — NEW FILE

This is the central authority for all access control. Every read and mutation in
`FileService`, `FolderService`, and future services must go through
`PermissionService.require` before acting.

#### `PermissionService.resolve`

```python
class PermissionService:
    @staticmethod
    def resolve(user: User, target: File | Folder) -> PermissionLevel | None:
        """
        Returns the effective permission level the user has on the target,
        or None if no access.

        Resolution order:
        1. Owner → OWNER
        2. Direct share on the target (FileShare or FolderShare)
        3. Inherited share from parent folder (FolderShare on any ancestor)
        4. Most permissive wins across all sources
        """
```

**Logic (in order):**

1. **Owner check:** If `target.owner_id == user.pk`, return `PermissionLevel.OWNER`.
   This is the fast path — owners always have full access. No DB queries needed beyond
   the target itself (which is already loaded).

2. **Direct share check:**
   - If target is a `File`: query
     `FileShare.objects.filter(file=target, shared_with=user, revoked_at__isnull=True)`
   - If target is a `Folder`: query
     `FolderShare.objects.filter(folder=target, shared_with=user, revoked_at__isnull=True)`
   - Extract the permission level from the share if found.

3. **Inherited folder share check** (applies to both `File` and `Folder` targets):
   - If target is a `File` and `target.folder` is not None, walk up the folder tree.
   - If target is a `Folder` and `target.parent` is not None, walk up from `target.parent`.
   - At each ancestor, check for an active `FolderShare` for the user.
   - Collect all found permission levels.

4. **Conflict resolution — "most permissive wins":**
   - Gather all permission levels from steps 2 and 3.
   - Return `max(levels)` if any exist, else `None`.

**Type discrimination:** Use `isinstance(target, File)` / `isinstance(target, Folder)`
to branch logic. Import both models at the top of the file. This is acceptable because
`sharing` already depends on `storage` for the FK relationships.

**Performance note:** The ancestor walk is O(depth) queries. This is acceptable per the
architecture's documented trade-off (Adjacency List, Section 9 of ARCHITECTURE.md). For
a typical folder depth of 3-5 levels, this means 3-5 lightweight indexed queries. If
performance becomes an issue, the upgrade path is materialized path (documented in
ARCHITECTURE.md Section 9).

**Optimization:** Use a single query with `__in` to check all ancestor folder IDs at
once rather than querying per level:

```python
ancestor_ids = []
current = target.folder if isinstance(target, File) else target.parent
while current is not None:
    ancestor_ids.append(current.pk)
    current = current.parent

if ancestor_ids:
    inherited_shares = FolderShare.objects.filter(
        folder_id__in=ancestor_ids,
        shared_with=user,
        revoked_at__isnull=True,
    )
```

This reduces the inheritance check to two queries total (one walk to collect ancestor
IDs — which hits the ORM cache if parents were `select_related` — and one IN query for
shares), regardless of depth.

#### `PermissionService.require`

```python
    @staticmethod
    def require(
        user: User,
        target: File | Folder,
        minimum_level: PermissionLevel,
    ) -> None:
        """
        Raises PermissionDenied if the user's effective permission on the target
        is below minimum_level, or if the user has no access at all.
        """
        effective = PermissionService.resolve(user, target)
        if effective is None or effective < minimum_level:
            raise PermissionDenied(
                "You do not have permission to perform this action."
            )
```

Raises `rest_framework.exceptions.PermissionDenied` — caught by the global exception
handler and rendered as `{"error": {"code": "PERMISSION_DENIED", ...}}`.

---

### D. ShareService (`apps/sharing/services.py`) — same file as PermissionService

#### `ShareService.share`

```python
class ShareService:
    @staticmethod
    def share(
        *,
        owner: User,
        target: File | Folder,
        recipient_email: str,
        permission: PermissionLevel,
        ip: str | None,
    ) -> FileShare | FolderShare:
```

**Logic (in order):**

1. **Resolve recipient:** `User.objects.get(email=recipient_email)` — raise
   `ValidationError("User not found.")` if no match. Using email lookup (not user ID)
   matches the frontend UX where users search by email.

2. **Self-share guard:** If `recipient.pk == owner.pk`, raise
   `ValidationError("Cannot share with yourself.")`.

3. **Permission check:** `PermissionService.require(owner, target, PermissionLevel.OWNER)`.
   Only owners can share. This is the architecture's rule — EDITOR cannot re-share.

4. **Create the share:**
   - If target is `File`: create `FileShare(file=target, shared_with=recipient, shared_by=owner, permission=permission.name)`.
   - If target is `Folder`: create `FolderShare(folder=target, shared_with=recipient, shared_by=owner, permission=permission.name)`.
   - The DB unique constraint (`unique_active_file_share_per_user` /
     `unique_active_folder_share_per_user`) prevents duplicate active shares. Catch
     `IntegrityError` and raise `ValidationError("This file/folder is already shared with this user.")`.

5. **Audit:**
   ```python
   AuditService.append(
       actor=owner,
       action=AuditAction.SHARE,
       target_type='file' if isinstance(target, File) else 'folder',
       target_id=target.id,
       target_name=target.name,
       ip=ip,
       shared_with=recipient.email,
       permission=permission.name,
   )
   ```

6. Return the created share object.

#### `ShareService.revoke`

```python
    @staticmethod
    def revoke(
        *,
        owner: User,
        share_id: uuid.UUID,
        target_type: str,
        ip: str | None,
    ) -> None:
```

**Logic (in order):**

1. **Lookup the share:** Based on `target_type`:
   - `"file"`: `FileShare.objects.get(pk=share_id, revoked_at__isnull=True)`
   - `"folder"`: `FolderShare.objects.get(pk=share_id, revoked_at__isnull=True)`
   - Raise `NotFound` if the share does not exist or is already revoked.

2. **Permission check:** Resolve the underlying target from the share object
   (`share.file` or `share.folder`).
   `PermissionService.require(owner, target, PermissionLevel.OWNER)`.
   Only the owner of the file/folder can revoke shares.

3. **Soft-revoke:** `share.revoked_at = timezone.now(); share.save()`.

4. **Audit:**
   ```python
   AuditService.append(
       actor=owner,
       action=AuditAction.REVOKE,
       target_type=target_type,
       target_id=target.id,
       target_name=target.name,
       ip=ip,
       revoked_share_id=str(share_id),
       revoked_user=share.shared_with.email,
       permission=share.permission,  # capture the level that was revoked
   )
   ```

   **Audit precision rationale:** Including the `permission` level in the revocation
   audit entry is essential for security forensics. When reviewing the audit trail for
   a file, investigators need to know not just *who* lost access, but *what level* of
   access was revoked. A `REVOKE` entry without the permission level is ambiguous —
   "was this a VIEWER being removed, or an EDITOR?" — and forces a join against the
   (now-soft-deleted) share row, defeating the purpose of denormalized audit metadata.
   The `share.permission` string is read before the revoke is committed, so it is
   always available at this point.

#### `ShareService.list_shared_with_me`

```python
    @staticmethod
    def list_shared_with_me(user: User) -> dict:
        """
        Returns all active shares where the user is the recipient.
        Result: {"files": QuerySet[FileShare], "folders": QuerySet[FolderShare]}
        """
```

**Logic:**

1. Query active file shares:
   ```python
   file_shares = FileShare.objects.filter(
       shared_with=user,
       revoked_at__isnull=True,
       file__deleted_at__isnull=True,  # exclude soft-deleted files
   ).select_related('file', 'file__current_version', 'shared_by')
   ```

2. Query active folder shares:
   ```python
   folder_shares = FolderShare.objects.filter(
       shared_with=user,
       revoked_at__isnull=True,
       folder__deleted_at__isnull=True,  # exclude soft-deleted folders
   ).select_related('folder', 'shared_by')
   ```

3. Return `{"files": file_shares, "folders": folder_shares}`.

**Design choice:** Return share objects (not bare files/folders) so the serializer can
include `permission` and `shared_by` metadata in the response. The frontend needs this
to render permission badges.

---

### E. Retroactive Permission Integration — `apps/storage/services.py` Modifications

This is the critical integration step. All Phase 3 inline ownership checks must be
replaced by `PermissionService.require` calls.

#### `FileService.download` — replace inline check

**Current (Phase 3):**
```python
if file.owner_id != user.pk:
    raise PermissionDenied("You do not have permission to access this file.")
```

**Replace with:**
```python
from apps.sharing.services import PermissionService
from apps.sharing.constants import PermissionLevel

PermissionService.require(user, file, PermissionLevel.VIEWER)
```

Minimum level: `VIEWER`. Viewers, editors, and owners can all download.

#### `FileService.soft_delete` — replace inline check

**Current (Phase 3):**
```python
if file.owner_id != user.pk:
    raise PermissionDenied("You do not have permission to delete this file.")
```

**Replace with:**
```python
PermissionService.require(user, file, PermissionLevel.OWNER)
```

Minimum level: `OWNER`. Only owners can delete. This is intentional — even EDITORs
cannot delete files they don't own.

#### `FileService.list` — expand to include shared files

**Current (Phase 3):**
```python
return File.objects.filter(
    owner=user, folder_id=folder_id, deleted_at__isnull=True
)
```

**Replace with:**
```python
from apps.sharing.models import FileShare, FolderShare

owned = File.objects.filter(
    owner=user,
    folder_id=folder_id,
    deleted_at__isnull=True,
)

# Files directly shared with the user in this folder
shared_file_ids = FileShare.objects.filter(
    shared_with=user,
    revoked_at__isnull=True,
    file__folder_id=folder_id,
    file__deleted_at__isnull=True,
).values_list('file_id', flat=True)

shared = File.objects.filter(
    pk__in=shared_file_ids,
    deleted_at__isnull=True,
)

return (owned | shared).distinct().select_related('current_version')
```

**Note on folder inheritance:** When listing files in a folder, if the user has a
`FolderShare` on that folder (or any ancestor), they should see all files in it. This
case is handled by adding:

```python
# Check if user has a folder share granting access to this folder
if folder_id is not None:
    folder = Folder.objects.filter(pk=folder_id, deleted_at__isnull=True).first()
    if folder is not None:
        effective = PermissionService.resolve(user, folder)
        if effective is not None:
            # User has access to the folder — show all active files in it
            folder_files = File.objects.filter(
                folder_id=folder_id,
                deleted_at__isnull=True,
            )
            return (owned | shared | folder_files).distinct().select_related('current_version')
```

This ensures that a user with a `FolderShare` on `Projects/` can list all files inside
`Projects/` even if they don't have individual `FileShare` entries per file.

#### `FileDetailView.get` — replace inline check in view

**Current (Phase 3):**
```python
if file.owner_id != request.user.pk:
    raise PermissionDenied(...)
```

**Replace with:**
```python
PermissionService.require(request.user, file, PermissionLevel.VIEWER)
```

The view fetches the file and delegates permission checking to the service layer.

---

### F. Sharing Serializers (`apps/sharing/serializers.py`) — NEW FILE

#### `ShareCreateSerializer`

```python
class ShareCreateSerializer(serializers.Serializer):
    recipient_email = serializers.EmailField()
    permission = serializers.ChoiceField(
        choices=[(level.name, level.name) for level in PermissionLevel],
    )
```

Used by both file and folder share creation views. The target is derived from the URL
path (`/shares/files/{file_id}/` or `/shares/folders/{folder_id}/`), not from the
request body.

#### `FileShareSerializer`

```python
class FileShareSerializer(serializers.ModelSerializer):
    shared_with_email = serializers.EmailField(source='shared_with.email', read_only=True)
    shared_by_email = serializers.EmailField(source='shared_by.email', read_only=True)
    file = FileSerializer(read_only=True)

    class Meta:
        model = FileShare
        fields = ['id', 'file', 'shared_with_email', 'shared_by_email',
                  'permission', 'created_at']
```

#### `FolderShareSerializer`

```python
class FolderShareSerializer(serializers.ModelSerializer):
    shared_with_email = serializers.EmailField(source='shared_with.email', read_only=True)
    shared_by_email = serializers.EmailField(source='shared_by.email', read_only=True)
    folder_id = serializers.UUIDField(source='folder.id', read_only=True)
    folder_name = serializers.CharField(source='folder.name', read_only=True)

    class Meta:
        model = FolderShare
        fields = ['id', 'folder_id', 'folder_name', 'shared_with_email',
                  'shared_by_email', 'permission', 'created_at']
```

#### `SharedWithMeSerializer`

```python
class SharedWithMeSerializer(serializers.Serializer):
    files = FileShareSerializer(many=True)
    folders = FolderShareSerializer(many=True)
```

---

### G. Sharing Views (`apps/sharing/views.py`) — NEW FILE

All views are thin. Pattern: validate input → call service → return response envelope.

#### `FileShareView` — `POST /shares/files/{file_id}/`

```python
class FileShareView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request, file_id):
        serializer = ShareCreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        file = File.objects.filter(
            pk=file_id, deleted_at__isnull=True
        ).first()
        if file is None:
            raise NotFound("File not found.")

        share = ShareService.share(
            owner=request.user,
            target=file,
            recipient_email=serializer.validated_data['recipient_email'],
            permission=PermissionLevel[serializer.validated_data['permission']],
            ip=get_client_ip(request),
        )
        return Response(
            {"data": FileShareSerializer(share).data},
            status=status.HTTP_201_CREATED,
        )
```

#### `FolderShareView` — `POST /shares/folders/{folder_id}/`

Mirrors `FileShareView` but with `Folder` lookup and `FolderShareSerializer`.

#### `ShareRevokeView` — `DELETE /shares/{share_id}/`

```python
class ShareRevokeView(APIView):
    permission_classes = [IsAuthenticated]

    def delete(self, request, share_id):
        target_type = request.query_params.get('type')
        if target_type not in ('file', 'folder'):
            raise ValidationError("Query parameter 'type' must be 'file' or 'folder'.")

        ShareService.revoke(
            owner=request.user,
            share_id=share_id,
            target_type=target_type,
            ip=get_client_ip(request),
        )
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**Design note:** The `type` query parameter tells the view which share model to look up.
This avoids a polymorphic endpoint that tries both models. The frontend knows the share
type from context (it rendered the share list with type metadata).

#### `SharedWithMeView` — `GET /shared-with-me/`

```python
class SharedWithMeView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        result = ShareService.list_shared_with_me(user=request.user)
        return Response({
            "data": {
                "files": FileShareSerializer(result["files"], many=True).data,
                "folders": FolderShareSerializer(result["folders"], many=True).data,
            }
        })
```

---

### H. URL Registration

#### `apps/sharing/urls.py` — NEW FILE

```python
from django.urls import path
from apps.sharing.views import (
    FileShareView,
    FolderShareView,
    ShareRevokeView,
    SharedWithMeView,
)

urlpatterns = [
    path("files/<uuid:file_id>/", FileShareView.as_view(), name="file-share"),
    path("folders/<uuid:folder_id>/", FolderShareView.as_view(), name="folder-share"),
    path("<uuid:share_id>/", ShareRevokeView.as_view(), name="share-revoke"),
]
```

#### `config/urls.py` — MODIFY

Add to `urlpatterns`:

```python
path("api/v1/shares/", include("apps.sharing.urls")),
path("api/v1/shared-with-me/", SharedWithMeView.as_view(), name="shared-with-me"),
```

**Note:** `shared-with-me` is a top-level route (not nested under `/shares/`) because
it is semantically distinct — it is a user-centric view, not a share management
operation. Import `SharedWithMeView` directly in `config/urls.py`.

---

### I. Audit View Update (`apps/audit/views.py`) — MODIFY

Phase 3 left a `TODO` for folder audit access. Now that `PermissionService` exists,
update the `AuditLogListView` to handle `target_type="folder"`:

**Current (Phase 3):**
```python
# "folder" / "public_link": return 403 "Audit access for this target_type is not yet implemented."
```

**Replace folder case with:**
```python
elif target_type == "folder":
    folder = Folder.objects.filter(pk=target_id, deleted_at__isnull=True).first()
    if folder is None:
        raise NotFound("Folder not found.")
    PermissionService.require(request.user, folder, PermissionLevel.OWNER)
```

Leave `"public_link"` as the Phase 7 TODO.

---

### J. Settings Update (`config/settings/base.py`) — MODIFY

Ensure `apps.sharing` is in `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    ...
    "apps.sharing",
    ...
]
```

If it is already listed (from Phase 1 scaffold), no change needed. Verify before
modifying.

---

### K. Error Codes Update (`apps/core/exceptions.py`) — MODIFY

Add `ValidationError` mapping if not already present (Phase 3 may have added it):

```python
from rest_framework.exceptions import ValidationError

ERROR_CODE_MAP = {
    ...
    ValidationError: "VALIDATION_FAILED",
}
```

New error codes used in Phase 4:

| Condition | HTTP status | `code` |
|---|---|---|
| Non-owner attempts to share | 403 | `PERMISSION_DENIED` |
| Non-owner attempts to revoke share | 403 | `PERMISSION_DENIED` |
| VIEWER attempts to delete | 403 | `PERMISSION_DENIED` |
| EDITOR attempts to delete | 403 | `PERMISSION_DENIED` |
| Share with self | 400 | `VALIDATION_FAILED` |
| Recipient email not found | 400 | `VALIDATION_FAILED` |
| Duplicate active share | 400 | `VALIDATION_FAILED` |
| Share not found on revoke | 404 | `NOT_FOUND` |
| Missing `type` query param on revoke | 400 | `VALIDATION_FAILED` |

---

## 4. Migration Plan

One new migration required:

1. `apps/sharing/migrations/0001_initial.py` — creates `FileShare` and `FolderShare`
   tables, including partial unique constraints.

No changes to `apps/storage` or `apps/audit` migrations.

**Generate and verify:** Run `makemigrations sharing --check` to confirm no unexpected
dependencies, then `makemigrations sharing` followed by `migrate`.

---

## 5. Seed Script Update (`make seed` v2)

**Location:** `apps/core/management/commands/seed.py`

**Scope (Phase 4 v2) — extend existing seed, do not rewrite:**

- Add 2 new users beyond the existing `demo@vaultshare.io`:
  - `alice@vaultshare.io` / `alice1234`
  - `bob@vaultshare.io` / `bob1234`
- Each new user gets 2-3 files uploaded via `FileService.upload`
- Cross-user shares:
  - `demo` shares a file with `alice` as `VIEWER`
  - `demo` shares a file with `bob` as `EDITOR`
  - `alice` shares a file with `demo` as `VIEWER`
  - `demo` shares a folder with `alice` as `VIEWER` (if folders exist; otherwise skip
    until Phase 6 extends the seed)
- Each share flows through `ShareService.share` → generates real `AuditLog` entries
- One revoked share: `demo` shares a file with `bob`, then immediately revokes it via
  `ShareService.revoke` — produces both `SHARE` and `REVOKE` audit entries

**Idempotency:** Check for existing users by email before creating. Check for existing
active shares before creating. The `--reset` flag wipes all sharing data along with
storage data.

---

## 6. Acceptance Tests (`tests/acceptance/test_permissions.py`) — NEW FILE

**Location:** `tests/acceptance/test_permissions.py`

Uses `httpx` against `http://localhost:8000`. Assumes `make seed` has run.

**Setup fixture:** Helper function `register_and_login(email, password, username)` that
registers a new user (or logs in if already exists) and returns Bearer headers. Each
test uses unique user emails with UUID suffix to avoid collisions with seed data.

```python
def register_and_login(email: str, password: str, username: str) -> dict:
    """Register (if needed) and login, return auth headers."""
    httpx.post(
        f"{BASE_URL}/api/v1/auth/register/",
        json={"email": email, "password": password, "username": username},
    )
    response = httpx.post(
        f"{BASE_URL}/api/v1/auth/login/",
        json={"email": email, "password": password},
    )
    token = response.json()["data"]["access"]
    return {"Authorization": f"Bearer {token}"}
```

### Scenario 4: VIEWER cannot delete

```python
def test_viewer_cannot_delete(fresh_filename):
    """
    Owner shares file with recipient as VIEWER.
    Recipient attempts DELETE → assert 403.
    """
    # 1. Owner registers, logs in, uploads a file
    owner_headers = register_and_login(f"owner4_{uuid}@test.io", "pass1234", "owner4")
    filename = fresh_filename("perm_viewer", "txt")
    upload_resp = httpx.post(
        f"{BASE_URL}/api/v1/files/",
        headers=owner_headers,
        files={"file": (filename, b"viewer cannot delete this", "text/plain")},
    )
    file_id = upload_resp.json()["data"]["id"]

    # 2. Recipient registers, logs in
    viewer_headers = register_and_login(f"viewer4_{uuid}@test.io", "pass1234", "viewer4")

    # 3. Owner shares file with recipient as VIEWER
    httpx.post(
        f"{BASE_URL}/api/v1/shares/files/{file_id}/",
        headers=owner_headers,
        json={"recipient_email": f"viewer4_{uuid}@test.io", "permission": "VIEWER"},
    )

    # 4. Recipient attempts DELETE
    delete_resp = httpx.delete(
        f"{BASE_URL}/api/v1/files/{file_id}/",
        headers=viewer_headers,
    )
    assert delete_resp.status_code == 403
    assert delete_resp.json()["error"]["code"] == "PERMISSION_DENIED"
```

### Scenario 5: EDITOR cannot revoke share

```python
def test_editor_cannot_revoke(fresh_filename):
    """
    Owner shares file with editor as EDITOR.
    Editor attempts to revoke the share → assert 403.
    """
    # 1. Owner uploads file
    # 2. Editor registers
    # 3. Owner shares file with editor as EDITOR
    share_resp = httpx.post(
        f"{BASE_URL}/api/v1/shares/files/{file_id}/",
        headers=owner_headers,
        json={"recipient_email": editor_email, "permission": "EDITOR"},
    )
    share_id = share_resp.json()["data"]["id"]

    # 4. Editor attempts to revoke the share
    revoke_resp = httpx.delete(
        f"{BASE_URL}/api/v1/shares/{share_id}/?type=file",
        headers=editor_headers,
    )
    assert revoke_resp.status_code == 403
    assert revoke_resp.json()["error"]["code"] == "PERMISSION_DENIED"
```

### Scenario 6: Owner revokes share → file absent from recipient's shared view

```python
def test_revoke_removes_access(fresh_filename):
    """
    Owner shares file with recipient. Owner revokes.
    Recipient checks shared-with-me → file is absent.
    """
    # 1. Owner uploads file, shares with recipient
    # 2. Confirm file appears in recipient's shared-with-me
    shared_resp = httpx.get(
        f"{BASE_URL}/api/v1/shared-with-me/",
        headers=recipient_headers,
    )
    file_ids = [s["file"]["id"] for s in shared_resp.json()["data"]["files"]]
    assert file_id in file_ids

    # 3. Owner revokes the share
    httpx.delete(
        f"{BASE_URL}/api/v1/shares/{share_id}/?type=file",
        headers=owner_headers,
    )

    # 4. Recipient checks again → file absent
    shared_resp = httpx.get(
        f"{BASE_URL}/api/v1/shared-with-me/",
        headers=recipient_headers,
    )
    file_ids = [s["file"]["id"] for s in shared_resp.json()["data"]["files"]]
    assert file_id not in file_ids
```

**`make verify` wiring:** Add `tests/acceptance/test_permissions.py` to the existing
acceptance test suite. `make verify` now runs all 6 scenarios (3 from Phase 3 + 3 new).

---

## 7. Tests — Unit & Integration

### Unit tests (`tests/unit/test_permissions.py`) — NEW FILE

- `test_owner_resolves_to_owner_level`: Create a file. Resolve for owner → assert
  `PermissionLevel.OWNER`.
- `test_no_share_resolves_to_none`: Create a file. Resolve for a different user with no
  shares → assert `None`.
- `test_direct_file_share_resolves`: Create a file, share as VIEWER. Resolve → assert
  `PermissionLevel.VIEWER`.
- `test_direct_folder_share_resolves`: Create a folder, share as EDITOR. Resolve → assert
  `PermissionLevel.EDITOR`.
- `test_inherited_folder_share_on_file`: Create folder, share folder as VIEWER with user.
  Create file in folder. Resolve user on file → assert `PermissionLevel.VIEWER`.
- `test_most_permissive_wins`: Share folder as VIEWER. Share file (inside folder)
  directly as EDITOR. Resolve on file → assert `PermissionLevel.EDITOR` (direct wins
  because it's higher).
- `test_require_raises_on_insufficient`: Create file, share as VIEWER. Require EDITOR →
  assert `PermissionDenied` raised.
- `test_require_passes_on_sufficient`: Create file, share as EDITOR. Require VIEWER →
  no exception.
- `test_revoked_share_not_resolved`: Create file, share as VIEWER, revoke the share.
  Resolve → assert `None`.

### Unit test: Exception Envelope (`tests/unit/test_exception_handler.py`) — NEW FILE

This test verifies that the global exception handler in `apps/core/exceptions.py`
correctly wraps DRF built-in exceptions into our custom `{ "error": { ... } }` envelope.
This is critical: if DRF's default renderer is ever reinstated or the handler is
misconfigured, these tests will catch it immediately rather than leaking non-envelope
error shapes to API consumers.

- `test_permission_denied_produces_error_envelope`: Simulate a view that raises
  `rest_framework.exceptions.PermissionDenied`. Assert the response body is
  `{"error": {"code": "PERMISSION_DENIED", "message": "..."}}` and the HTTP status
  is 403. Do **not** check the exact message — only the shape and `code`.

- `test_validation_error_produces_error_envelope`: Simulate a view that raises
  `rest_framework.exceptions.ValidationError("some detail")`. Assert the response body
  is `{"error": {"code": "VALIDATION_FAILED", "message": "..."}}` and the HTTP status
  is 400.

**Implementation note:** Use `pytest-django`'s `APIRequestFactory` to fabricate a
request and call the handler directly:

```python
from rest_framework.exceptions import PermissionDenied, ValidationError
from apps.core.exceptions import custom_exception_handler
from rest_framework.views import exception_handler as drf_default_handler

def test_permission_denied_produces_error_envelope(rf):
    exc = PermissionDenied("You shall not pass.")
    context = {"request": rf.get("/")}
    response = custom_exception_handler(exc, context)
    assert response.status_code == 403
    assert response.data["error"]["code"] == "PERMISSION_DENIED"
    assert "error" in response.data
    assert "data" not in response.data  # must never have both keys
```

The dual assertion `"error" in response.data` and `"data" not in response.data` guards
against an accidental hybrid envelope which would be valid JSON but violate the
API contract.

### Integration tests (`tests/integration/test_sharing.py`) — NEW FILE

- `test_share_file_creates_share_and_audit`: Owner shares file → assert `FileShare`
  exists in DB → assert `AuditLog` entry with `action=SHARE`.
- `test_share_with_self_rejected`: Owner tries to share with own email → assert 400.
- `test_duplicate_active_share_rejected`: Share file with user, share same file with same
  user again → assert 400.
- `test_revoke_sets_revoked_at_and_audit`: Owner revokes share → assert
  `share.revoked_at` is set → assert `AuditLog` entry with `action=REVOKE`.
- `test_viewer_can_download`: Share file as VIEWER → recipient downloads → assert 200.
- `test_viewer_cannot_delete`: Share file as VIEWER → recipient attempts delete → assert 403.
- `test_editor_can_download`: Share file as EDITOR → recipient downloads → assert 200.
- `test_editor_cannot_delete`: Share file as EDITOR → recipient attempts delete → assert 403.
- `test_shared_with_me_returns_active_shares`: Share file and folder. Query
  `/shared-with-me/` → assert both appear.
- `test_shared_with_me_excludes_revoked`: Share, then revoke. Query → assert absent.
- `test_shared_with_me_excludes_deleted_files`: Share file, then owner soft-deletes it.
  Query `/shared-with-me/` → assert file absent.

---

## 8. Sequence of Implementation Steps

The implementer must follow this sequence to avoid circular import issues and ensure
each step is independently verifiable:

1. `apps/sharing/constants.py` — `PermissionLevel` enum
2. `apps/sharing/models.py` — `FileShare`, `FolderShare`
3. `makemigrations sharing` + `migrate` — verify tables created
4. `apps/sharing/services.py` — `PermissionService` (resolve + require)
5. **Retroactive integration** — `apps/storage/services.py`:
   - Replace inline ownership check in `FileService.download` with `PermissionService.require(..., VIEWER)`
   - Replace inline ownership check in `FileService.soft_delete` with `PermissionService.require(..., OWNER)`
   - Expand `FileService.list` to include shared files
6. **Retroactive integration** — `apps/storage/views.py`:
   - Replace inline check in `FileDetailView.get` with `PermissionService.require(..., VIEWER)`
7. `apps/sharing/services.py` — `ShareService` (share, revoke, list_shared_with_me)
8. `apps/sharing/serializers.py`
9. `apps/sharing/views.py`
10. `apps/sharing/urls.py`
11. `config/urls.py` — add sharing URL includes
12. `apps/audit/views.py` — update folder audit access to use `PermissionService`
13. `config/settings/base.py` — verify `apps.sharing` in `INSTALLED_APPS`
14. Update seed script (`apps/core/management/commands/seed.py`)
15. `tests/unit/test_permissions.py`
16. `tests/integration/test_sharing.py`
17. `tests/acceptance/test_permissions.py`
18. `make check` (ruff + mypy) — must pass clean
19. `make verify` — must be green on all 6 acceptance scenarios

---

## 9. What Is Deliberately Out of Scope for Phase 4

| Feature | Phase |
|---|---|
| Version increment on same-name re-upload | 5 |
| `GET /files/{id}/versions/` endpoint | 5 |
| Folder CRUD endpoints | 6 |
| ZIP download | 6 |
| FolderShare inheritance in `FolderService.list` | 6 |
| Public links | 7 |
| PERMISSION_CHANGED audit action (share update/upgrade) | Future |
| Cascading revoke when parent folder share is revoked | Future |

**Deliberate simplification:** Phase 4 does not implement "update share permission"
(e.g., upgrading a VIEWER to EDITOR). The owner must revoke and re-share. This is a
known UX gap acceptable for the time budget. The `PERMISSION_CHANGED` audit action is
defined in the enum but unused until this feature is added.

---

## 10. Consistency Notes

- **`PermissionService.require` is the single gate.** No service method or view may
  perform its own ownership check after Phase 4. All access control flows through
  `PermissionService.require`. Grep for `owner_id != user` and `owner != user` after
  implementation — zero matches expected outside of `PermissionService.resolve`.
- **Share model uses string permission, not IntEnum.** The `permission` field stores
  `"VIEWER"`, `"EDITOR"`, `"OWNER"` as strings. Conversion to `PermissionLevel` for
  comparison happens in `PermissionService.resolve` via `PermissionLevel[share.permission]`.
- **Revocation is soft.** `revoked_at` is set to `now()`. The share row is preserved for
  audit trail purposes. Filters must always include `revoked_at__isnull=True` when
  checking active shares.
- **`AuditService.append` metadata:** Share and revoke events include `shared_with`
  (email) and `permission` (level name) in the `metadata` JSON field. This denormalizes
  the data for audit readability — the audit log must be interpretable without joining
  to the share table. Revocation metadata additionally includes `permission` (the level
  that was revoked) for security forensics — see Section D `ShareService.revoke`.
- **Response envelope:** All new views return `{"data": ...}` on success. The global
  exception handler covers error responses. No manual error wrapping in views.
- **Cross-app import:** `apps.sharing` imports from `apps.storage.models` (File, Folder)
  and `apps.audit.services` (AuditService). `apps.storage` imports from
  `apps.sharing.services` (PermissionService) and `apps.sharing.models` (FileShare,
  FolderShare). This creates a bidirectional dependency between `storage` and `sharing`.
  This is acceptable because both are within the same Django project and the dependency
  is through service calls (not model inheritance). The architecture document permits
  cross-app communication through service calls.
- **`make check` gate:** `ruff check` + `mypy --strict` must pass with zero errors after
  each step. Do not defer linting to end of phase.

---

## 11. Architectural Amendment — ContentTypes & `core` Consolidation (Decision Record)

### Context

The current plan (Section 10, "Cross-app import") acknowledges a **bidirectional
dependency** between `apps.storage` and `apps.sharing`: storage calls `PermissionService`
(in sharing), and sharing holds FKs to `File`/`Folder` (in storage). This is a circular
dependency at the model level — a recognised sign of suboptimal decoupling.

### Superior Alternative: Move Sharing Models into `apps/core` with ContentTypes

The architecturally cleaner solution — appropriate if time budget allows — is to:

1. **Move `FileShare`, `FolderShare`, and `PermissionService`** into `apps/core`,
   replacing the concrete FK fields (`file = FK('storage.File')`) with Django
   **Generic Foreign Keys** (`ContentType` + `object_id`):

   ```python
   # apps/core/models.py (new addition)
   from django.contrib.contenttypes.fields import GenericForeignKey
   from django.contrib.contenttypes.models import ContentType

   class Share(BaseModel):
       content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
       object_id = models.UUIDField()
       target = GenericForeignKey('content_type', 'object_id')  # File or Folder

       shared_with = models.ForeignKey(settings.AUTH_USER_MODEL, ...)
       shared_by  = models.ForeignKey(settings.AUTH_USER_MODEL, ...)
       permission = models.CharField(max_length=16, choices=...)
       revoked_at = models.DateTimeField(null=True, blank=True)
   ```

2. **`PermissionService`** in `apps/core/services.py` calls a `get_owner(target)`
   helper that each app registers. `apps.storage` calls a generic permission service in
   `core`, but `core` has **zero hard imports from `storage`** — the dependency arrow is
   now strictly one-way: `storage → core`.

3. **App count reduction:** Merging into `core` collapses the project from 5 apps
   (`core`, `accounts`, `storage`, `sharing`, `audit`) to **4 apps** — eliminating the
   `sharing` bounded context entirely. For a 12-hour challenge, this is a meaningful
   reduction in scaffolding overhead.

### Why This Plan Does Not Implement It

This plan keeps `apps/sharing` as a standalone app with concrete FKs for the following
reasons:

| Factor | Concrete FKs (current plan) | ContentTypes (superior) |
|---|---|---|
| **DB integrity** | Full referential integrity enforced by DB | No FK constraints on `object_id` — orphans possible |
| **Query simplicity** | Standard ORM joins, type-checked by mypy | GFK lookups are not type-safe; requires `prefetch_related('content_type')` |
| **Admin** | Inline admin on File/Folder with FK drill-down | GFK admin requires custom `GenericInlineAdmin` |
| **Implementation time** | ~1.5h as budgeted | +30–45min for GFK wiring, ContentType registration, and mypy suppression |
| **Circular dependency** | Acknowledged trade-off, contained within Django project | Eliminated |

**Decision: Keep concrete FKs for Phase 4. Document ContentTypes as the production
upgrade path in `CODE_REVIEW.md`.**

If Phase 4 is completed significantly ahead of schedule (i.e., before the hour-4:45
checkpoint), this migration should be considered before moving to Phase 5. The test
suite would need to be re-run from scratch after the refactor.

### Impact on `CODE_REVIEW.md`

Add a `Warning`-severity finding:

> **Bidirectional dependency between `storage` and `sharing`.**
> `apps.sharing` holds concrete FKs to `storage.File` / `storage.Folder`, and
> `apps.storage.services` imports `sharing.services.PermissionService`. The cleaner
> production design is to move sharing models into `core` with Generic Foreign Keys
> (ContentTypes), which eliminates the cycle and reduces app count from 5 to 4.
> Accepted as a deliberate time-budget trade-off for this build.
>
> **Suggested fix (post-challenge):** Migrate `FileShare` / `FolderShare` to a single
> `Share` model with `GenericForeignKey` in `apps/core`. Implement a `PermissionBackend`
> protocol so `storage` resolves permissions without importing `sharing` directly.
