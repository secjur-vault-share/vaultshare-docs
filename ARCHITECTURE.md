# VaultShare — System Architecture & Implementation Plan

## 0. Constraints & Time Budget

| Budget | Allocation |
|---|---|
| 12h planned | Full feature set including frontend |
| 4h buffer | Debugging, polish, bonus scaffolding |
| Hard stop | If Phase 7 is done, all scored features are covered |

The plan has 9 phases. Phases 1–7 cover everything the reviewers score. Phase 8 is the
frontend. Phase 9 is verification + bonus scaffolding. Each phase ends in a working,
shippable state.

---

## 1. Multi-Repo Structure

| Repository | Purpose | Required |
|---|---|---|
| `vaultshare-api` | Django backend | Yes |
| `vaultshare-frontend` | Vue 3 + TypeScript | Yes |
| `vaultshare-infra` | Docker Compose, Makefile, shared env | Recommended |

The `docker-compose.yml` references both API and frontend and should not live in either
service repo. The infra repo is the composition root. Each repo has its own `README.md`.
The infra `README.md` is the reviewer entry point.

---

## 2. Backend Architecture

### 2.1 Django Project Layout (Modular Monolith)

Domain-driven app boundaries. Each app is a bounded context: it owns its models,
services, serializers, views, URLs, and tests. No cross-app ORM imports — apps
communicate through service calls only.

```
vaultshare-api/
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   └── urls.py
├── apps/
│   ├── core/        # BaseModel, custom exceptions, base permissions, health check
│   ├── accounts/    # User model, JWT auth
│   ├── storage/     # File, Folder, FileVersion + storage backends
│   ├── sharing/     # FileShare, FolderShare, PublicLink
│   └── audit/       # AuditLog (append-only)
├── Makefile
├── pyproject.toml           # ruff + mypy configuration
├── pytest.ini
├── IMPLEMENTATION.md        # Agent Playbook: implementation persona + rules
├── TESTING.md               # Agent Playbook: testing strategy + prompts
├── VERIFICATION.md
└── CODE_REVIEW.md           # Agent Playbook: review persona + output format
```

Rule: views are thin — validate input, call a service, return the response. All business
logic lives in services. This makes services independently testable.

### 2.2 Data Models

**`core` — abstract base**
```
BaseModel (abstract)
  id: UUIDField(primary_key=True, default=uuid4)
  created_at: DateTimeField(auto_now_add=True)
  updated_at: DateTimeField(auto_now=True)
```

**`accounts`**
```
User (extends AbstractUser)
  id: UUID
  email: EmailField(unique=True)           # login field
  username: CharField                       # display name
  storage_quota_bytes: BigIntegerField     # default 500MB (Bonus B scaffold)
  storage_used_bytes: BigIntegerField      # tracked on upload/delete (Bonus B scaffold)
  USERNAME_FIELD = 'email'
```

**`storage`**
```
Folder
  id: UUID
  name: CharField
  owner: FK(User)
  parent: FK(self, null=True)              # Adjacency List
  deleted_at: DateTimeField(null=True)     # null = active; non-null = soft-deleted

File
  id: UUID
  name: CharField
  owner: FK(User)
  folder: FK(Folder, null=True)
  mime_type: CharField
  deleted_at: DateTimeField(null=True)     # null = active; non-null = soft-deleted
  current_version: OneToOneField(FileVersion, null=True, related_name='+')
  # current_version is nullable during initial insert only

  class Meta:
    constraints = [
      models.UniqueConstraint(
        fields=['owner', 'folder', 'name'],
        condition=models.Q(deleted_at__isnull=True),
        name='unique_active_file_per_folder'
      )
    ]
  # DB-level enforcement prevents race conditions when two simultaneous uploads
  # of the same filename hit the versioning check concurrently.

FileVersion
  id: UUID
  file: FK(File, related_name='versions')
  version_number: PositiveIntegerField
  storage_key: CharField                   # opaque key, backend-specific
  size_bytes: BigIntegerField
  created_by: FK(User)
  created_at: DateTimeField(auto_now_add=True)

  class Meta:
    unique_together = ('file', 'version_number')
```

