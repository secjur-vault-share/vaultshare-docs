# Phase 5 — Versioning: Detailed Implementation Blueprint

This blueprint dictates the exact technical specifications for Phase 5 implementation.
It dictates the architectural boundaries, service interfaces, endpoint contracts, and
test requirements. No code will be written until this blueprint is approved.

---

## 1. Scope & Deliverables

Phase 5 activates the full versioning feature. The core versioning data model (`File`,
`FileVersion`, `FileService.upload` with version bump logic) and `AuditAction.VERSION_CREATED`
were already built and unit-tested in Phase 3. The upload service in `apps/storage/services.py`
already detects duplicate filenames and increments `version_number` correctly.

**What Phase 5 adds on top:**

- `FileService.list_versions(user, file_id)` — new service method
- `FileService.download_version(user, file_id, version_number, ip)` — new service method
- `GET /api/v1/files/{id}/versions/` — list all versions of a file
- `GET /api/v1/files/{id}/versions/{n}/download/` — download a specific version
- `AuditAction.VERSION_DOWNLOADED` — new audit action for specific version access
- Extended `FileSerializer` to expose `version_count` (optimized via annotation)
- Updated `make seed` v3: 3 files with 2+ versions each, audit entries per version bump
- `tests/acceptance/test_versioning.py` — 3 new acceptance scenarios (cumulative: 9)
- New unit and integration tests for the version download path

**Working state at phase end:** `make up && make seed && make verify` → green on 9 scenarios.

---

## 2. Prerequisite Checks

Before writing any code, verify:

1. `FileService.upload` in `apps/storage/services.py` already contains the versioning
   branch: detects `existing_file` by `(name, owner, folder, deleted_at__isnull=True)`,
   sets `action = AuditAction.VERSION_CREATED`, and bumps `version_number` correctly.
   **This is already in place — do not touch it.**

2. The `File.versions` reverse relation (`related_name='versions'` on `FileVersion.file`)
   exists and is functional. Confirm with `file.versions.count()` in the existing
   unit test `apps/storage/tests/test_versioning.py`. **Already in place.**

3. `AuditAction.VERSION_CREATED` and `AuditAction.VERSION_DOWNLOADED` are defined in `apps/audit/models.py`. **Need to add VERSION_DOWNLOADED.**

4. `PermissionService.require` is imported and called with `PermissionLevel.VIEWER` in
   the existing `FileService.download` and `FileDetailView.get`. All new download paths
   must follow the same pattern.

5. The `FileVersionSerializer` in `apps/storage/serializers.py` already serialises
   `[id, version_number, size_bytes, created_at]`. It will be reused directly.

6. **Refactor `FileService.upload`** to handle versioning in shared folders by checking for
   any existing file with the same name in the target folder and verifying permissions.

---

## 3. Strict File-by-File Technical Plan

### A. `FileService` — new methods (`apps/storage/services.py`) — MODIFY

Both new methods follow the exact same pattern as `FileService.download`: fetch → permission
check → act → audit. **No model changes. No new migrations.**

#### `FileService.list_versions`

```python
@staticmethod
def list_versions(
    *,
    user: User,
    file_id: uuid.UUID,
) -> models.QuerySet[FileVersion]:
    """
    Returns all versions of a file, ordered by version_number ascending.
    Requires at least VIEWER permission on the file.
    Does NOT write an audit entry — listing versions is a read-only metadata
    operation equivalent to loading file metadata.
    """
    try:
        file = File.objects.get(pk=file_id, deleted_at__isnull=True)
    except File.DoesNotExist:
        raise NotFound("File not found.")

    PermissionService.require(user, file, PermissionLevel.VIEWER)

    return file.versions.order_by("version_number")
```

**Design rationale:** No audit entry for `list_versions`. The architecture's audit actions
(`UPLOAD`, `DOWNLOAD`, `DELETE`, `SHARE`, `REVOKE`, `VERSION_CREATED`, `LINK_GENERATED`,
`LINK_ACCESSED`, `PERMISSION_CHANGED`) do not include a `LIST_VERSIONS` action. Listing
version metadata is analogous to `GET /files/{id}/` — a metadata read, not a content
access event. Only `download_version` (actual content retrieval) generates an audit entry.

#### `FileService.download_version`

