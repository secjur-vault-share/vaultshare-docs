# VaultShare — Design Decisions & Trail of Thought

This document captures the reasoning behind every significant architectural decision in
`ARCHITECTURE.md`. It is not a summary — it is the record of how we got there, including
what was wrong with earlier ideas and why we changed them.

---

## 1. Starting Point: What Was Wrong With the Original Plan

The original plan was a six-phase MVS-optimised roadmap. The critique that drove the
entire redesign identified three structural problems:

**Problem 1: The three most-evaluated features were at the end.**
The task brief explicitly states: *"The features we will look at most closely are the
permission model, the audit trail, and the versioning logic."* The original plan put
versioning in Phase 6 and permissions in Phase 5. If time ran short, the most important
features would be incomplete.

**Problem 2: The frontend was missing entirely.**
The task requires Vue 3 + TypeScript as a mandatory deliverable, not a bonus. The
original plan had no frontend phase at all.

**Problem 3: The verification suite was documentation, not code.**
The task assigns 20% of the score to acceptance and verification, and explicitly requires
it to be "runnable with a single command." The original plan only mentioned writing a
`VERIFICATION.md` file — a document, not a test suite.

**Resolution:** Rebuild the plan risk-adjusted: front-load the top-scored features,
add a frontend phase, and turn verification into a runnable artifact.

---

## 2. Multi-Repo Structure

**Decision:** Three repositories — `vaultshare-api`, `vaultshare-frontend`,
`vaultshare-infra` — under a GitHub Organization (`secjur-vault-share`).

**Reasoning:** The task explicitly states the multi-repo split is intentional and that
it "demonstrates service-thinking." A monorepo would satisfy the letter but miss the
point. The `docker-compose.yml` references both the API and the frontend, so it cannot
live in either service repo without creating an artificial dependency — it belongs in a
dedicated infra repo that acts as the composition root.

**On organization visibility:** GitHub organizations are always publicly visible, but
individual repositories can be private. The workflow is: create the org immediately
(the URL is ready to share), keep repos private during development, flip them public
at submission time.

---

## 3. Django Project Structure: Modular Monolith

**Decision:** Domain-driven app boundaries. Apps: `core`, `accounts`, `storage`,
`sharing`, `audit`. No cross-app ORM imports. Service layer between views and models.

**Reasoning:** Current market consensus for Django at this scale is a modular monolith
with bounded contexts per domain. Each app owns its models, services, serializers,
views, URLs, and tests. The alternative — a single flat app — works at demo scale but
makes the "are service boundaries clean?" rubric criterion unanswerable. The alternative
— microservices — is over-engineered for a take-home challenge and adds operational
complexity that eats into the time budget.

The rule "views are thin, logic lives in services" was chosen deliberately: services
are independently testable without HTTP overhead, which is what makes the 70% coverage
target achievable within the time budget.

---

## 4. Phase Ordering: Risk-Adjusted for Score Weight

**Decision:** Phases 3 (AuditLog), 4 (Permissions), 5 (Versioning) come before
Phases 6 (Folders/ZIP), 7 (Public Links), 8 (Frontend).

**Reasoning:** The three features the reviewers evaluate most closely (permissions,
audit trail, versioning) are now complete by hour 5:45. If development stops at that
point, the submission still covers what matters most. Folders and ZIP are valuable but
not in the top-3. The frontend is required for a full score but not for a strong score
— a well-tested, verified backend without a frontend beats an unverified full stack.

The explicit rule: *"If Phase 8 is not started at hour 8, skip to Phase 9."*

---

## 5. File Model: Version-Aware Schema From Day One

**Decision:** `File` and `FileVersion` are separate models from Phase 3 onwards.
`File` is the identity. `FileVersion` is the content. `File.current_version` is a
`OneToOneField` pointing at the active version.

**Reasoning:** Many implementations add versioning as an afterthought by adding a
`version` counter to the `File` model. This creates a schema migration mid-project that
breaks existing tests and queries. Starting with the correct schema — even before
versioning logic is activated — means Phase 5 (versioning) is purely a service-layer
change with no model migration required.

---

## 6. Soft Delete: `deleted_at` Only, Not `is_deleted + deleted_at`

**Decision:** Use only `deleted_at: DateTimeField(null=True)` on `File` and `Folder`.
`deleted_at IS NOT NULL` encodes the deleted state. No separate `is_deleted` boolean.

**Reasoning:** Two fields encoding the same state is a consistency hazard — they can
drift. `deleted_at` alone is sufficient: `null` means active, any timestamp means
soft-deleted. It also provides the deletion timestamp for free, which is useful for
audit purposes and quota calculations.

**Consequence for the UniqueConstraint:** The partial unique index on
`(owner, folder, name)` uses `condition=Q(deleted_at__isnull=True)` instead of
`condition=Q(is_deleted=False)`.

---

## 7. Unique Constraint on File: Database-Level Race Condition Prevention

**Decision:** Add `UniqueConstraint(fields=['owner', 'folder', 'name'], condition=Q(deleted_at__isnull=True))` to the `File` model.