**`sharing`**
```
FileShare
  id: UUID
  file: FK(File)
  shared_with: FK(User, related_name='received_file_shares')
  shared_by: FK(User, related_name='sent_file_shares')
  permission: CharField(choices=['VIEWER','EDITOR','OWNER'])
  revoked_at: DateTimeField(null=True)

FolderShare  # mirrors FileShare
  id: UUID
  folder: FK(Folder)
  shared_with: FK(User)
  shared_by: FK(User)
  permission: CharField(choices=['VIEWER','EDITOR','OWNER'])
  revoked_at: DateTimeField(null=True)

PublicLink
  id: UUID
  token: CharField(unique=True, default=generate_token)  # secrets.token_urlsafe(32)
  file: FK(File)
  created_by: FK(User)
  expires_at: DateTimeField
  is_revoked: BooleanField(default=False)
  access_count: IntegerField(default=0)
```

**`audit`**
```
AuditLog
  id: UUID
  actor: FK(User, null=True)              # null = public link access
  action: CharField(choices=AuditAction)
  # Actions: UPLOAD, DOWNLOAD, DELETE, SHARE, REVOKE, VERSION_CREATED,
  #          LINK_GENERATED, LINK_ACCESSED, PERMISSION_CHANGED
  target_type: CharField                  # 'file', 'folder', 'public_link'
  target_id: UUIDField
  target_name: CharField                  # denormalized — survives soft delete
  ip_address: GenericIPAddressField
  metadata: JSONField(default=dict)
  created_at: DateTimeField(auto_now_add=True)
  # NO updated_at field

  Immutability enforced at model level:
    def save(self, *args, **kwargs):
        if self.pk:
            raise ImproperlyConfigured("AuditLog is append-only.")
        super().save(*args, **kwargs)

    def delete(self, *args, **kwargs):
        raise ImproperlyConfigured("AuditLog entries cannot be deleted.")

  Custom QuerySet also overrides .update() and .delete() to raise.
```

### 2.3 Service Layer

One service class per concern. Services receive dependencies via constructor injection.

```
accounts.services.UserService
  - register(email, password, username) -> User
  - authenticate(email, password) -> User

storage.services.FileService
  - upload(owner, file_obj, name, folder, ip) -> File
  - download(user, file_id, ip) -> (FileVersion, IO)
  - download_version(user, file_id, version_number, ip) -> (FileVersion, IO)
  - soft_delete(user, file_id, ip) -> None
  - list(user, folder_id) -> QuerySet[File]
  - list_versions(user, file_id) -> QuerySet[FileVersion]

storage.services.FolderService
  - create(owner, name, parent_id) -> Folder
  - list(owner, parent_id) -> QuerySet[Folder]
  - soft_delete(user, folder_id, ip) -> None
  - download_as_zip(user, folder_id, ip) -> StreamingHttpResponse

sharing.services.PermissionService
  - resolve(user, target) -> PermissionLevel | None
    # Conflict resolution: "most permissive wins"
    # Evaluates both direct shares (on the file) and inherited shares (from parent
    # folder) and returns the highest level. A direct EDITOR share + inherited VIEWER
    # resolves to EDITOR. This matches Google Drive behaviour and avoids the need for
    # deny-permission concepts. Documented trade-off: it is not possible to restrict
    # a user on a specific file if they have broader access via a folder share.
  - require(user, target, minimum_level) -> None  # raises PermissionDenied

sharing.services.ShareService
  - share(owner, target, recipient, permission, ip) -> FileShare | FolderShare
  - revoke(owner, share_id, ip) -> None
  - list_shared_with_me(user) -> dict[files, folders]

sharing.services.PublicLinkService
  - create(owner, file_id, expires_at, ip) -> PublicLink
  - access(token, ip) -> (File, IO)
  - revoke(owner, link_id, ip) -> None

audit.services.AuditService
  - append(actor, action, target, ip, **metadata) -> AuditLog
  # This is the only write operation ever called on AuditLog
```

### 2.4 Storage Abstraction

