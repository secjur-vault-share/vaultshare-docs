# Phase 3 — Core Files + AuditLog: Detailed Implementation Blueprint

This blueprint dictates the exact technical specifications for Phase 3 implementation.
It dictates the architectural boundaries, database schema, file structures, service
interfaces, and test contracts. No code will be written until this blueprint is
approved.

---

## 1. Scope & Deliverables

Phase 3 is the **MVS checkpoint**. It delivers:

- `Folder`, `File`, `FileVersion` models (version-aware schema, supports Phase 5 with no
  further migrations)
- `AuditLog` model with model-level immutability enforcement
- `AuditAction` enum (all action types for all future phases, defined now)
- `FileService`: `upload`, `download`, `soft_delete`, `list`
- `AuditService`: `append` (single write path, append-only)
- API endpoints: `POST /api/v1/files/`, `GET /api/v1/files/`, `GET /api/v1/files/{id}/`, `GET /api/v1/files/{id}/download/`, `DELETE /api/v1/files/{id}/`
- `GET /api/v1/audit/` — query log by `target_id` (owner only)
- Django admin: `AuditLog` (read-only), `FileVersion` (read-only)
- `make seed` v1: 1 user (`demo@vaultshare.io` / `demo1234`), 5 files, audit entries
- `tests/acceptance/test_core.py` — 3 acceptance scenarios, `make verify` wired

**Working state at phase end:** `make up && make seed && make verify` → green on 3 scenarios.

---

## 2. Prerequisite Checks

Before writing any code, verify:

1. `storage.backends.get_storage_backend()` factory exists and returns a `StorageBackend`
   instance using `settings.STORAGE_BACKEND`. **This does not exist yet** — it must be
   created in `apps/storage/services.py` as part of this phase (it is referenced by
   `FileService`).
2. `apps.audit` and `apps.storage` are already in `INSTALLED_APPS`. ✓
3. `config/urls.py` currently only registers `/api/v1/auth/`. File and audit URL includes
   must be added.

---

## 3. Strict File-by-File Technical Plan

### A. Storage Factory (`apps/storage/services.py`) — NEW FILE

**Action:** Create the storage backend dependency injection factory.

```python
# apps/storage/services.py
def get_storage_backend() -> StorageBackend:
    """
    Reads settings.STORAGE_BACKEND ('local' or 'apps.storage.backends.s3.S3Backend'),
    uses Django's import_string to resolve the class, instantiates it, and returns it.
    Defaults to LocalDiskBackend for the value 'local'.
    """
```

Implementation note: `settings.STORAGE_BACKEND` holds the string `"local"` (as set in
`base.py`). The factory maps `"local"` → `LocalDiskBackend`. Any dotted import path
falls through to `import_string`.

---

### B. New App: `apps/core/models.py` — BaseModel

**Action:** Create abstract `BaseModel` used as base for all domain models.

```python
# apps/core/models.py
class BaseModel(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
```

All models in `apps/storage` and `apps/audit` inherit from `BaseModel`.

---

### C. App: `apps/storage/` — Models (`models.py`)

All three models must be defined **in this phase**. `FileVersion` is included now so
Phase 5 (versioning logic) requires zero migrations.

#### `Folder`

```python
class Folder(BaseModel):
    name: CharField(max_length=255)
    owner: FK(settings.AUTH_USER_MODEL, on_delete=CASCADE, related_name='owned_folders')
    parent: FK('self', null=True, blank=True, on_delete=SET_NULL, related_name='children')
    deleted_at: DateTimeField(null=True, blank=True, default=None)

    class Meta:
        # Note: folder uniqueness constraint is intentionally deferred to Phase 6
        # (FolderService). Not enforced at DB level in this phase.
        pass
```

#### `File`

```python
class File(BaseModel):
    name: CharField(max_length=255)
    owner: FK(settings.AUTH_USER_MODEL, on_delete=CASCADE, related_name='owned_files')
    folder: FK(Folder, null=True, blank=True, on_delete=SET_NULL, related_name='files')
    mime_type: CharField(max_length=255, blank=True)
    deleted_at: DateTimeField(null=True, blank=True, default=None)
    current_version: OneToOneField(
        'FileVersion',
        null=True, blank=True,
        on_delete=SET_NULL,
        related_name='+'
    )

    class Meta:
        constraints = [
            UniqueConstraint(
                fields=['owner', 'folder', 'name'],
                condition=Q(deleted_at__isnull=True),
                name='unique_active_file_per_folder'
            )
        ]
```

