# 🗂️ VaultShare — Backend Developer Take-Home Task

**SECJUR Engineering Challenge · Backend Developer (all genders)**

---

## Overview

Welcome to the SECJUR engineering challenge! This task is designed to give you creative freedom while letting us evaluate how you think, architect, and ship software.

You will build **VaultShare** — a lightweight file sharing platform inspired by Google Drive. Users can upload files, organise them, and share them with other users at different permission levels.

This project reflects the kind of work we do at SECJUR: multi-service backend architectures, thoughtful API design, AI integrations, and production-quality engineering practices.

> **Time expectation:** We suggest spending 6–10 hours. We are not looking for a finished product — we are looking for good decisions, honest trade-offs, and clean code. Document what you would do next if you had more time.

> **On using AI:** This task is intentionally scoped beyond what a single engineer can deliver alone in 6–10 hours. You are expected to use AI tools heavily — for code generation, review, debugging, and anything else that helps you ship. That is not a shortcut; it is the point. We are less interested in what you can write by hand than in how you direct, evaluate, and take ownership of what your AI agents produce. Your AI toolset, your prompting approach, and your judgment about when to trust or push back on generated output are all part of what we are evaluating.

---

## Repository Structure

You must create **at least two separate Git repositories** (not a monorepo). More repositories are welcome and will be rewarded — they demonstrate service-thinking.

| Repository | Content |
|---|---|
| `vaultshare-api` | Django backend (required) |
| `vaultshare-frontend` | Vue 3 + TypeScript frontend (required) |
| `vaultshare-e2e` *(optional)* | Playwright end-to-end tests as a standalone suite |
| `vaultshare-infra` *(optional)* | Docker Compose orchestration & shared config |

All repositories must be public on GitHub (or GitLab) and submitted together. Each repo should have its own `README.md` with setup instructions.

> **Note on the monorepo rule:** The split is intentional. We want to see that you can design clean service boundaries and think about deployment independence — even in a small project.

---

## Core Features

### 1. Authentication

- User registration and login (JWT or session-based — your choice, justify it)
- Each user has a private file space

### 2. File Management

- Upload files (any type; set a reasonable size limit)
- Download files
- Download an entire folder as a ZIP archive — think carefully about how you handle this when the folder is large
- Delete files (soft delete preferred — nothing truly disappears)
- List files in your own space with metadata: name, size, MIME type, uploaded at, last modified
- Organise files into **folders**

### 3. Sharing