```python
# apps/storage/backends/base.py
class StorageBackend(ABC):
    @abstractmethod
    def save(self, key: str, file_obj: BinaryIO) -> str: ...
    # returns the storage_key (may differ from input key)

    @abstractmethod
    def retrieve(self, key: str) -> BinaryIO: ...

    @abstractmethod
    def delete(self, key: str) -> None: ...

    @abstractmethod
    def exists(self, key: str) -> bool: ...

    @abstractmethod
    def size(self, key: str) -> int: ...

    @abstractmethod
    def stream(self, key: str) -> Iterator[bytes]: ...
    # Used for streaming downloads and ZIP generation
```

- `LocalDiskBackend` — full implementation, stores to Docker volume at `/media/`
- `S3Backend` — stub class with `NotImplementedError` on each method, plus docstrings
  mapping each method to its `boto3` equivalent

Key format: `{owner_id}/{file_id}/{version_number}/{original_name}`

### 2.5 API Design

Base URL: `/api/v1/`

```
Auth
  POST   /auth/register/
  POST   /auth/login/
  POST   /auth/token/refresh/

Files
  GET    /files/                               # list root files
  GET    /files/?folder={id}                   # list files in folder
  POST   /files/                               # upload
  GET    /files/{id}/                          # metadata
  DELETE /files/{id}/                          # soft delete
  GET    /files/{id}/download/                 # download current version
  GET    /files/{id}/versions/                 # list versions
  GET    /files/{id}/versions/{n}/download/    # download specific version

Folders
  GET    /folders/
  GET    /folders/?parent={id}
  POST   /folders/
  PATCH  /folders/{id}/                        # rename
  DELETE /folders/{id}/
  GET    /folders/{id}/download-zip/           # streaming ZIP

Sharing
  POST   /shares/files/{file_id}/              # share a file
  POST   /shares/folders/{folder_id}/          # share a folder
  DELETE /shares/{share_id}/                   # revoke
  GET    /shared-with-me/                      # unified view

Public Links
  POST   /public-links/                        # generate link
  DELETE /public-links/{id}/                   # revoke
  GET    /p/{token}/                           # public access (no auth)
  GET    /p/{token}/info/                      # metadata for frontend viewer

Audit
  GET    /audit/?target_id={id}                # log for a file/folder (owner only)
```

All responses use a consistent envelope:
```json
{ "data": { ... } }
{ "error": { "code": "PERMISSION_DENIED", "message": "..." } }
```

### 2.6 ZIP Download Strategy

Use `StreamingHttpResponse` with `stream-zip` (or `zipstream-new`) — not the standard
`zipfile` module. Standard `zipfile` requires seeking backwards to write central directory
headers at EOF, making true streaming impossible without holding the entire ZIP in memory
first, which would crash the server on large folders.

`stream-zip` uses ZIP Data Descriptors to write metadata inline, enabling genuine
chunk-by-chunk streaming with no temp file and no memory accumulation:

```python
def download_as_zip(user, folder_id, ip):
    folder = FolderService.get(user, folder_id)
    files = FolderService.walk_tree(folder)  # generator

    def member_files():
        for file, path in files:
            yield path, file.current_version.created_at, 0o600, ZIP_32, \
                  storage_backend.stream(file.current_version.storage_key)

    response = StreamingHttpResponse(stream_zip(member_files()), content_type='application/zip')
    response['Content-Disposition'] = f'attachment; filename="{folder.name}.zip"'
    return response
```

Add `stream-zip` to `requirements.txt`. Document trade-off in `VERIFICATION.md`:
efficient for demo scale, but production folders >500MB should use a Celery async job
that generates the ZIP to storage and returns a download URL.

---

## 3. Frontend Architecture

```
vaultshare-frontend/
├── src/
│   ├── api/           # Axios client + typed API functions per domain
│   ├── components/    # Reusable UI components
│   ├── composables/   # useAuth, useFiles, useSharing, usePublicLink
│   ├── pages/         # Route-level components
│   ├── router/
│   ├── stores/        # Pinia: auth, files, notifications
│   └── types/         # TypeScript interfaces mirroring backend DTOs
├── nginx/             # nginx.conf for production serving
├── Dockerfile
└── vite.config.ts
```

**Pages:**
| Route | Purpose |
|---|---|
| `/login` | Login form |
| `/register` | Registration form |
| `/` | File browser (root) |
| `/folders/:id` | File browser (subfolder) |
| `/shared` | Shared with me |
| `/p/:token` | Public link viewer (no auth) |