#### `FileVersion`

```python
class FileVersion(BaseModel):
    file: FK(File, on_delete=CASCADE, related_name='versions')
    version_number: PositiveIntegerField()
    storage_key: CharField(max_length=1024)   # format: {owner_id}/{file_id}/{version}/{name}
    size_bytes: BigIntegerField()
    created_by: FK(settings.AUTH_USER_MODEL, on_delete=CASCADE, related_name='created_versions')
    # created_at inherited from BaseModel; updated_at not used but inherited

    class Meta:
        unique_together = ('file', 'version_number')
```

**Migration note:** `File.current_version` creates a circular FK
(`File → FileVersion → File`). Django handles this via `null=True` on
`current_version` and the two-step create pattern in `FileService.upload`:
create `File` (current_version=None) → create `FileVersion` → update
`File.current_version`. The migration must use `deferred` constraints or simply
accept that `current_version` is nullable. No special migration trick required —
nullable FK with `SET_NULL` is sufficient.

---

### D. App: `apps/audit/` — Models (`models.py`)

#### `AuditAction` enum

```python
class AuditAction(models.TextChoices):
    UPLOAD = "UPLOAD"
    DOWNLOAD = "DOWNLOAD"
    DELETE = "DELETE"
    SHARE = "SHARE"
    REVOKE = "REVOKE"
    VERSION_CREATED = "VERSION_CREATED"
    LINK_GENERATED = "LINK_GENERATED"
    LINK_ACCESSED = "LINK_ACCESSED"
    PERMISSION_CHANGED = "PERMISSION_CHANGED"
```

Define **all actions upfront** — even those used only in Phase 4–7. This prevents
a migration mid-project.

#### `AuditLog`

```python
class AuditLog(models.Model):
    # Deliberately does NOT inherit BaseModel — no updated_at field.
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    actor = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        null=True, blank=True,          # null = public link access (Phase 7)
        on_delete=models.SET_NULL,
        related_name='audit_actions'
    )
    action = models.CharField(max_length=64, choices=AuditAction.choices)
    target_type = models.CharField(max_length=32)    # 'file', 'folder', 'public_link'
    target_id = models.UUIDField()
    target_name = models.CharField(max_length=255)   # denormalized, survives soft delete
    ip_address = models.GenericIPAddressField(null=True, blank=True)
    metadata = models.JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)
    # NO updated_at
```

#### Immutability enforcement

```python
    def save(self, *args, **kwargs):
        if self.pk:
            raise ImproperlyConfigured("AuditLog is append-only.")
        super().save(*args, **kwargs)

    def delete(self, *args, **kwargs):
        raise ImproperlyConfigured("AuditLog entries cannot be deleted.")
```

#### `AuditLogQuerySet` (custom Manager)

```python
class AuditLogQuerySet(models.QuerySet):
    def update(self, **kwargs):
        raise ImproperlyConfigured("AuditLog is append-only.")

    def delete(self):
        raise ImproperlyConfigured("AuditLog entries cannot be deleted.")

class AuditLogManager(models.Manager):
    def get_queryset(self):
        return AuditLogQuerySet(self.model, using=self._db)
```

Assign `objects = AuditLogManager()` on the model. This prevents bulk
`AuditLog.objects.filter(...).update(...)` and `.delete()` at the ORM level.

---

### E. `apps/audit/services.py` — AuditService

```python
class AuditService:
    @staticmethod
    def append(
        *,
        actor: User | None,
        action: AuditAction,
        target_type: str,
        target_id: uuid.UUID,
        target_name: str,
        ip: str | None,
        **metadata: Any,
    ) -> AuditLog:
        return AuditLog.objects.create(
            actor=actor,
            action=action,
            target_type=target_type,
            target_id=target_id,
            target_name=target_name,
            ip_address=ip,
            metadata=metadata,
        )
```

**This is the only code path that ever writes an `AuditLog` row.** Services call this
method; nothing else calls `AuditLog.objects.create` directly.

---

### F. `apps/storage/services.py` — FileService methods

`FileService` depends on `AuditService` and the storage backend. Use the factory
`get_storage_backend()` inside each method (not injected at class level) to allow
test overrides via `settings` patching.

#### `FileService.upload`