```python
@staticmethod
def download_version(
    *,
    user: User,
    file_id: uuid.UUID,
    version_number: int,
    ip: str | None,
) -> tuple[File, FileVersion, BinaryIO]:
    """
    Downloads a specific version of a file by version number.
    Requires at least VIEWER permission on the file.
    Emits AuditAction.VERSION_DOWNLOADED.
    """
    try:
        file = File.objects.get(pk=file_id, deleted_at__isnull=True)
    except File.DoesNotExist:
        raise NotFound("File not found.")

    PermissionService.require(user, file, PermissionLevel.VIEWER)

    try:
        version = file.versions.get(version_number=version_number)
    except FileVersion.DoesNotExist:
        raise NotFound(f"Version {version_number} not found.")

    backend = get_storage_backend()
    stream = backend.retrieve(version.storage_key)

    AuditService.append(
        actor=user,
        action=AuditAction.VERSION_DOWNLOADED,
        target_type="file",
        target_id=file.id,
        target_name=file.name,
        ip=ip,
        version_number=version_number,
    )

    return file, version, stream
```

**Audit metadata:** The `AuditAction.VERSION_DOWNLOADED` action explicitly distinguishes
this from a standard `DOWNLOAD` of the current version. The `version_number` context
is also preserved in metadata for additional granularity.

#### `FileService.list` (Performance Update)

Update the existing `list` method to include an annotation for `version_count` to avoid N+1 queries.

```python
@staticmethod
def list(
    *,
    user: User,
    folder_id: uuid.UUID | str | None,
) -> models.QuerySet[File]:
    # ... (existing selection logic)
    # ...
    return qs.select_related("current_version").annotate(
        version_count=models.Count("versions")
    )
```

#### `FileService.upload` (Shared Folder Update)

Refactor `upload` to look for existing files by name and folder, regardless of owner,
enabling versioning for editors in shared folders.

```python
# Updated logic inside upload():
existing_file = File.objects.filter(
    name=safe_name,
    folder=folder,
    deleted_at__isnull=True,
).first()

if existing_file:
    # If file exists, the uploader MUST have EDITOR permission on THAT specific file
    # to create a new version.
    PermissionService.require(owner, existing_file, PermissionLevel.EDITOR)
    file = existing_file
    last_version = file.current_version
    version_number = (last_version.version_number + 1) if last_version else 1
    action = AuditAction.VERSION_CREATED
else:
    # New file
    file = File.objects.create(...)
```

**NotFound on soft-deleted file vs. missing version:** The file fetch uses
`deleted_at__isnull=True`, which means downloading a version of a soft-deleted file
returns 404. This is intentional — soft-deleted files are invisible to users. The version
fetch has no `deleted_at` filter because `FileVersion` has no soft-delete; version rows
are permanent records (they survive `File.soft_delete` and serve as the audit trail of
what was uploaded).

---

### B. Serializers (`apps/storage/serializers.py`) — MODIFY

#### Extend `FileSerializer` with `version_count`

Add a `version_count` IntegerField to `FileSerializer`. This uses the annotated value
from the queryset, ensuring O(1) performance in list views.

```python
class FileSerializer(serializers.ModelSerializer):
    current_version = FileVersionSerializer(read_only=True)
    version_count = serializers.IntegerField(read_only=True)

    class Meta:
        model = File
        fields = [
            "id",
            "name",
            "mime_type",
            "folder",
            "current_version",
            "version_count",
            "created_at",
            "updated_at",
        ]
```

**Performance note:** By using `annotate(version_count=Count('versions'))` in the
`FileService.list` method, we eliminate the N+1 issue. The serializer now simply
reads the pre-fetched attribute.

`FileVersionSerializer` remains unchanged.

---

### C. Views (`apps/storage/views.py`) — MODIFY

Add two new view classes. Both are thin: validate URL parameters, call the service,
return the envelope.

#### `FileVersionListView` — `GET /files/{id}/versions/`

```python
class FileVersionListView(APIView):
    def get(self, request: Request, pk: uuid.UUID) -> Response:
        versions = FileService.list_versions(
            user=request.user,
            file_id=pk,
        )
        return Response({"data": FileVersionSerializer(versions, many=True).data})
```

No input validation beyond `pk` (type-enforced by the URL router via `<uuid:pk>`).
Returns an ordered list of all versions.

#### `FileVersionDownloadView` — `GET /files/{id}/versions/{n}/download/`