**Key components:**
- `FileTable` — lists files/folders with metadata, actions per permission level
- `UploadButton` — file input with progress bar
- `BreadcrumbNav` — folder path navigation
- `SharingModal` — search user by email, set permission, list/revoke existing shares
- `PublicLinkModal` — generate link with expiry, copy to clipboard, revoke

**Auth:** JWT access token in Pinia store (memory), refresh token in `httpOnly` cookie.

**Styling:** Tailwind CSS. Functional, not beautiful.

---

## 4. Infrastructure

`vaultshare-infra/docker-compose.yml` services:
- `api` — Django via gunicorn, port 8000
- `frontend` — nginx serving built Vue app, port 3000
- `db` — PostgreSQL 16, volume-mounted
- `redis` — Redis 7
- `celery` — Celery worker (same image as api)
- `celery-beat` — Celery Beat scheduler (cleanup expired links)

Health checks on `db` and `redis` so `api` and `celery` wait before starting.

`Makefile` commands:
```
make up              docker compose up -d --build
make down            docker compose down
make test            pytest inside api container
make test-coverage   pytest --cov with HTML report
make seed            run management command
make verify          run acceptance test suite against live stack
make check           ruff check + mypy (read-only, CI gate)
make format          ruff check --fix + ruff format (auto-fix)
make logs            docker compose logs -f
```

`make check` is the CI gate — it must pass before any phase is considered complete.
`make format` is the auto-fix loop used with the Linting Agent (see IMPLEMENTATION.md).

---

## 5. Logging

`structlog` configured to emit JSON on all handlers. Log level via `LOG_LEVEL` env var.

### request_id propagation

Every log entry produced within a request — including service layer calls, storage
backend calls, and AuditService writes — must carry the same `request_id`. This is
achieved via `structlog.contextvars`, which binds values to the current thread/async
context and merges them automatically into every subsequent log call.

The request middleware is responsible for the full lifecycle:

```python
import uuid
import structlog
import structlog.contextvars

class RequestLoggingMiddleware:
    def __call__(self, request):
        request_id = str(uuid.uuid4())
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(request_id=request_id)

        start = time.monotonic()
        response = self.get_response(request)
        duration_ms = round((time.monotonic() - start) * 1000)

        structlog.get_logger().info(
            "http_request",
            method=request.method,
            path=request.path,
            status_code=response.status_code,
            duration_ms=duration_ms,
            user_id=str(request.user.pk) if request.user.is_authenticated else None,
        )

        response["X-Request-ID"] = request_id  # returned to caller for correlation
        return response
```

Because `bind_contextvars` is set before `get_response(request)` is called, every
`structlog.get_logger().info(...)` call anywhere in the call stack — views, services,
storage backends — automatically includes `request_id` without being passed explicitly.

### Celery task logging

Celery tasks run in a separate thread and do not inherit the HTTP context. When
enqueueing a task, pass `request_id` explicitly as a task argument. The task binds it
at startup:

```python
@app.task
def cleanup_expired_links(request_id: str | None = None):
    structlog.contextvars.clear_contextvars()
    structlog.contextvars.bind_contextvars(
        request_id=request_id or str(uuid.uuid4()),  # new ID if not triggered by a request
        task="cleanup_expired_links",
    )
    log = structlog.get_logger()
    log.info("task_started")
    try:
        ...
        log.info("task_completed", revoked_count=n)
    except Exception as exc:
        log.error("task_failed", error=str(exc))
        raise
```

### Log shape

Every emitted log line includes `request_id` automatically via contextvars:

```json
{ "event": "http_request", "request_id": "550e8400-...", "method": "POST",
  "path": "/api/v1/files/", "status_code": 201, "duration_ms": 42,
  "user_id": "uuid-here" }

{ "event": "task_started", "request_id": "550e8400-...", "task": "cleanup_expired_links" }
```

Rule: never log file contents, file paths beyond storage key, or PII beyond `user_id`.

---

## 6. Testing & Verification Strategy

### Layered Test Strategy