- Share a file or folder with another registered user
- Three permission levels: **Viewer** (download only), **Editor** (upload/rename), **Owner** (full control including delete and re-share)
- A user can see all files and folders shared with them in a unified "Shared with me" view
- Revoke access at any time (owner only)
- **Public link sharing**: generate a time-limited, shareable link for a file that works without authentication (like Google Drive's "Anyone with the link")

### 4. File Versioning

- Uploading a file with the same name into the same folder creates a new **version**, not a duplicate
- Users can list versions of a file and download any previous version
- The current version is always the default download

### 5. Activity Log (Audit Trail)

- Every meaningful action is recorded: upload, download, share, permission change, delete, version created, link generated
- The log includes: actor, action, target (file/folder), timestamp, IP address
- Users can view the activity log for files they own
- This log is **append-only** — no deletes, no updates

> *Why this matters to us: SECJUR builds compliance tools. An immutable audit trail is core to how our customers think about data. We want to see that you approach this feature with the same mindset.*

---

## Technical Requirements

### Backend (`vaultshare-api`)

- **Framework:** Django 6+ + Django REST Framework
- **Language:** Python 3.12+
- **Database:** PostgreSQL
- **File storage:** Local filesystem is fine for this task (but design the storage layer so it could be swapped for Azure Blob Storage or S3 without changing business logic)
- **Task queue:** Celery + Redis for async tasks (e.g., generating public links, cleanup of expired links)
- **API:** RESTful, with clear and consistent endpoint naming

### Frontend (`vaultshare-frontend`)

- **Framework:** Vue 3 with Composition API
- **Language:** TypeScript (strict mode encouraged)
- **Build tool:** Vite
- **Styling:** Your choice — keep it clean and usable. It doesn't need to be beautiful, but it must be functional.
- The frontend must cover at minimum: login/register, file browser, upload, sharing modal, and the "Shared with me" view.

### Infrastructure

- **Everything runs with a single `docker compose up`** — no manual steps required beyond copying an `.env.example`
- The compose setup must include: API, frontend (served by nginx or Vite dev server), PostgreSQL, Redis
- Include a `Makefile` or `justfile` with common commands: `make up`, `make test`, `make seed`

---

## Demo Data Generator

Running `make seed` (or equivalent) must populate the database with realistic demo data:

- At least 5 users (including one named `demo@vaultshare.io` with password `demo1234`)
- A realistic folder hierarchy per user (e.g., "Projects / Q1 2025 / Reports")
- A variety of file types and sizes (you can use lorem-ipsum-style generated files)
- Some files already shared between users at different permission levels
- At least 20 activity log entries per user
- At least 3 files with multiple versions

The goal: when a reviewer runs `make seed` and opens the app, they should immediately be able to explore a living, populated system — not an empty screen.

---

## Testing Strategy

Document your testing strategy in a `TESTING.md` file at the root of `vaultshare-api`. Explain:

- What you chose to test and why
- What you explicitly skipped and why
- What your coverage targets are and how you measured them

### Required tests

- **Unit tests** for core business logic: permission checks, versioning logic, public link expiry, audit log writes
- **Integration tests** for the most important API endpoints: auth, file upload/download, sharing, version listing
- Use `pytest` and `pytest-django`
- Coverage report must be generatable via `make test-coverage`
- Aim for >70% coverage on the backend — but a thoughtful 70% beats a meaningless 90%

### Frontend tests (optional but appreciated)

- At least one component test using Vitest
- One happy-path E2E test using Playwright or Cypress is a strong bonus

---

## Logging System

- Use Python's `structlog` library (or standard `logging` with JSON formatting)
- Every HTTP request must be logged: method, path, status code, duration, user ID (if authenticated)
- Every background task must log start, completion, and any errors
- Logs must be structured JSON — not free-form text — so they could be shipped to a log aggregator
- Log levels must be configurable via environment variable (`LOG_LEVEL`)
- **Do not log file contents or PII beyond user ID**

---

## Bonus Challenges

These are genuinely optional. Do one, do all, do none — but if you do one, do it properly.

### 🔍 Bonus A: Full-Text Search
Implement a search endpoint that searches across filenames and folder names. Use PostgreSQL full-text search (via `django.contrib.postgres`) or integrate Elasticsearch. The frontend should have a search bar.

### 📊 Bonus B: Storage Quota
Each user has a storage quota (configurable, default 500 MB). Track usage in real-time. The API returns `HTTP 413` with a meaningful error when a user exceeds their quota. Display current usage in the UI.

### ⚡ Bonus C: Real-Time Notifications
Use Django Channels (WebSockets) or Server-Sent Events to push real-time notifications to the frontend when a file is shared with the logged-in user. A toast notification should appear without requiring a page refresh.

---

## Minimum Viable Submission

The full brief is intentionally more than one person can complete in a weekend. If the scope feels daunting, here is the floor we will accept — and evaluate seriously.

A minimum viable submission must include:

- A running `docker compose up` with API and database (frontend is a bonus at this level)
- Auth (register + login)
- File upload and download
- A seed script with at least one user and a few files
- `VERIFICATION.md` describing how you would verify your features — even as a written test plan if a runnable suite isn't ready
- `CODE_REVIEW.md` documenting at least one AI review pass over your own code

Everything beyond this — folders, sharing, versioning, audit log, public links, the frontend, the bonus features — raises your score but is not required to submit.

> The minimum deliverables are deliberately chosen so that an LLM alone cannot produce them in one prompt. They require a running stack, real HTTP responses, and a verification step that executes against it. That bar is the point.

---

## Acceptance Criteria & Verification

Your submission must include a way to **automatically verify that every feature described in this brief actually works** — from the outside, against the running stack, without reading the source code.

How you design and implement this is part of the challenge. Think carefully: if a reviewer (or an automated pipeline) had nothing but your repositories and a `docker compose up`, how would they know — with confidence — that file versioning works correctly, that a Viewer cannot delete a file, or that an expired public link returns the right response?

Document your approach in a `VERIFICATION.md` and make it runnable with a single command.

---

## AI-Assisted Code Review

At SECJUR, AI agents are part of our engineering workflow — including code review. We expect you to apply this to your own submission.

Before finalising your code, run at least one dedicated AI code review pass over your work. This is separate from using AI to generate code. The goal is critical evaluation: treat the AI as a skeptical senior engineer who has been asked to find problems.

What you do with the review is up to you. You might address every finding, dismiss some with a clear reason, defer others to a "what I'd do next" list, or discover that a finding reveals a genuine design flaw worth rethinking. All of these are valid — what matters is that the process is **visible and reasoned**.

Deliver this as a `CODE_REVIEW.md` file in the `vaultshare-api` repository containing:

- The prompts or approach you used to conduct the review
- A summary of the findings, categorised by severity or theme
- For each finding: what you did about it and why

There is no expected length or format. A short, honest document is better than a long performative one.

> *Why we ask for this: generating code and evaluating code are different skills. We want to see that you can do both — and that you use AI as a tool you direct, not a process you follow.*

---

## What We Will Evaluate

### 1. Architecture & Design (20%)
- Are service boundaries clean and well-reasoned?
- Is the Django project structured in a way that could scale?
- Is the storage abstraction genuinely swappable?
- Are models normalised appropriately?

### 2. Code Quality (20%)
- Is the code readable and consistent?
- Are abstractions at the right level — not too shallow, not over-engineered?
- Are edge cases handled (missing files, concurrent uploads, permission escalation attempts)?
- Is error handling thoughtful, with meaningful API error responses?

### 3. Testing (15%)
- Is the testing strategy documented and defensible?
- Do the tests give genuine confidence in the most important logic?
- Are tests fast, isolated, and reproducible?

### 4. Acceptance & Verification (20%)
- Does the verification suite run with a single command against the live stack?
- Does it cover the critical paths: permissions, versioning, public links, audit log immutability?
- Are failure messages meaningful enough to diagnose what broke?
- Is `VERIFICATION.md` clear enough for a reviewer who has never seen the project?

### 5. AI Code Review (15%)
- Was the review conducted critically, not superficially?
- Are findings categorised and prioritised — or is it an undifferentiated wall of suggestions?
- Is the response to each finding reasoned, not reflexive?
- Does `CODE_REVIEW.md` show that the candidate can distinguish a real problem from a style preference?

### 6. Developer Experience & Communication (10%)
- Does `docker compose up` actually work first try?
- Is `make seed` reliable and fast (under 60 seconds)?
- Are the READMEs clear enough for someone new to the codebase?
- Are trade-offs documented? Does the commit history tell a story?
- Is there a "what I'd do next" section in the README?

---

## What We Are NOT Evaluating

- Visual design — functional beats beautiful
- Completeness of bonus features — depth over breadth
- Frameworks or libraries you've never used before
- Test coverage above 70% — quality over quantity

---

## Submission

1. Share your work via **one of the following**:
   - Links to public repositories on GitHub or GitLab, or
   - A ZIP archive containing all repositories, sent to your hiring contact
2. Include a short cover note (3–5 sentences max) explaining:
   - The decision you're most proud of
   - The shortcut you took and why
   - One thing you'd architect differently with more time

**Deadline:** As agreed with your hiring contact.

---

## Technical Stack Summary

| Layer | Required | Optional / Bonus |
|---|---|---|
| Backend framework | Django 6+ + DRF | FastAPI for a secondary service |
| Task queue | Celery + Redis | — |
| Database | PostgreSQL | Elasticsearch (Bonus A) |
| File storage | Local (Docker volume) | Abstracted for S3/Azure swap |
| Frontend | Vue 3 + TypeScript + Vite | — |
| Real-time | — | Django Channels / SSE (Bonus C) |
| AI integration | — | — |
| Logging | structlog / JSON logging | — |
| Testing | pytest, pytest-django | Vitest, Playwright (frontend) |
| Infrastructure | Docker Compose | Separate infra repo |

---

## A Note From the Team

This task is deliberately overwhelming — by design. Nobody is expected to deliver everything, and we will not penalise you for leaving features out. What we are looking for is how you navigate that constraint: what you choose to build, what you consciously skip, and how clearly you communicate both.

The features we will look at most closely are the ones that reflect our daily work at SECJUR: the permission model, the audit trail, and the versioning logic. A submission that implements these three things well, with working verification and a thoughtful code review, is a strong submission — regardless of what else is or isn't there.

Spend your time budget wisely. A well-directed AI agent that ships a working, tested core is more impressive to us than an exhausted engineer who half-finishes everything.

Good luck, and feel free to reach out if something in the brief is genuinely ambiguous.

— The SECJUR Engineering Team