```python
class FileVersionDownloadView(APIView):
    def get(self, request: Request, pk: uuid.UUID, version_number: int) -> StreamingHttpResponse:
        file, version, stream = FileService.download_version(
            user=request.user,
            file_id=pk,
            version_number=version_number,
            ip=get_client_ip(request),
        )
        response = StreamingHttpResponse(
            stream, content_type=file.mime_type or "application/octet-stream"
        )
        response["Content-Disposition"] = f'attachment; filename="{file.name}"'
        return response
```

The `version_number` URL parameter is an integer. The URL pattern uses `<int:version_number>`
to enforce this at the routing layer — no additional validation needed in the view.

---

### D. URL Registration (`apps/storage/urls.py`) — MODIFY

Add two new patterns to the existing `urlpatterns`:

```python
from .views import (
    FileDetailView,
    FileDownloadView,
    FileListCreateView,
    FileVersionListView,
    FileVersionDownloadView,
)

urlpatterns = [
    path("", FileListCreateView.as_view(), name="file-list"),
    path("<uuid:pk>/", FileDetailView.as_view(), name="file-detail"),
    path("<uuid:pk>/download/", FileDownloadView.as_view(), name="file-download"),
    path("<uuid:pk>/versions/", FileVersionListView.as_view(), name="file-version-list"),
    path(
        "<uuid:pk>/versions/<int:version_number>/download/",
        FileVersionDownloadView.as_view(),
        name="file-version-download",
    ),
]
```

No changes to `config/urls.py` — the storage URL include already covers these paths.

---

### E. Seed Script Update (`apps/core/management/commands/seed_db.py`) — MODIFY

**Scope (Phase 5 v3) — extend existing seed, do not rewrite.**

The Phase 4 seed creates 3 users and 5 files (all at version 1). Phase 5 must create
additional versions for 3 of those files to exercise versioning in the acceptance tests.

Add a `_seed_versions` method called after `_seed_files` and `_seed_shares`:

```python
VERSIONED_FILES = [
    # (owner_email, filename, [version_contents])
    ("demo@vaultshare.io", "readme.txt", [
        b"Welcome to VaultShare. This is version 2 of the readme.",
        b"Welcome to VaultShare. Version 3 — final draft.",
    ]),
    ("demo@vaultshare.io", "report.pdf", [
        b"%PDF-1.4 updated report content — version 2",
    ]),
    ("alice@vaultshare.io", "data.csv", [
        b"id,name,value\n1,alpha,110\n2,beta,210\n3,gamma,310\n4,delta,400",
    ]),
]
```

```python
def _seed_versions(self, users: dict[str, User]) -> None:
    from apps.storage.models import File

    for owner_email, filename, version_contents in VERSIONED_FILES:
        owner = users[owner_email]
        file = File.objects.filter(
            owner=owner, name=filename, deleted_at__isnull=True
        ).first()
        if file is None:
            self.stdout.write(f"  Skipping versions: {filename} not found for {owner_email}")
            continue
        for content in version_contents:
            FileService.upload(
                owner=owner,
                file_obj=io.BytesIO(content),
                name=filename,
                folder=None,
                ip="127.0.0.1",
            )
            self.stdout.write(
                f"  Uploaded new version of: {filename} for {owner_email}"
            )
```

**Idempotency:** `FileService.upload` detects existing filenames by name + owner + folder
and creates a new `FileVersion` each time. If the seed is re-run (without `--reset`), it
will add *additional* version rows for each repeated call. To make `_seed_versions`
idempotent, wrap each version upload in a check:

```python
existing_count = file.versions.count()
expected_count = len(version_contents) + 1  # +1 for initial upload from phase 4
if existing_count >= expected_count:
    self.stdout.write(f"  Versions already seeded for: {filename}")
    continue
```

**`--reset` scope:** `_reset()` already deletes all `FileVersion` objects, so seeded
versions are cleaned up automatically on `--reset`.

---

### F. Acceptance Tests (`tests/acceptance/test_versioning.py`) — NEW FILE

**Location:** `tests/acceptance/test_versioning.py`

Uses `httpx` against `http://localhost:8000`. Assumes `make seed` has run (which seeds
`readme.txt` for `demo@vaultshare.io` with 3 versions, per Phase 5 seed above).

**Setup fixture:** Reuse the `get_auth_headers` pattern from `test_permissions.py`.