| Layer | Tool | Scope |
|---|---|---|
| Unit | pytest | Pure logic: permissions, versioning, AuditLog immutability, token expiry |
| Integration | pytest-django | Services + DB: FileService, ShareService, AuditService |
| API | pytest-django + APIClient | HTTP endpoints: auth, upload, sharing, versioning |
| Acceptance | httpx against live Docker stack | End-to-end critical paths |

### `make verify` — Runnable Acceptance Suite

`tests/acceptance/` using `httpx`. Built incrementally — one file per phase. Runs
against `http://localhost:8000` after `make up && make seed`.

| File | Scenarios | Phase added |
|---|---|---|
| `test_core.py` | 1. Upload→download round trip; 2. Soft delete; 3. Audit log written | Phase 3 |
| `test_permissions.py` | 4. VIEWER cannot delete; 5. EDITOR cannot revoke; 6. Revoke removes access | Phase 4 |
| `test_versioning.py` | 7. Same-name upload creates v2; 8. v1/v2 bytes distinct; 9. VERSION_CREATED audit entry | Phase 5 |
| `test_folders.py` | 10. Upload into folder; 11. ZIP download valid | Phase 6 |
| `test_public_links.py` | 12. Access without auth; 13. Expired → 410; 14. Revoked → 410; 15. LINK_ACCESSED audit entry | Phase 7 |

Exit code 0 = all pass. Exit code 1 = failure with the scenario name and assertion in the message.

`make verify` is functional after Phase 3 and grows with each subsequent phase. If development
stops at any phase, whatever features exist are already automatically verifiable.

Note: `VERIFICATION.md` also documents an agent-based extension where an LLM verifies
subjective properties (error message quality, log structure) that assertions cannot check.

---

## 7. Django Admin (Staff Views)

`AuditLogAdmin` and `FileVersionAdmin` are read-only for staff:
- All fields `readonly_fields`
- `has_add_permission`, `has_change_permission`, `has_delete_permission` all return `False`
- `list_display`, `search_fields`, `list_filter` configured for easy browsing

---

## 8. Implementation Phases

### Phase 1 — Infrastructure Foundation [~45min]

**Deliverable:** `docker compose up` boots all services. `make test` runs empty suite.
`make check` passes on empty codebase. Agent Playbook files written.

**Package manager: `uv`**
Use `uv` instead of pip/poetry in the Dockerfile. Faster dependency resolution, single
binary, lock file included. The Dockerfile base pattern:
```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .
```

**Linting & typing from day one:**
- `pyproject.toml` with `ruff` (replaces black + isort + flake8) and `mypy` (strict mode)
  configured from Phase 1 — enforcing from Hour 1 keeps errors at zero rather than
  accumulating 400 type errors to fix at the end.
- `make check` must pass before each phase is marked complete.

**Agent Playbook files created in Phase 1** (written before any application code):

`IMPLEMENTATION.md` — the implementation agent persona:
> You are an expert Django 6 architect. The project follows a strict Service Layer
> pattern. Business logic lives exclusively in service classes under `apps/*/services.py`.
> Views validate input and call services — no ORM queries in views, ever.
> Always output complete file paths. Never rename existing functions.
> When adding a feature, output: (1) model changes, (2) service method, (3) serializer,
> (4) view, (5) URL registration, (6) unit test — in that order.

`TESTING.md` — the testing agent persona and strategy:
> Write pytest-django tests for the following service method.
> Mock the StorageBackend using pytest fixtures — never touch the real filesystem.
> Prioritise: (1) permission escalation edge cases, (2) audit log write on every action,
> (3) the happy path last. Use `APIClient` for endpoint tests, plain pytest for unit tests.

`CODE_REVIEW.md` — the review agent persona (filled after Phase 5 or later):
> Act as a skeptical Principal Engineer reviewing a Django REST API.
> Examine the provided code for: race conditions, N+1 queries, missing permission checks,
> DRF serializer validation gaps, and structlog misuse.
> Note: watch out for QuerySet `.update()` calls which bypass `auto_now` on `updated_at` fields. If bulk updates are needed, ensure timestamp is updated explicitly, otherwise prefer `.save()`.
> Output findings grouped by: Critical (must fix), Warning (should fix), Style (optional).
> For each finding: file path + line number, description, suggested fix.
> Do not suggest architectural refactors beyond the described patterns.