```python
@staticmethod
def upload(
    *,
    owner: User,
    file_obj: BinaryIO,
    name: str,
    folder: Folder | None,
    ip: str | None,
) -> File:
```

Logic (in order):
1. Detect MIME type from the file content (use `python-magic` or `mimetypes.guess_type`).
2. Try `File.objects.get(owner=owner, folder=folder, name=name, deleted_at__isnull=True)`.
   - If **not found**: create new `File` (current_version=None).
   - If **found**: use the existing `File` for a new version (Phase 5 activates the
     version increment; for now a second upload to the same name raises
     `IntegrityError` — the DB constraint is the gate. Phase 5 handles the version
     path). **Important:** for Phase 3, treat same-name upload as a new version = 1
     for a fresh file. The re-upload scenario is a Phase 5 concern.
3. Derive `version_number = 1` (Phase 5 will compute `max + 1`).
4. Derive `storage_key = f"{owner.id}/{file_id}/1/{name}"`.
5. Call `storage_backend.save(storage_key, file_obj)`.
6. Get `size_bytes = storage_backend.size(storage_key)`.
7. Create `FileVersion(file=file, version_number=1, storage_key=..., size_bytes=..., created_by=owner)`.
8. Set `file.current_version = version` and `file.save()`.
9. Call `AuditService.append(actor=owner, action=AuditAction.UPLOAD, target_type='file', target_id=file.id, target_name=file.name, ip=ip)`.
10. Return `file`.

#### `FileService.download`

```python
@staticmethod
def download(
    *,
    user: User,
    file_id: uuid.UUID,
    ip: str | None,
) -> tuple[File, FileVersion, BinaryIO]:
```

Return type is `(File, FileVersion, BinaryIO)`. The `File` object is returned so
the view can access `file.name` and `file.mime_type` directly without an additional
query or FK traversal.

Logic:
1. Get `file = File.objects.select_related('current_version').get(pk=file_id, deleted_at__isnull=True)` — raise
   `NotFound` if missing.
2. **Permission check:** `PermissionService.require(user, file, PermissionLevel.VIEWER)`
   — **Note:** `PermissionService` is implemented in Phase 4. In Phase 3, apply a
   simpler inline ownership check: `if file.owner != user: raise PermissionDenied`.
   This inline check must be replaced by `PermissionService.require` in Phase 4 without
   changing the method signature.
3. Retrieve stream: `storage_backend.retrieve(file.current_version.storage_key)`.
4. `AuditService.append(actor=user, action=AuditAction.DOWNLOAD, ...)`.
5. Return `(file, file.current_version, stream)`.

#### `FileService.soft_delete`

```python
@staticmethod
def soft_delete(
    *,
    user: User,
    file_id: uuid.UUID,
    ip: str | None,
) -> None:
```

Logic:
1. Fetch file (active only).
2. Ownership check (same Phase 3 inline pattern; replaced by `PermissionService` in
   Phase 4 with `minimum_level=PermissionLevel.OWNER`).
3. `file.deleted_at = timezone.now(); file.save()`.
4. `AuditService.append(actor=user, action=AuditAction.DELETE, ...)`.

#### `FileService.list`

```python
@staticmethod
def list(
    *,
    user: User,
    folder_id: uuid.UUID | None,
) -> QuerySet:
```

Returns `File.objects.filter(owner=user, folder_id=folder_id, deleted_at__isnull=True)`.

**Note:** Shared files visible to non-owners are a Phase 4 concern. For Phase 3,
`list` returns only files owned by the requesting user.

---

### G. `apps/storage/serializers.py` — NEW FILE

```python
class FileVersionSerializer(ModelSerializer):
    class Meta:
        model = FileVersion
        fields = ['id', 'version_number', 'size_bytes', 'created_at']

class FileSerializer(ModelSerializer):
    current_version = FileVersionSerializer(read_only=True)

    class Meta:
        model = File
        fields = ['id', 'name', 'mime_type', 'folder', 'current_version',
                  'created_at', 'updated_at']
```

---

### H. `apps/storage/views.py` — NEW FILE

All views are thin. Pattern: validate input → call service → return response envelope.

#### `FileListCreateView` — `GET /files/`, `POST /files/`

- `GET`: call `FileService.list(user=request.user, folder_id=request.query_params.get('folder'))`.
  Return `{"data": FileSerializer(queryset, many=True).data}`.