**Reasoning:** The versioning spec says "uploading a file with the same name into the
same folder creates a new version." The naive implementation — `File.objects.filter(name=...).exists()` — is vulnerable to a race condition: two simultaneous uploads of the same filename can both pass the check before either has committed, resulting in duplicate `File` rows instead of a new version.

Enforcing this at the database level via a conditional unique index prevents this
entirely. The condition `deleted_at__isnull=True` ensures that soft-deleted files do
not block new uploads of the same name, which matches user expectation.

---

## 8. AuditLog Immutability: Model-Level Enforcement

**Decision:** Override `save()`, `delete()`, and the QuerySet's `update()` and
`delete()` methods on `AuditLog` to raise `ImproperlyConfigured`.

**Reasoning:** The task brief calls out the audit trail specifically as a compliance
feature: *"SECJUR builds compliance tools. An immutable audit trail is core to how our
customers think about data."* Immutability enforced only by convention is not
immutability — a future developer, a Django admin action, or a bulk ORM operation can
silently break it. Enforcement at the model level means the constraint is architectural,
not procedural.

**On `target_name` denormalization:** The audit log captures `target_name` as a plain
string at write time. If a folder named "Q1 Reports" is later renamed or deleted, the
historical log entry still reads "Q1 Reports" — because that was the fact at the time
the action occurred. This is the correct compliance behaviour.

---

## 9. Permission Model: "Most Permissive Wins"

**Decision:** When a user has both a direct file share and an inherited folder share,
`PermissionService.resolve()` returns the higher of the two.

**Reasoning:** This matches Google Drive's behaviour and avoids the need for a "deny"
permission concept, which would require a three-valued permission system (grant, deny,
inherit) and significantly more complex resolution logic. The documented trade-off:
it is not possible to restrict a user on a specific file if they already have broader
access via a folder share. This is explicitly noted in the codebase and in the
`CODE_REVIEW.md` as a known limitation.

---

## 10. Storage Abstraction: Why a Stub S3Backend Matters

**Decision:** Implement `StorageBackend` as an ABC with `LocalDiskBackend` (full) and
`S3Backend` (stub with `NotImplementedError` + docstrings mapping to `boto3`).

**Reasoning:** The rubric asks: *"Is the storage abstraction genuinely swappable?"*
Writing only a `LocalDiskBackend` and claiming swappability is not enough — the reviewer
cannot evaluate the claim. A `S3Backend` stub, even with `NotImplementedError` on each
method, demonstrates that the interface contract is real: it shows exactly which methods
a new backend must implement, and what each one maps to in `boto3`. The abstraction is
genuinely swappable because you can see what swapping would require.

---

## 11. ZIP Download: `stream-zip` Not `zipfile`

**Decision:** Use the `stream-zip` library, not Python's standard `zipfile` module.

**Reasoning:** This was a correction to the original plan. Standard `zipfile` requires
seeking backwards to write central directory headers at EOF. This makes true streaming
impossible — without a seekable stream, you must hold the entire ZIP in memory before
writing it out, which crashes the server on large folders.

`stream-zip` uses ZIP Data Descriptors to write metadata inline, enabling genuine
chunk-by-chunk streaming via `StreamingHttpResponse` with no temp file and no memory
accumulation. The dependency must be in `requirements.txt`.

---

## 12. Verification Suite: Built Incrementally, Not Deferred

**Decision:** Write one `tests/acceptance/test_*.py` file per feature phase, starting
at Phase 3. `make verify` is functional after Phase 3 and grows with each phase.

**Reasoning:** The original plan built the acceptance suite entirely in Phase 9 — the
last phase, after the frontend. The problem: if Phase 8 (Frontend, ~3h) overran the
time budget, Phase 9 would be skipped entirely, submitting a project without automated
verification. Verification is worth 20% of the score and the brief calls it an absolute
requirement.

The fix is structural: build the test for each feature at the same time as the feature.
If development stops at Phase 6, the submission has a passing `make verify` covering
everything built so far. The cost is ~10 minutes per phase.

**On agent-based verification:** An LLM agent could verify subjective properties
(error message clarity, log structure quality) that `assert` statements cannot check.
This is documented in `VERIFICATION.md` as a future direction, but the runnable suite
uses deterministic `httpx` assertions — an LLM is not a stable CI dependency.

---

## 13. `request_id` in All Log Entries

**Decision:** Generate a UUID per request in the middleware, bind it via
`structlog.contextvars`, and return it in the `X-Request-ID` response header.

**Reasoning:** Without a `request_id`, correlating log entries from a single request
across the middleware, service layer, storage backend, and audit service is impossible.
With `structlog.contextvars`, the `request_id` is bound once in the middleware before
`get_response()` is called, and every subsequent `structlog.get_logger().info(...)` call
anywhere in the call stack automatically includes it — no explicit passing required.

Celery tasks run in a separate thread and do not inherit the HTTP context. The pattern
is to pass `request_id` as an explicit task argument at enqueue time and bind it at
task startup. Tasks triggered by Celery Beat (not by a request) generate a fresh UUID.