**Linting Agent rule** (in IMPLEMENTATION.md):
> Your only job is to resolve the provided `ruff`/`mypy` output.
> Add Python 3.12 type hints. Do not rename functions. Do not alter business logic.
> Do not refactor working code. Output only the changed lines with their file paths.

**Remaining Phase 1 tasks:**
- Create 3 repos (`vaultshare-api`, `vaultshare-frontend`, `vaultshare-infra`)
- `docker-compose.yml` with all services + health checks + `.env.example`
- Django project scaffold, app directories, settings split (base/dev/prod)
- `structlog` + request logging middleware
- `pytest.ini` + `conftest.py`
- `Makefile` with all commands

**Working state:** `make up` → all containers healthy. `make check` → clean.

---

### Phase 2 — Accounts & Storage Abstraction [~1h]

**Deliverable:** Register, login, JWT tokens. Storage layer defined and unit-tested.

- `User` model (email-first, AbstractUser, quota fields scaffolded), `UserService`
- JWT endpoints: `POST /auth/register/`, `POST /auth/login/`, `POST /auth/token/refresh/`
- `StorageBackend` ABC + `LocalDiskBackend` (full) + `S3Backend` (stub with docs)
- Storage backend injected via Django settings (`STORAGE_BACKEND = 'local'`)
- Unit tests: StorageBackend contract tests run against `LocalDiskBackend`
- Integration tests: register → login → get token → access protected endpoint

**Working state:** `curl POST /auth/register/` → 201. `curl POST /auth/login/` → JWT pair.

---

### Phase 3 — Core Files + AuditLog — MVS Checkpoint [~1.5h]

**Deliverable:** Upload, download, soft delete, audit log recording. Django admin operational.
First acceptance tests written and passing.

- `Folder`, `File`, `FileVersion` models (version-aware schema from day one)
- `AuditLog` model with immutability enforcement (custom `save`, `delete`, QuerySet)
- `AuditAction` enum (all action types defined upfront)
- `FileService`: `upload` (creates File + FileVersion + AuditLog), `download`, `soft_delete`
- `AuditService`: `append` only
- API endpoints: `POST /files/`, `GET /files/{id}/download/`, `DELETE /files/{id}/`, `GET /files/`
- Django admin: `AuditLog` (read-only, staff-only), `FileVersion` (read-only)
- `make seed` v1: 1 user (`demo@vaultshare.io` / `demo1234`), 5 files, audit entries
- `VERIFICATION.md` v1: manual cURL steps
- `CODE_REVIEW.md` draft
- **`tests/acceptance/test_core.py` v1:**
  - Scenario 1: register → login → upload file → download → assert bytes match
  - Scenario 2: soft delete → assert file absent from listing → assert still in DB
  - Scenario 3: perform upload → assert audit log entry exists via `GET /audit/`
  - `make verify` wired and green against these three scenarios

**Working state:** `make up && make seed && make verify` → green.

**This is the MVS. Stop here if needed.**

---

### Phase 4 — Permissions & Sharing [~1.5h]

**Deliverable:** Sharing enforced. Escalation attempts return 403. Shared-with-me works.

- `FileShare`, `FolderShare` models
- `PermissionService`: `resolve(user, target)` → effective permission;
  `require(user, target, minimum)` → raises `PermissionDenied`
- Permission hierarchy: OWNER > EDITOR > VIEWER. Folder shares inherited by children.
- `ShareService`: `share`, `revoke`, `list_shared_with_me`
- Endpoints: `POST /shares/files/{id}/`, `POST /shares/folders/{id}/`,
  `DELETE /shares/{id}/`, `GET /shared-with-me/`
- All `FileService` methods call `PermissionService.require(...)` before acting
- Integration tests: VIEWER cannot delete, EDITOR cannot revoke, only OWNER can re-share
- Update `make seed`: 3 users, files shared at different permission levels
- **`tests/acceptance/test_permissions.py` added to suite:**
  - Scenario 4: share file as VIEWER → attempt DELETE → assert 403
  - Scenario 5: share file as EDITOR → attempt revoke share → assert 403
  - Scenario 6: owner revokes share → assert file absent from recipient's shared view
  - `make verify` green across all 6 scenarios

**Working state:** `make verify` → 6 scenarios green.