```python
import io
import httpx
import pytest

BASE_URL = "http://localhost:8000"


def get_auth_headers(email: str, password: str) -> dict[str, str]:
    response = httpx.post(
        f"{BASE_URL}/api/v1/auth/login/",
        json={"email": email, "password": password},
    )
    assert response.status_code == 200, f"Login failed for {email}: {response.text}"
    token = response.json()["data"]["access"]
    return {"Authorization": f"Bearer {token}"}


@pytest.fixture(scope="module")
def demo_headers() -> dict[str, str]:
    return get_auth_headers("demo@vaultshare.io", "demo1234")
```

#### Scenario 7: Same-name upload creates v2

```python
def test_same_name_upload_creates_new_version(demo_headers: dict[str, str]) -> None:
    """
    Upload test.txt → upload test.txt again → assert versions.count == 2.
    Uses a fresh unique filename to avoid interference with seed data.
    """
    import uuid
    filename = f"versioning_test_{uuid.uuid4().hex[:8]}.txt"

    # Upload v1
    v1_content = b"version 1 content"
    upload1 = httpx.post(
        f"{BASE_URL}/api/v1/files/",
        headers=demo_headers,
        files={"file": (filename, io.BytesIO(v1_content), "text/plain")},
    )
    assert upload1.status_code == 201
    file_id = upload1.json()["data"]["id"]
    assert upload1.json()["data"]["current_version"]["version_number"] == 1

    # Upload v2 (same name)
    v2_content = b"version 2 content — different bytes"
    upload2 = httpx.post(
        f"{BASE_URL}/api/v1/files/",
        headers=demo_headers,
        files={"file": (filename, io.BytesIO(v2_content), "text/plain")},
    )
    assert upload2.status_code == 201
    assert upload2.json()["data"]["id"] == file_id  # same file, not a new one
    assert upload2.json()["data"]["current_version"]["version_number"] == 2

    # Confirm version list
    versions_resp = httpx.get(
        f"{BASE_URL}/api/v1/files/{file_id}/versions/",
        headers=demo_headers,
    )
    assert versions_resp.status_code == 200
    assert len(versions_resp.json()["data"]) == 2
```

#### Scenario 8: Download v1 → assert original bytes; download v2 → assert new bytes

```python
def test_version_specific_download_returns_correct_bytes(
    demo_headers: dict[str, str],
) -> None:
    """
    Upload two distinct versions. Download v1 → assert original bytes.
    Download v2 → assert updated bytes.
    """
    import uuid
    filename = f"bytes_test_{uuid.uuid4().hex[:8]}.txt"

    v1_content = b"this is the original v1 payload"
    v2_content = b"this is the updated v2 payload — completely different"

    upload1 = httpx.post(
        f"{BASE_URL}/api/v1/files/",
        headers=demo_headers,
        files={"file": (filename, io.BytesIO(v1_content), "text/plain")},
    )
    file_id = upload1.json()["data"]["id"]

    httpx.post(
        f"{BASE_URL}/api/v1/files/",
        headers=demo_headers,
        files={"file": (filename, io.BytesIO(v2_content), "text/plain")},
    )

    dl_v1 = httpx.get(
        f"{BASE_URL}/api/v1/files/{file_id}/versions/1/download/",
        headers=demo_headers,
    )
    assert dl_v1.status_code == 200
    assert dl_v1.content == v1_content

    dl_v2 = httpx.get(
        f"{BASE_URL}/api/v1/files/{file_id}/versions/2/download/",
        headers=demo_headers,
    )
    assert dl_v2.status_code == 200
    assert dl_v2.content == v2_content

    # Confirm the two payloads are distinct (guard against a backend key collision bug)
    assert dl_v1.content != dl_v2.content
```

#### Scenario 9: `VERSION_CREATED` audit entry exists after second upload

```python
def test_version_created_audit_entry_written(demo_headers: dict[str, str]) -> None:
    """
    Upload a file twice. Assert a VERSION_CREATED audit entry exists for the second upload.
    """
    import uuid
    filename = f"audit_version_{uuid.uuid4().hex[:8]}.txt"

    upload1 = httpx.post(
        f"{BASE_URL}/api/v1/files/",
        headers=demo_headers,
        files={"file": (filename, io.BytesIO(b"v1"), "text/plain")},
    )
    file_id = upload1.json()["data"]["id"]

    httpx.post(
        f"{BASE_URL}/api/v1/files/",
        headers=demo_headers,
        files={"file": (filename, io.BytesIO(b"v2"), "text/plain")},
    )

    audit_resp = httpx.get(
        f"{BASE_URL}/api/v1/audit/",
        headers=demo_headers,
        params={"target_id": file_id, "target_type": "file"},
    )
    assert audit_resp.status_code == 200
    actions = [entry["action"] for entry in audit_resp.json()["data"]]
    assert "UPLOAD" in actions         # first upload
    assert "VERSION_CREATED" in actions  # second upload
```