---

## 14. Agent Playbooks: Written Before Application Code

**Decision:** Create `IMPLEMENTATION.md`, `TESTING.md`, and `CODE_REVIEW.md` in Phase 1,
before any application code is written.

**Reasoning:** Writing AI prompts after the code is written leads to "prompt drift" —
iteratively tweaking prompts in chat windows without a consistent contract. Writing the
playbooks upfront establishes the rules the AI must follow for every interaction:
the service layer rule, the output ordering rule, the linting agent constraint. It also
demonstrates to the reviewers that AI is being used as a directed tool with rigorous
prompting standards, not as an autocomplete button — which the brief explicitly says
is what they are evaluating.

The most important constraint is the **Linting Agent rule** in `IMPLEMENTATION.md`:
*"Your only job is to resolve the provided ruff/mypy output. Do not rename functions.
Do not alter business logic. Do not refactor working code."* Without this constraint,
AI agents frequently "help" by rewriting entire functions when asked to add a type hint.

---

## 15. `uv` + `ruff` + `mypy` From Phase 1

**Decision:** Use `uv` as the package manager. Configure `ruff` and `mypy` in
`pyproject.toml` from Phase 1. Add `make check` (CI gate) and `make format` (auto-fix)
to the Makefile.

**Reasoning on `uv`:** Faster dependency resolution and container builds than
pip/poetry. Single binary. Reviewers familiar with modern Python tooling will recognise
it as a deliberate, current choice.

**Reasoning on linting from day one:** Adding linters at Phase 7 means 400+ type errors
and formatting violations accumulated over the previous 6 phases. Fixing them in bulk
at the end is slow and risky — the Linting Agent may make incorrect "fixes." Enforcing
from Phase 1 keeps the error count at zero throughout, making `make check` a trivially
fast gate rather than a debugging session.

`ruff` was chosen because it replaces `black` + `isort` + `flake8` as a single tool,
keeping the `pyproject.toml` configuration minimal and the `Makefile` simple.

---

## 16. Public Link Invalidation: Documented Shortcut

**Decision:** `PublicLink` validity is independent of the creator's current permission
level. A link created by an owner who is later downgraded remains valid.

**Reasoning:** Implementing cascading invalidation requires hooks on every permission
change event that hunt down and revoke related links. This is non-trivial complexity
with significant edge cases (indirect permission changes via folder share revocation,
chain revocations). For a time-limited build, this is a conscious trade-off.

The production fix is simple and documented: add a `check_creator_permission` step in
`PublicLinkService.access` that re-validates the creator's access before serving the
file. The fix is noted in `CODE_REVIEW.md` as a deliberate shortcut with a known
resolution — which is exactly the kind of visible, reasoned decision the AI Code Review
rubric rewards.

---

## 17. Folder Hierarchy: Adjacency List With a Documented Upgrade Path

**Decision:** Use `Folder.parent = FK(self)` (Adjacency List). Document the O(N)
limitation and the upgrade path in the README.

**Reasoning:** Adjacency List is the fastest to build and the easiest to reason about
within the time budget. The pitfall — fetching all descendants requires one query per
depth level — is real but acceptable at demo scale, where folder trees are shallow.

The documented upgrade path matters because the rubric asks whether trade-offs are
documented. Simply choosing Adjacency List without explanation looks like ignorance.
Choosing it explicitly, naming the problem, and specifying the two production alternatives
(django-treebeard Materialized Path for medium effort, PostgreSQL LTREE for high effort)
shows the decision was made with full awareness.

---

## Summary: Decision Chronology

| # | Decision | Driver |
|---|---|---|
| 1 | Rebuild phase order (top features first) | Phase ordering was inverted relative to score weight |
| 2 | Add frontend phase | Frontend is required, not optional |
| 3 | Three separate repos + GitHub org | Task rubric rewards service-thinking |
| 4 | Modular monolith, domain-driven apps | Architecture is 20% of score |
| 5 | File + FileVersion separated from day one | Prevents mid-project schema migration |
| 6 | `deleted_at` only, no `is_deleted` | Removes dual-state consistency hazard |
| 7 | DB-level UniqueConstraint on active files | Race condition fix, not convention |
| 8 | AuditLog model-level immutability | Compliance feature — enforced, not assumed |
| 9 | "Most permissive wins" permission resolution | Simpler than deny-permissions, documented limit |
| 10 | S3Backend stub | Makes storage abstraction evaluable, not just claimed |
| 11 | `stream-zip` not `zipfile` | Corrected a genuine error in the original plan |
| 12 | Verification suite built incrementally | Removed 20%-score risk from last phase |
| 13 | `request_id` via structlog contextvars | Log correlation across service layers |
| 14 | Agent Playbooks in Phase 1 | TDD for agentic engineering — prevents prompt drift |
| 15 | `uv` + `ruff` + `mypy` from Phase 1 | Zero error debt, modern tooling signal |
| 16 | Public link creator permission shortcut | Documented trade-off, not a hidden bug |
| 17 | Adjacency List + upgrade path documented | Honest trade-off with concrete next steps |