---

### Phase 5 — Versioning [~1h]

**Deliverable:** Same-name upload creates new version. Any version downloadable.

- Activate versioning in `FileService.upload`: same name + same folder → increment
  `version_number`, update `current_version` pointer; else create new File
- `AuditAction.VERSION_CREATED` emitted on version bump
- Endpoints: `GET /files/{id}/versions/`, `GET /files/{id}/versions/{n}/download/`
- Integration tests: upload `report.pdf` twice → `versions.count() == 2` → download v1 → assert original bytes
- Update `make seed`: 3 files with 2+ versions, audit entries per version
- **`tests/acceptance/test_versioning.py` added to suite:**
  - Scenario 7: upload `test.txt` → upload `test.txt` again → assert `versions.count == 2`
  - Scenario 8: download v1 → assert original bytes; download v2 → assert new bytes
  - Scenario 9: assert `VERSION_CREATED` audit entry exists after second upload
  - `make verify` green across all 9 scenarios

**Working state:** `make verify` → 9 scenarios green.

---

### Phase 6 — Folders & ZIP Download [~1h]

**Deliverable:** Folder hierarchy navigable. Folder downloadable as streaming ZIP.

- `FolderService`: `create`, `list`, `rename`, `soft_delete`, `walk_tree`
- Folder endpoints: full CRUD + `GET /folders/{id}/download-zip/`
- `download_as_zip` returns `StreamingHttpResponse` (`Content-Type: application/zip`)
- ZIP entry paths preserve folder structure: `Reports/Q1/sales.pdf`
- AuditLog `DOWNLOAD` entry for each file included in ZIP
- Update `make seed` to full spec: 5 users, `Projects/Q1 2025/Reports` hierarchy,
  variety of file types and sizes, 20+ audit entries per user
- **`tests/acceptance/test_folders.py` added to suite:**
  - Scenario 10: create folder → upload file into it → list folder → assert file present
  - Scenario 11: `GET /folders/{id}/download-zip/` → assert valid ZIP → assert entry paths
  - `make verify` green across all 11 scenarios

**Working state:** `make verify` → 11 scenarios green.

---

### Phase 7 — Public Links & Celery Cleanup [~45min]

**Deliverable:** Time-limited public links. Celery cleans expired links on schedule.

- `PublicLink` model, `generate_token = lambda: secrets.token_urlsafe(32)`
- `PublicLinkService`: `create`, `access` (checks expiry + revoked, increments
  `access_count`, logs `LINK_ACCESSED`), `revoke`
- `GET /p/{token}/` endpoint — no authentication required
- `GET /p/{token}/info/` — metadata endpoint for frontend viewer (no auth)
- Expired/revoked links → `410 Gone` + `{"error": {"code": "LINK_EXPIRED"}}`
- Celery task `cleanup_expired_links`: periodic via Celery Beat (every hour)
- Every Celery task logs `task_started`, `task_completed`, `task_failed`
- Update `make seed`: 2 active links, 1 expired
- **`tests/acceptance/test_public_links.py` added to suite:**
  - Scenario 12: create public link → access `GET /p/{token}/` without auth → assert 200
  - Scenario 13: create link with past `expires_at` → access → assert 410 + `LINK_EXPIRED`
  - Scenario 14: revoke active link → access → assert 410
  - Scenario 15: assert `LINK_ACCESSED` audit entry written on successful access
  - `make verify` green across all 15 scenarios

**Working state:** `make verify` → 15 scenarios green.

**At this point, all three top-scored features are complete, tested, and automatically verifiable.**

---

### Phase 8 — Frontend [~3h]

**Deliverable:** Full Vue 3 frontend covering all required screens.

- Setup (15min): Vite + Vue 3 + TypeScript strict + Pinia + Vue Router + Tailwind + Axios.
  Dockerfile + nginx.conf.
- Auth (20min): Login/register pages, JWT refresh interceptor in Axios, `useAuth`.
- File Browser (45min): `FileTable`, folder navigation with `BreadcrumbNav`,
  `UploadButton` with progress. Calls `GET /files/` and `GET /folders/`.