- `POST`: expects `multipart/form-data`. Fields: `file` (required), `folder` (optional UUID).
  Validate with a `UploadSerializer`. Call `FileService.upload(...)`. Return 201.

```python
class UploadSerializer(serializers.Serializer):
    file = serializers.FileField()
    folder = serializers.UUIDField(required=False, allow_null=True)
    name = serializers.CharField(required=False, max_length=255)
    # If 'name' is omitted, derive it from file.name in the view.
```

#### `FileDetailView` — `GET /files/{id}/`, `DELETE /files/{id}/`

- `GET`: fetch file metadata (no download). Return `{"data": FileSerializer(file).data}`.
- `DELETE`: call `FileService.soft_delete(...)`. Return 204.

#### `FileDownloadView` — `GET /files/{id}/download/`

```python
def get(self, request, pk):
    version, stream = FileService.download(user=request.user, file_id=pk, ip=get_client_ip(request))
    response = StreamingHttpResponse(stream, content_type=file.mime_type or 'application/octet-stream')
    response['Content-Disposition'] = f'attachment; filename="{file.name}"'
    return response
```

`StreamingHttpResponse` is used even for single-file downloads — it avoids loading the
full file into memory and is consistent with the ZIP streaming pattern in Phase 6.

---

### I. `apps/audit/views.py` and `apps/audit/serializers.py` — NEW FILES

#### `AuditLogSerializer`

```python
class AuditLogSerializer(ModelSerializer):
    actor_email = serializers.EmailField(source='actor.email', read_only=True, allow_null=True)

    class Meta:
        model = AuditLog
        fields = ['id', 'actor_email', 'action', 'target_type', 'target_id',
                  'target_name', 'ip_address', 'metadata', 'created_at']
```

#### `AuditLogListView` — `GET /audit/?target_id={uuid}`

Access rule: the requesting user must be the owner of the target object. For Phase 3,
this means verifying `File.objects.filter(pk=target_id, owner=request.user).exists()`.
Phase 4 will extend this to handle shared files with OWNER permission.

Returns: `{"data": AuditLogSerializer(queryset, many=True).data}` ordered by `-created_at`.

---

### J. URL Registration (`config/urls.py`)

Add to `urlpatterns`:

```python
path("api/v1/files/", include("apps.storage.urls")),
path("api/v1/audit/", include("apps.audit.urls")),
```

#### `apps/storage/urls.py` — NEW FILE

```python
urlpatterns = [
    path("", FileListCreateView.as_view(), name="file-list"),
    path("<uuid:pk>/", FileDetailView.as_view(), name="file-detail"),
    path("<uuid:pk>/download/", FileDownloadView.as_view(), name="file-download"),
]
```

#### `apps/audit/urls.py` — NEW FILE

```python
urlpatterns = [
    path("", AuditLogListView.as_view(), name="audit-list"),
]
```

---

### K. Django Admin (`apps/audit/admin.py`, `apps/storage/admin.py`) — NEW FILES

#### `AuditLogAdmin`

```python
@admin.register(AuditLog)
class AuditLogAdmin(admin.ModelAdmin):
    list_display = ['created_at', 'actor', 'action', 'target_type', 'target_name', 'ip_address']
    search_fields = ['actor__email', 'target_name', 'action']
    list_filter = ['action', 'target_type']
    readonly_fields = [f.name for f in AuditLog._meta.get_fields()]

    def has_add_permission(self, request): return False
    def has_change_permission(self, request, obj=None): return False
    def has_delete_permission(self, request, obj=None): return False
```

#### `FileVersionAdmin`

```python
@admin.register(FileVersion)
class FileVersionAdmin(admin.ModelAdmin):
    list_display = ['file', 'version_number', 'size_bytes', 'created_by', 'created_at']
    search_fields = ['file__name', 'created_by__email']
    readonly_fields = [f.name for f in FileVersion._meta.get_fields()]

    def has_add_permission(self, request): return False
    def has_change_permission(self, request, obj=None): return False
    def has_delete_permission(self, request, obj=None): return False
```

---

### L. Settings Update (`config/settings/base.py`)

Add to `base.py`:

```python
# File upload size limit: 100MB
DATA_UPLOAD_MAX_MEMORY_SIZE = 104857600
FILE_UPLOAD_MAX_MEMORY_SIZE = 104857600
```

No new `INSTALLED_APPS` entries needed — `apps.storage` and `apps.audit` are already
registered.

---

