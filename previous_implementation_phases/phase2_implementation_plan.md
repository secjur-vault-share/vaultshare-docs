# Phase 2 — Accounts & Storage Abstraction Detailed Implementation Blueprint

This blueprint dictates the exact technical specifications for Phase 2 implementation. It dictates the architectural boundaries, database schema overrides, file structures, and programmatic interfaces. No code will be written until this blueprint is approved.

---

## 1. Scope & Deliverables

This phase establishes the user authentication boundary and the fundamental storage abstraction.
- Build a customized, email-first `User` model.
- Set up JWT authentication endpoints.
- Implement the storage layer abstraction with a functional `LocalDiskBackend` and a documented `S3Backend` stub.

---

## 2. Strict File-by-File Technical Plan

### A. Environment Configuration

#### `pyproject.toml`
- **Action**: Add `djangorestframework-simplejwt>=5.3.1` to the `dependencies` array.

#### `config/settings/base.py`
- **Action**: Register `apps.accounts` and `apps.storage` into `INSTALLED_APPS`.
- **Action**: Add configurations:
  ```python
  AUTH_USER_MODEL = "accounts.User"
  REST_FRAMEWORK = {
      "DEFAULT_AUTHENTICATION_CLASSES": (
          "rest_framework_simplejwt.authentication.JWTAuthentication",
      )
  }
  SIMPLE_JWT = {
      "ACCESS_TOKEN_LIFETIME": timedelta(minutes=60),
      "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
      "ROTATE_REFRESH_TOKENS": True,
  }
  STORAGE_BACKEND = "apps.storage.backends.local.LocalDiskBackend"
  ```

---

### B. App 1: Accounts (`apps/accounts/`)

#### `models.py`
- **Action**: Define `User` inheriting from `AbstractUser`.
- **Fields**:
  - `id`: `models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)`
  - `email`: `models.EmailField("email address", unique=True)`
  - `username`: `models.CharField("username", max_length=150, blank=True)`
  - `storage_quota_bytes`: `models.BigIntegerField(default=524288000)` *(500MB)*
  - `storage_used_bytes`: `models.BigIntegerField(default=0)`
- **Meta Options**:
  - `USERNAME_FIELD = "email"`
  - `REQUIRED_FIELDS = []`
- **Action**: Create a custom `UserManager` overriding `create_user` and `create_superuser` to remove the mandatory `username` parameter checking, allowing purely email-based registration.

#### `services.py`
- **Action**: Create declarative business logic.
- **Signature**: `def register(*, email: str, password: str, username: str = "") -> User:`
- **Logic**: Normalize email, check uniqueness (raise `ValidationError` if duplicate), instantiate User, hash password via `set_password()`, save and return.

#### `serializers.py`
- **Action**: Define `UserRegistrationSerializer(serializers.ModelSerializer)`.
- **Fields**: `["email", "password", "username"]`
- **Overrides**: `create(self, validated_data)` invokes `UserService.register(...)`.

#### `views.py` & `urls.py`
- **Action**: Create `RegisterView(APIView)` allowing `AllowAny` permissions. Invokes `UserRegistrationSerializer`.
- **Action**: Map routes in `apps/accounts/urls.py`:
  - `path("register/", RegisterView.as_view(), name="register")`
  - `path("login/", TokenObtainPairView.as_view(), name="login")`
  - `path("token/refresh/", TokenRefreshView.as_view(), name="token_refresh")`

---

### C. App 2: Storage (`apps/storage/`)

#### `backends/base.py`
- **Action**: Define abstract interface contract.
- **Class**: `class StorageBackend(ABC):`
  - `abstractmethod def save(self, key: str, file_obj: BinaryIO) -> str`
  - `abstractmethod def retrieve(self, key: str) -> BinaryIO`
  - `abstractmethod def delete(self, key: str) -> None`
  - `abstractmethod def exists(self, key: str) -> bool`
  - `abstractmethod def size(self, key: str) -> int`
  - `abstractmethod def stream(self, key: str, chunk_size: int = 8192) -> Iterator[bytes]`

#### `backends/local.py`
- **Action**: Concrete implementation storing files on OS native disk.
- **Class**: `class LocalDiskBackend(StorageBackend):`
- **Logic Constraints**: 
  - Instantiation resolves destination root using `getattr(settings, "MEDIA_ROOT", "/media/")`.
  - Directory Traversal Protection: Helper method `_get_abs_path(key)` resolves absolute path and asserts it strictly starts with the base `MEDIA_ROOT`. If not, raises `ValueError`.

#### `backends/s3.py`
- **Action**: Concrete S3 implementation stub.
- **Class**: `class S3Backend(StorageBackend):`
- **Logic Constraints**: Implements all 6 abstract methods required by `StorageBackend` but hardcodes `raise NotImplementedError("S3 backend is not implemented yet.")`. Docstrings will cite exact `boto3` operations that would eventually belong here (e.g. `boto3.client('s3').upload_fileobj(...)`).

#### `services.py`
- **Action**: Dependency Injection Factory.
- **Signature**: `def get_storage_backend() -> StorageBackend:`
- **Logic**: Reads `settings.STORAGE_BACKEND`, uses Django's `import_string`, instances the class, and returns it.

---

### D. Testing & Verification Requirements (`tests/`)

**Unit Tests (`tests/unit/test_storage.py`)**
- `test_local_disk_save_and_retrieve`: Write file, read file, assert content matching.
- `test_local_disk_directory_traversal`: Attempt to save file with key `../../../etc/passwd`. Assert `ValueError` is correctly raised blocking the bypass attempt.
- `test_local_disk_stream_chunks`: Inject payload >8192 bytes, iter over stream chunks and assert exact piece generation.

**Integration Tests (`tests/integration/test_accounts.py`)**
- `test_register_user_success`: POST `/api/v1/auth/register/` and assert HTTP 201 with returned user UUID.
- `test_register_duplicate_email`: Create user in setup, POST duplicate to `/register/`, assert HTTP 400.
- `test_login_user`: POST `/api/v1/auth/login/` asserting payload contains structured string types under `access` and `refresh` JSON keys.