**`make verify` wiring:** Add `tests/acceptance/test_versioning.py` to the test suite.
`make verify` now runs all 9 scenarios (6 from previous phases + 3 new).

---

## 4. Tests — Unit & Integration

### Unit tests (`apps/storage/tests/test_versioning.py`) — MODIFY

The existing `apps/storage/tests/test_versioning.py` covers the upload-side of versioning
(`test_upload_creates_first_version`, `test_upload_same_name_creates_new_version`,
`test_upload_different_folders_unique_files`). Phase 5 adds tests for the **download side**.

Add the following test cases to the existing `TestFileVersioning` class (or in a new
`TestVersionDownload` class in the same file):

- `test_list_versions_returns_all_versions`: Upload file twice. Call
  `FileService.list_versions(user=user, file_id=file.id)` → assert queryset count is 2
  and version numbers are `[1, 2]` in order.

- `test_list_versions_requires_viewer_permission`: Upload file as `user_a`. Call
  `FileService.list_versions(user=user_b, file_id=file.id)` without a share →
  assert `PermissionDenied` is raised.

- `test_list_versions_returns_empty_on_no_versions`: This cannot happen for a live file
  (upload always creates v1), but verify that a freshly-created file (through the model
  directly, before upload) returns an empty queryset without crashing.

- `test_download_version_returns_correct_stream`: Upload v1 (`b"v1 bytes"`), upload
  v2 (`b"v2 bytes"`). Call `FileService.download_version(..., version_number=1)` →
  read the stream → assert `b"v1 bytes"`. Repeat for `version_number=2` →
  assert `b"v2 bytes"`.

- `test_download_version_emits_audit_with_version_number`: Upload v1, upload v2.
  Call `FileService.download_version(..., version_number=1, ip="127.0.0.1")` →
  assert `AuditLog.objects.filter(action=AuditAction.DOWNLOAD, target_id=file.id).exists()`.
  Also assert the `metadata` dict of the audit entry contains `{"version_number": 1}`.

- `test_download_version_requires_viewer_permission`: Upload as `user_a`. Call
  `FileService.download_version(user=user_b, ...)` without share →
  assert `PermissionDenied`.

- `test_download_version_not_found_raises_404`: Upload v1 only. Call
  `FileService.download_version(..., version_number=99)` →
  assert `NotFound` raised.

- `test_download_version_on_deleted_file_raises_404`: Upload file, soft-delete it.
  Call `FileService.download_version(...)` → assert `NotFound` raised.

### Integration tests (`tests/integration/test_files.py`) — MODIFY

Add the following integration test cases to the existing file:

- `test_version_list_endpoint`: Upload a file twice via the API. Call
  `GET /api/v1/files/{id}/versions/` → assert status 200 → assert response contains
  2 version objects with `version_number` 1 and 2.

- `test_version_download_endpoint_returns_v1_bytes`: Upload v1 (`b"first"`), upload v2
  (`b"second"`). Call `GET /api/v1/files/{id}/versions/1/download/` →
  assert status 200 → assert response body equals `b"first"`.

- `test_version_download_endpoint_returns_v2_bytes`: Same setup as above. Call
  `GET /api/v1/files/{id}/versions/2/download/` →
  assert status 200 → assert response body equals `b"second"`.

- `test_version_list_returns_403_for_non_owner_without_share`: User A uploads file.
  User B calls `GET /api/v1/files/{id}/versions/` → assert 403.

- `test_version_download_returns_403_for_non_owner_without_share`: User A uploads file.
  User B calls `GET /api/v1/files/{id}/versions/1/download/` → assert 403.

- `test_version_download_returns_404_for_invalid_version`: Upload file (1 version).
  Call `GET /api/v1/files/{id}/versions/99/download/` → assert 404.

- `test_file_serializer_includes_version_count`: Upload a file twice. Call
  `GET /api/v1/files/` → find the file in the response → assert
  `response["version_count"] == 2`.

---

## 5. No New Migrations

Phase 5 introduces no new models and no model field changes. All new functionality
(`list_versions`, `download_version`) operates on the existing `FileVersion` model
and the existing `file.versions` reverse relation.