### M. Dependencies (`pyproject.toml`)

Add to the `dependencies` array:
- `python-magic>=0.4.27` — MIME type detection from file content (more reliable than
  file extension inference). Requires `libmagic` in the Docker image (`apt-get install libmagic1`).

**Alternative (no system dep):** use `mimetypes.guess_type(name)` based on filename.
Simpler, no system package needed. Acceptable for this phase — MIME type is metadata,
not a security gate. Use `mimetypes.guess_type` and move to `python-magic` only if
MIME correctness becomes a Phase 9 polish item.

**Decision:** Use `mimetypes.guess_type` in Phase 3. Document in `DESIGN_DECISIONS.md`
as a known limitation.

---

### N. Helper: IP Extraction (`apps/core/utils.py`) — NEW FILE

```python
def get_client_ip(request: HttpRequest) -> str | None:
    x_forwarded_for = request.META.get("HTTP_X_FORWARDED_FOR")
    if x_forwarded_for:
        return x_forwarded_for.split(",")[0].strip()
    return request.META.get("REMOTE_ADDR")
```

Used by all views that pass `ip` to service methods.

---

## 4. Migration Plan

Two new migrations required:

1. `apps/storage/migrations/0001_initial.py` — creates `Folder`, `File`, `FileVersion`
   tables. Must include the partial `UniqueConstraint` on `File`.
2. `apps/audit/migrations/0001_initial.py` — creates `AuditLog` table.

No changes to `apps/accounts` migrations.

**Circular FK resolution:** `File.current_version → FileVersion` and
`FileVersion.file → File` create a circular dependency at the Django model level. Django
resolves this through the `null=True` on `current_version`. Migration generation:
Django will emit both models in the same migration file (same app) with the `File`
model created first with `current_version=None`, then `FileVersion`, then the FK
from `File` to `FileVersion` added via `AlterField`. Verify this with `makemigrations
--check` before proceeding.

---

## 5. Seed Script (`make seed` v1)

**Location:** `apps/core/management/commands/seed.py`

**Scope (Phase 3 v1):**
- 1 user: `demo@vaultshare.io` / `demo1234` (idempotent — skip if exists)
- 5 files in root (no folder): mix of `.txt`, `.pdf`, `.png` types
- Each upload flows through `FileService.upload` → generates real `FileVersion` and
  `AuditLog` entries
- 1 manual `AuditService.append` call for a `DOWNLOAD` event per file

Use `io.BytesIO` with generated content (lorem-ipsum-style bytes). No real files
needed.

**Update contract:** This command will be extended in Phases 4–7. Use `--reset` flag
to wipe and re-seed cleanly. The idempotent path (no `--reset`) is for quick restarts.

---

## 6. Acceptance Tests (`tests/acceptance/test_core.py`)

**Location:** `tests/acceptance/test_core.py`

Uses `httpx` against `http://localhost:8000`. Assumes `make seed` has run.

```python
# Setup fixture: login as demo@vaultshare.io, return Bearer token
# Scenario 1: Upload → Download round-trip
def test_upload_download_roundtrip(auth_headers):
    content = b"vaultshare acceptance test payload"
    response = httpx.post(
        "http://localhost:8000/api/v1/files/",
        headers=auth_headers,
        files={"file": ("test.txt", content, "text/plain")},
    )
    assert response.status_code == 201
    file_id = response.json()["data"]["id"]

    dl = httpx.get(f"http://localhost:8000/api/v1/files/{file_id}/download/", headers=auth_headers)
    assert dl.status_code == 200
    assert dl.content == content

# Scenario 2: Soft delete → absent from listing → still in DB
def test_soft_delete(auth_headers):
    # upload
    # delete → 204
    # GET /files/ → file not present in list
    # (DB check is done by the seed/unit tests — acceptance tests verify HTTP boundary only)

# Scenario 3: Upload → AuditLog entry exists
def test_audit_log_on_upload(auth_headers):
    # upload a file
    # GET /api/v1/audit/?target_id={file_id}
    # assert at least one entry with action == 'UPLOAD'
```

**`make verify` wiring:** Add `tests/acceptance/` to `pytest.ini` or to a dedicated
`make verify` target that runs `pytest tests/acceptance/ -v` against the live stack.
This is separate from `make test` (which runs the full suite including unit/integration
against the test DB).

---

## 7. Tests — Unit & Integration