- Sharing Modal (30min): search user by email, set permission, list/revoke shares.
- Shared with Me (20min): unified view, permission badge per item.
- Public Link Viewer (20min): `/p/:token` route — file info + expiry + download button.
  Works without auth.
- Public Link Modal (20min): expiry date picker, copy-to-clipboard, revoke button.
- Polish + Dockerfile (10min): nginx serves built assets.

**Working state:** `make up && make seed` → `localhost:3000` → full flow explorable.

---

### Phase 9 — Bonus Scaffolding + Final Polish [~1.5h]

**Deliverable:** Bonuses scaffolded. `make verify` runs clean against full stack. `CODE_REVIEW.md` complete.

**Verification suite is already complete from Phases 3–7. This phase only finalises it:**
- Run `make verify` from scratch on a clean state and confirm all 15 scenarios pass
- Add `VERIFICATION.md` final section: agent-based verification approach (documented, not implemented)

**Bonus A scaffold (15min):** `SearchVectorField` on `File.name` + `Folder.name`.
`GET /search/?q=` backed by PostgreSQL full-text search. Search bar in frontend header.

**Bonus B scaffold (10min):** `User.storage_used_bytes` updated in `FileService.upload`
and `soft_delete`. Upload raises `HTTP 413` when `used + incoming > quota`.
Frontend quota bar in sidebar. On `soft_delete`, deduct the bytes of ALL active versions
of the file (not just the current version) — this gives the user immediate quota relief
matching the perceived behaviour of a hard delete.

**Bonus C scaffold (10min):** `django-channels` dependency + `notifications` consumer
stub. Document WebSocket URL and message schema in README.

**AI Code Review (15min):** Structured review pass. `CODE_REVIEW.md`: prompt used,
findings by severity (Critical / Warning / Style), decision per finding.

**Final polish (5min):** README "what I'd do next" in each repo. Verify clean-state
`docker compose up`.

---

## 9. Known Trade-offs & Documented Upgrade Paths

These are intentional shortcuts taken for the time budget. Each is documented here so
the CODE_REVIEW.md and README "what I'd do next" sections have concrete content.

### Folder Hierarchy: Adjacency List → O(N) queries

**Current:** `Folder.parent = FK(self)`. Simple to build, easy to reason about.

**Problem:** Fetching all descendants of a folder (for ZIP generation or permission
inheritance) requires one query per depth level — O(N) round-trips for a tree of depth N.

**Production upgrade path:**
1. **Materialized Path** (`django-treebeard` `MP_Node`) — medium effort migration.
   Stores the full path as a string (`0001.0002.0003`). Descendants in one indexed query.
2. **PostgreSQL LTREE** — high effort. Native hierarchical index, the most
   performant option, but requires a raw SQL migration and `django-ltree` wrapper.

Document this in the API `README.md` under "What I'd do next".

### Public Link Invalidation: Independent of Creator's Permission

**Current:** A `PublicLink` is valid until its `expires_at` or explicit revocation,
regardless of whether the creator has since been downgraded or removed as owner.

**Rationale:** Implementing cascading invalidation would require hooks on every
permission change event to hunt down and revoke related links — significant complexity
for a time-limited build.

**Production fix:** Add a `check_creator_permission` step in `PublicLinkService.access`
that re-validates the creator still has at least VIEWER access before serving the file.

Document this in `CODE_REVIEW.md` as a deliberate shortcut with a known fix.

---

## 10. Phase Checkpoint Summary

| Phase | Cumulative Time | Key Output | Scored Feature |
|---|---|---|---|
| 1 | 0:45 | Docker stack boots | DX |
| 2 | 1:45 | JWT auth working | Auth |
| 3 | 3:15 | **MVS reached** | Audit log |
| 4 | 4:45 | **Permissions enforced** | Permissions ← top 3 |
| 5 | 5:45 | **Versioning complete** | Versioning ← top 3 |
| 6 | 6:45 | Folders + ZIP | File management |
| 7 | 7:30 | Public links + Celery | Async + sharing |
| 8 | 10:30 | Frontend complete | UI requirement |
| 9 | 12:00 | Verified + reviewed | Verification (20%) + AI review (15%) |

If Phase 8 is not started at hour 8, skip to Phase 9. A well-verified backend without a
frontend scores higher than an unverified full stack.