**Confirm before starting:** Run `python manage.py makemigrations --check`. If it
detects pending migrations, something is wrong — stop and investigate before proceeding.

---

## 6. Error Codes

No new error codes are introduced. All error scenarios use existing codes:

| Condition | HTTP status | `code` |
|---|---|---|
| File not found (deleted or non-existent) | 404 | `NOT_FOUND` |
| Requested version number does not exist | 404 | `NOT_FOUND` |
| User has no read access to the file | 403 | `PERMISSION_DENIED` |

These are already handled by the global exception handler in `apps/core/exceptions.py`
with no changes required.

---

## 7. Sequence of Implementation Steps

The implementer must follow this sequence to keep each step independently verifiable:

1. **`apps/storage/services.py`** — Add `FileService.list_versions` and
   `FileService.download_version` methods. Run existing unit tests after each addition:
   `pytest apps/storage/tests/ -x`.

2. **`apps/storage/serializers.py`** — Add `version_count` field to `FileSerializer`.
   Confirm the existing test `test_upload_creates_file_version_and_audit` still passes
   (the new field is read-only and non-breaking).

3. **`apps/storage/views.py`** — Add `FileVersionListView` and
   `FileVersionDownloadView`.

4. **`apps/storage/urls.py`** — Register the two new URL patterns.

5. **Manual smoke test:** `curl -H "Authorization: Bearer <token>" http://localhost:8000/api/v1/files/<id>/versions/`
   → should return 200 with the version list.

6. **`apps/storage/tests/test_versioning.py`** — Add unit tests for the new service
   methods (Section 4 above). Run: `pytest apps/storage/tests/test_versioning.py -x`.

7. **`tests/integration/test_files.py`** — Add integration tests for the new endpoints
   (Section 4 above). Run: `pytest tests/integration/ -x`.

8. **`apps/core/management/commands/seed_db.py`** — Add `VERSIONED_FILES` constant and
   `_seed_versions` method. Wire `_seed_versions` call into `handle`. Run
   `python manage.py seed_db --reset` and verify seed output.

9. **`tests/acceptance/test_versioning.py`** — Write the 3 acceptance scenarios.

10. **`make check`** — `ruff check` + `mypy --strict` must pass with zero errors.

11. **`make verify`** — Must be green on all 9 scenarios.

---

## 8. What Is Deliberately Out of Scope for Phase 5

| Feature | Phase |
|---|---|
| Folder CRUD endpoints | 6 |
| `FolderService.walk_tree` | 6 |
| ZIP download | 6 |
| FolderShare inheritance in `FolderService.list` | 6 |
| Public links | 7 |
| Version deletion / pruning | Future |
| Version restore ("roll back to v1 as current") | Future |
| `version_count` N+1 optimisation (`annotate`) | 6 or Future |

**Deliberate simplification:** There is no endpoint to update `file.current_version`
to point to a previous version (i.e., "rollback"). The current version is always the
latest upload. This is documented as a known UX gap. The `current_version` pointer
always advances forward — it never retreats.

---

## 9. Consistency Notes

- **`PermissionService.require` is mandatory before every file access.** Both
  `list_versions` and `download_version` call
  `PermissionService.require(user, file, PermissionLevel.VIEWER)` before acting.
  The same minimum level applies to both operations: any user with at least VIEWER
  access can list and download any version. This is intentional — if a user can read
  the current version of a file, they should be able to read its history.

- **Storage key format is version-specific.** The storage key
  `{owner_id}/{file_id}/{version_number}/{original_name}` (defined in `FileService.upload`)
  ensures that each version's bytes are stored independently. `download_version` retrieves
  bytes using `version.storage_key` directly — no path reconstruction needed.

- **`download_version` emits `AuditAction.VERSION_DOWNLOADED`.** This new action
  provides explicit tracking of historical data access, distinct from current version access.
  The `version_number` context is also preserved in the `metadata` JSONField.

- **`FileVersionSerializer` is unchanged.** The version list endpoint reuses the
  existing serializer (`[id, version_number, size_bytes, created_at]`). No `created_by`
  field is exposed in the API response — the version is created by the owner or an
  editor, and the audit log is the proper place to record who created it.

- **Response envelope:** All new views return `{"data": ...}` on success. The global
  exception handler covers error responses. No manual error wrapping in views.

- **`make check` gate:** `ruff check` + `mypy --strict` must pass with zero errors after
  each step. Do not defer linting to end of phase.
