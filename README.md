# DSN — Digital School Note

A smart school platform for collaborative note-taking: teachers and students create notes, draw on a shared canvas, and (in later phases) edit together in real time — with multi-tenant school management, OneRoster sync, and Japanese/English i18n.

Built as a full rebuild in strict TypeScript: **NestJS + Next.js 15 + PostgreSQL + Prisma + Redis**.

> **Status:** 🚧 Phase 0 (MVP / Proof of Concept) — in progress. See [Roadmap](#roadmap).

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Development Conventions](#development-conventions)
- [Roles & Authorization](#roles--authorization)
- [Roadmap](#roadmap)
- [Documentation](#documentation)
- [License](#license)

---

## Overview

DSN replaces a legacy JavaScript / Express / Vue / MySQL stack with a typed, modular architecture designed for:

- **Collaborative canvas notes** — Fabric.js drawing (pen, shapes, text), multi-page notes, and real-time co-editing.
- **Multi-tenant isolation** — each municipality/region is a tenant; tenant data never crosses boundaries.
- **School management** — tenants, schools, users, groups (classes), and channels.
- **Interoperability** — LTI 1.3 launch and OneRoster rostering sync.
- **i18n from day one** — Japanese (`ja`, default) and English (`en`).

### Phase 0 goal (MVP)

> Run `docker compose up -d`, start Api and FE, then in the browser: **register/login → create a note → draw on the canvas → save → see it in the list → reopen and keep editing** — with strict TypeScript, structured logging, and passing tests.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | TypeScript (strict, no `any`) |
| Backend | NestJS (modules, DI, guards, interceptors, pipes) |
| Frontend | Next.js 15 (App Router, Server Components) |
| ORM | Prisma |
| Database | PostgreSQL 17 (JSONB for canvas pages) |
| Cache / Session / Queue backend | Redis 7 |
| Queue | BullMQ (later phases) |
| Real-time | Socket.IO |
| Canvas | Fabric.js |
| Auth | Local (Phase 0) → OIDC Google/Azure AD → LTI 1.3 · JWT in httpOnly cookies |
| Validation | Joi (env), class-validator + class-transformer (Api), Zod (Next.js) |
| i18n | nestjs-i18n (Api), next-intl (FE/Admin) |
| Logging | Winston via nest-winston (structured JSON, daily rotation) |
| Testing | Jest, Supertest, Pact (contract) |
| Code quality | ESLint, Prettier, Husky + lint-staged |
| Infrastructure | Docker Compose |
| Runtime | Node.js 24, npm |

---

## Architecture

### Monorepo layout

```
digital-school-note/
├── api/                   # NestJS — REST API + Socket.IO gateway        (Phase 0)
├── FE/                    # Next.js — teacher & student app              (Phase 0)
├── Admin/                 # Next.js — internal admin dashboard           (Phase 1)
├── Rostering/             # NestJS worker — OneRoster sync               (Phase 3)
├── docker-compose.yml     # Shared infrastructure: PostgreSQL + Redis
├── .env.example           # Infra-scope variables (read by docker-compose)
├── package.json           # Root tooling only: Husky + lint-staged
├── lint-staged.config.mjs # Pre-commit lint/format per sub-project
└── .husky/pre-commit
```

Each sub-project owns its own `package.json`, `.env`, and `.env.example`. The root `.env` holds **infrastructure credentials only**; application variables live in each sub-project.

### Api structure (target)

```
api/src/
├── main.ts
├── app.module.ts
├── config/          # Joi env validation + typed AppConfig
├── common/
│   ├── guards/      # JwtAuthGuard, RolesGuard, TenantGuard, OwnershipGuard
│   ├── interceptors/# LoggingInterceptor (requestId/userId/tenantId)
│   ├── filters/     # HttpExceptionFilter
│   ├── decorators/  # @CurrentUser, @Roles, @Public
│   ├── constants/   # Enums and named constants (no magic values)
│   └── logger/      # Winston configuration
├── modules/         # auth, user, note, canvas (Phase 0) · tenant, school, group, channel, admin (later)
├── prisma/          # PrismaService, schema, migrations, seed
├── socket/          # Socket.IO gateway
└── i18n/            # ja/, en/
```

### Request authorization chain

```
Request → JwtAuthGuard → RolesGuard → TenantGuard → OwnershipGuard → Controller
```

Tenant isolation is enforced at **both** the guard layer and the query layer (every tenant-owned query is scoped by `tenantId`).

### Multi-tenancy

| Database | Contents | Scope |
|---|---|---|
| **UM** — User Management | Tenants, schools, users, auth | One shared database |
| **FM** — Feature Management | Channels, notes, canvas | One schema per tenant (by region) |

Tenant resolution comes from a JWT claim. **Phase 0 runs single-tenant**, but every tenant-owned table already carries `tenantId` and data access goes through a single `PrismaService`, so the per-tenant split in Phase 1 slots in without reshaping existing tables.

### Default ports

| Service | Port |
|---|---|
| Api (HTTP) | 3000 |
| Api (WebSocket) | 3001 |
| Admin | 3100 |
| FE | 3200 |
| Rostering | 3300 |
| PostgreSQL | 5432 |
| Redis | 6379 |

---

## Getting Started

### Prerequisites

- Node.js **24** and npm (recommended via [nvm](https://github.com/nvm-sh/nvm))
- Docker with Docker Compose v2+

### 1. Clone and install root tooling

```bash
git clone git@github.com:dzulfikriAlfik/digital-school-note.git
cd digital-school-note
npm install          # installs Husky + lint-staged and activates the pre-commit hook
```

### 2. Start infrastructure

```bash
cp .env.example .env # then edit credentials
docker compose up -d
docker compose ps    # both containers should report (healthy)
```

### 3. Run the Api

```bash
cd api
npm install
cp .env.example .env # then fill in the required values
npm run start:dev
```

The Api listens on `http://localhost:3000`. Logs print to the console and are written as JSON to `api/logs/`.

> Database migrations, seed data, and the FE app will be added to this guide as their Phase 0 steps land.

---

## Environment Variables

### Root `.env` — infrastructure (read by `docker-compose.yml`)

| Variable | Example | Description |
|---|---|---|
| `POSTGRES_USER` | `dsn_user` | PostgreSQL user |
| `POSTGRES_PASSWORD` | — | PostgreSQL password |
| `POSTGRES_DB` | `dsn_um` | Initial database (UM) |
| `POSTGRES_PORT` | `5432` | Host port for PostgreSQL |
| `REDIS_PORT` | `6379` | Host port for Redis |

### `api/.env` — application

Validated by Joi at startup: the Api **refuses to boot** if a required variable is missing or malformed.

| Variable | Required | Description |
|---|---|---|
| `NODE_ENV` | — | `development` \| `production` \| `test` |
| `PORT` | — | HTTP port (default `3000`) |
| `DEFAULT_LOCALE` | — | `ja` (default) or `en` |
| `SUPPORTED_LOCALES` | — | Comma-separated, e.g. `ja,en` |
| `LOG_LEVEL` | — | `error` \| `warn` \| `info` \| `debug` \| `verbose` |
| `DATABASE_URL` | ✅ | PostgreSQL connection string |
| `REDIS_URL` | ✅ | Redis connection string |
| `JWT_SECRET` | ✅ | Access-token secret (min 32 chars) |
| `JWT_REFRESH_SECRET` | ✅ | Refresh-token secret (min 32 chars) |
| `GOOGLE_*`, `AZURE_AD_*`, `LTI_*` | later | Reserved for OIDC / LTI (Phase 1+) |

Never commit real `.env` files — only `.env.example` is tracked.

---

## Development Conventions

### Code rules

- **TypeScript strict** — `strict` plus `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitReturns`, `noImplicitOverride`, `noUnusedLocals/Parameters`.
- **Never `any`** — use `unknown` with type guards. Enforced by `no-explicit-any` and all `no-unsafe-*` ESLint rules.
- **Explicit types** on all parameters and return values.
- **No magic values** — enums and named constants only.
- **JSDoc** on every function (`@param` / `@returns` / `@throws`); prose descriptions for non-trivial logic. Comments in English.
- **Prisma discipline** — `select` only needed fields, paginate every list, scope by `tenantId` + ownership, soft-delete via `deletedAt`.
- **Security** — tokens in httpOnly cookies only (never `localStorage`); no secrets or PII in logs.
- **i18n** — no hard-coded user-facing strings; every key exists in both `ja` and `en`.

### Quality gates

- Pre-commit hook runs ESLint (`--fix`) and Prettier on staged files; commits with violations are rejected.
- Every service method has a unit test; every endpoint has an integration test.
- Coverage targets: services ≥ 80%, controllers ≥ 60%.

### Git workflow

- Main branch: `main`
- [Conventional Commits](https://www.conventionalcommits.org/): `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`, `perf:`
- Small, logical commits — one verified unit per commit.

---

## Roles & Authorization

| Code | Role | Scope |
|---|---|---|
| `0` | Super Admin | Entire platform, all tenants |
| `1` | Teacher | Own notes + channels they belong to |
| `2` | Student | Own notes + channels they belong to |
| `3` | Tenant Admin | One municipality/region |
| `4` | School Admin | One school |

Critical rules:

- A user from Tenant A **must never** access Tenant B's data.
- A student or teacher **must never** CRUD another user's private notes.

---

## Roadmap

### Phase 0 — MVP / Proof of Concept 🚧

- [ ] **Step 1 — Infrastructure:** Docker Compose (PostgreSQL + Redis), git guards, env templates
- [ ] **Step 2 — Api foundation**
  - [ ] NestJS scaffold + strict tsconfig
  - [ ] ESLint, Prettier, Husky + lint-staged
  - [ ] Winston structured logging (console + rotating JSON files)
  - [ ] Joi env validation with typed config
  - [ ] Prisma schema (User, Channel, Note) + first migration + seed; slow-query logging
  - [ ] Redis connection module
  - [ ] nestjs-i18n (ja/en)
  - [ ] Global validation pipe, exception filter, health endpoint
- [ ] **Step 3 — Auth + User:** local login, JWT httpOnly cookies, refresh, logout, user CRUD, request-context logging, tests
- [ ] **Step 4 — Note + Canvas:** ownership-scoped note CRUD, canvas save/load, Socket.IO gateway, cross-user access tests
- [ ] **Step 5 — FE:** Next.js 15, next-intl, API client, route protection, login, note list, Fabric.js canvas (pen, text, rectangle, circle, triangle, line, selection, color, stroke width)
- [ ] **Step 6 — Quality gates:** coverage thresholds, zero `any` / magic values audit, full Definition of Done walkthrough

### Phase 1 — Core Foundation

- [ ] Multi-tenant data: Tenant model, per-tenant FM schemas, tenant provisioning, `forTenant(region)`
- [ ] TenantGuard + query-level tenant scoping
- [ ] Channel CRUD + membership; full OwnershipGuard
- [ ] OIDC (Google, Azure AD), token rotation, multi-session detection + forced logout
- [ ] Redis caching layer
- [ ] Admin app: tenant / school / user / group management with admin scope enforcement
- [ ] Cross-tenant, cross-user, and admin-scope tests; Pact contract tests

### Phase 2 — Canvas & Collaboration

- [ ] Full toolbar (29 tools) + keyboard shortcuts
- [ ] Multi-page notes; undo/redo; group, lock, and layer ordering
- [ ] Grid / ruled backgrounds; image import + crop; PDF export
- [ ] CRDT-based real-time collaborative editing with presence

### Phase 3 — Advanced Features

- [ ] LTI 1.3 (launch, read-only sessions, deep linking)
- [ ] Rostering service: OneRoster sync (daily cron + manual trigger with progress UI)
- [ ] Teacher monitoring + blackout mode; presentation mode with laser pointer
- [ ] Stamp/sticker library; CSV bulk import/export; activity logs + audit trail

### Phase 4 — Polish & Production

- [ ] Performance (p95 < 200 ms) and bundle size (< 200 KB initial)
- [ ] OWASP Top 10 audit + cross-tenant penetration testing
- [ ] Load testing (k6, 1000+ concurrent connections) and E2E tests (Playwright)
- [ ] Production Docker (multi-stage, nginx, SSL/TLS), monitoring & alerting, VPS deployment

---

## Documentation

| File | Purpose |
|---|---|
| `PRD.md` | Product requirements (v2.0) |
| `AGENTS.md` | Coding standards and instructions for AI-assisted development |
| `PHASE-TRACKER.md` | Spec-level phase checklists |
| `PROGRESS-TRACKER.md` | Step-by-step build progress, commits, and decisions |

---

## License

Proprietary — © Pantona Inc. All rights reserved. Unauthorized copying, distribution, or use is prohibited.