### Unit tests (`tests/unit/test_audit.py`) — NEW FILE

- `test_audit_log_is_append_only`: Create an `AuditLog`, attempt `log.save()` with
  modified field — assert `ImproperlyConfigured` raised.
- `test_audit_log_delete_raises`: Create an `AuditLog`, call `log.delete()` — assert
  `ImproperlyConfigured` raised.
- `test_audit_log_queryset_update_raises`: Call `AuditLog.objects.filter(...).update(action='DELETE')` — assert `ImproperlyConfigured` raised.
- `test_audit_log_queryset_delete_raises`: Call `AuditLog.objects.filter(...).delete()` — assert `ImproperlyConfigured` raised.

### Integration tests (`tests/integration/test_files.py`) — NEW FILE

- `test_upload_creates_file_version_and_audit`: Upload via API → assert `File`, `FileVersion`, and `AuditLog` (action=`UPLOAD`) all exist in DB.
- `test_download_returns_correct_bytes`: Upload bytes B → download → assert response body == B.
- `test_soft_delete_sets_deleted_at`: Delete via API → assert `file.deleted_at` is not None, `file` still in DB.
- `test_deleted_file_absent_from_list`: Upload → delete → `GET /files/` → assert not in response list.
- `test_non_owner_cannot_download`: Create user A and user B. User A uploads. User B attempts download → assert 403.
  *(This establishes the ownership gate before Phase 4's full permission system.)*
- `test_audit_log_on_download`: Upload → download → `GET /audit/?target_id={id}` → assert `DOWNLOAD` entry exists.

### Storage unit tests (`tests/storage/test_backends.py`) — already exists ✓

No changes needed.

---

## 8. Sequence of Implementation Steps

The implementer must follow this sequence to avoid circular import issues:

1. `apps/core/models.py` — `BaseModel`
2. `apps/audit/models.py` — `AuditAction`, `AuditLog`, `AuditLogQuerySet`, `AuditLogManager`
3. `apps/audit/services.py` — `AuditService.append`
4. `apps/storage/models.py` — `Folder`, `File`, `FileVersion`
5. `apps/storage/services.py` — `get_storage_backend` factory, then `FileService`
6. `apps/core/utils.py` — `get_client_ip`
7. `apps/storage/serializers.py`
8. `apps/storage/views.py`
9. `apps/storage/urls.py`
10. `apps/audit/serializers.py`
11. `apps/audit/views.py`
12. `apps/audit/urls.py`
13. `apps/storage/admin.py`, `apps/audit/admin.py`
14. `config/urls.py` — add includes
15. `config/settings/base.py` — add upload size limits
16. `makemigrations` + `migrate`
17. `apps/core/management/commands/seed.py`
18. `tests/unit/test_audit.py`
19. `tests/integration/test_files.py`
20. `tests/acceptance/test_core.py`
21. `make check` (ruff + mypy) — must pass clean
22. `make verify` — must be green on all 3 acceptance scenarios

---

## 9. What Is Deliberately Out of Scope for Phase 3

| Feature | Phase |
|---|---|
| PermissionService (shared files, VIEWER/EDITOR/OWNER) | 4 |
| Version increment on same-name re-upload | 5 |
| Folder creation and listing | 6 |
| ZIP download | 6 |
| Public links | 7 |
| `GET /files/{id}/versions/` endpoint | 5 |
| `storage_used_bytes` tracking | 9 (Bonus B) |

The Phase 3 ownership checks (inline `if file.owner != user`) are **intentional
placeholders**. They must be replaced by `PermissionService.require` in Phase 4 without
changing method signatures.

---

## 10. Consistency Notes

- **Response envelope:** All views return `{"data": ...}` on success and
  `{"error": {"code": "...", "message": "..."}}` on error. This matches the pattern
  already established in `apps/accounts/views.py`.
- **`AuditService.append` signature:** Uses keyword-only arguments (`*`) to prevent
  argument-order bugs. All callers must use named args.
- **`AuditLog` does not inherit `BaseModel`** — it must not have `updated_at`. This is
  the only model that deviates from `BaseModel`.
- **Storage key format:** `{owner.id}/{file.id}/1/{file.name}`. In Phase 5, `1` becomes
  `{version.version_number}`. The format is unchanged; only the version segment varies.
- **`make check` gate:** `ruff check` + `mypy --strict` must pass with zero errors after
  each step. Do not defer linting to end of phase.
