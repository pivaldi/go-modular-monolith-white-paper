# Technical Reference for AI Agents — MMW Go Monorepo

> **Audience**: AI coding agents (Claude, Gemini, etc.) working on this codebase.
> **Purpose**: Comprehensive reference to understand, navigate, and extend this monorepo without human guidance.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [libs/platform — The Platform Library](#3-libsplatform--the-platform-library)
4. [modules/auth — Authentication & Identity](#4-modulesauth--authentication--identity)
5. [modules/todo — Domain Service](#5-modulestodo--domain-service)
6. [modules/notifications — Event Consumer](#6-modulesnotifications--event-consumer)
7. [modules/todo/web — Angular Frontend](#7-modulestodoweb--angular-frontend)
8. [Cross-Cutting Concerns](#8-cross-cutting-concerns)
9. [Event-Driven Architecture](#9-event-driven-architecture)
10. [Developer Guide](#10-developer-guide)
11. [Key Patterns Reference](#11-key-patterns-reference)

---

## 1. Project Overview

### Architecture Philosophy

- **Modular Monolith**: All services run as isolated `Module` implementations within a single process (`cmd/mmw`), coordinated by the `mmw/platform` runner.
- **Clean Architecture / Hexagonal**: Strict separation between Domain logic, Application use cases, and Infrastructure adapters. Dependencies always flow inward.
- **Domain-Driven Design (DDD)**: Business logic resides in Aggregate Roots. State transitions are governed by domain methods that enforce invariants and emit Domain Events.
- **Protocol-First API**: API contracts are defined in Protobuf (`contracts/proto`) and served via **Connect RPC**.
- **Event-Driven via Transactional Outbox**: Asynchronous inter-module communication using an outbox table to guarantee atomicity between database writes and event publishing.
- **Architecture Enforcement**: `arch-go` and custom tools in `tools/arch-test` ensure layer boundaries are respected.

### Goals

- Reusable Go platform library (`github.com/piprim/mmw/platform`) for consistent service building.
- Reliable event propagation using outbox + in-memory bus (Watermill).
- Seamless integration of multiple domains (`auth`, `todo`) in a single repo.
- Type-safe communication between Go backend and Angular frontend via ConnectRPC.

### Technology Stack

| Concern | Technology |
| :--- | :--- |
| Language | Go 1.26.1 (Go Workspaces) |
| API Framework | Connect RPC + Protocol Buffers |
| Database | PostgreSQL via `pgx/v5` |
| Migrations | Goose |
| Event Bus | Watermill (GoChannel — in-memory) |
| Configuration | Layered TOML + Env vars via `mmw/platform/config` |
| Logging | `slog` + `lmittmann/tint` (dev) / JSON (prod) |
| Error Handling | `rotisserie/eris` (structured stack traces) |
| Task Runner | Mise |
| Frontend | Angular 17 / TypeScript |
| Architecture Test| `arch-go` |
| Symlink Management| GNU Stow |

---

## 2. Repository Structure

```text
/workspace
├── go.work                      # Go Workspace — root of all modules
├── go.mod                       # Root module (github.com/pivaldi/mmw)
├── buf.yaml                     # Protocol Buffer toolchain config
├── mise.toml                    # Task runner (build, test, lint, run)
├── STOW.md                      # Guide for script/config sharing via Stow
│
├── cmd/
│   └── mmw/                     # Monolith entry point (composition root)
│       └── main.go              # Wires all modules (auth, todo, notifications)
│
├── libs/
│   ├── ogl/                     # OVYA Go Library — file, os, string utilities only
│   │   ├── file/
│   │   ├── os/
│   │   └── string/
│   └── platform/                # github.com/piprim/mmw/platform — shared platform primitives
│       ├── config/              # Layered config loader (Context-aware)
│       ├── db/outbox/           # Transactional outbox relay
│       ├── pg/uow/              # Unit of Work pattern for PostgreSQL
│       ├── core/module.go       # Module interface
│       ├── connect/             # ConnectRPC interceptors (logging, etc.)
│       ├── events/              # Watermill bus abstraction
│       ├── server/              # HTTP server with health checks & pprof
│       └── slog/                # Structured logging handlers
│
├── modules/
│   ├── auth/                    # Auth domain service
│   │   ├── auth.go              # Module implementation
│   │   └── internal/            # DDD layers: domain, application, adapters
│   │
│   ├── todo/                    # Todo domain service
│   │   ├── todo.go              # Module implementation
│   │   ├── internal/            # DDD layers: domain, application, adapters
│   │   └── web/                 # Frontend assets and Angular app
│   │
│   └── notifications/           # Notifications consumer
│       └── notifications.go     # Subscribes to auth and todo events
│
├── contracts/                   # Shared Protobuf definitions + generated code
│   ├── proto/
│   │   ├── auth/v1/auth.proto
│   │   └── todo/v1/todo.proto
│   ├── gen/go/                  # Generated Go stubs
│   └── gen/ts/                  # Generated TypeScript stubs
│
├── stow/
│   └── common/                  # Shared configs (.golangci.yml) and scripts
│
└── tools/
    └── arch-test/               # Architectural linting tool (wraps arch-go)
```

---

## 3. libs/platform — The Platform Library

**Module**: `github.com/piprim/mmw/platform` (at `poc/libs/platform/`)

> The `ogl` library (`poc/libs/ogl/`) now only contains `file/`, `os/`, and `string/` utilities. All platform, config, database, and observability packages live here.

### `config` — Layered Configuration

**Purpose**: Fills a struct with values from `configs/*.toml` and environment variables.
**API**: `config.NewContext(ctx, fs, prefix).Fill(cfg)`
**Import**: `github.com/piprim/mmw/platform/config`
- Prefers env vars over TOML.
- Supports `mapstructure` tags.

### `pg/uow` — Unit of Work

**Purpose**: Manages PostgreSQL transactions via `context.Context`.
**Import**: `github.com/piprim/mmw/platform/pg/uow`
- `uow.WithTransaction(ctx, fn)`: Starts a transaction and passes it via context.
- `uow.GetExecutor(ctx, pool)`: Returns the active transaction or the pool.

### `server` — HTTP Server

**Import**: `github.com/piprim/mmw/platform/server`
- Built-in `/healthz` and `/readyz` endpoints.
- Development-only `/debug/pprof` routes.
- Middleware: Logging (with RequestID), Recovery, CORS.

### `db/outbox` — Events Relay

**Import**: `github.com/piprim/mmw/platform/db/outbox`
- `EventsRelay`: Polls a database table (e.g., `todo.event`) and publishes to the `SystemEventBus`.
- Ensures at-least-once delivery.

---

## 4. modules/auth — Authentication & Identity

### Overview
Handles user registration, login, and token validation.

### API (`auth.v1.AuthService`)
- `Register`: Creates a new user.
- `Login`: Returns a JWT token.
- `ValidateToken`: Verifies JWT and returns `user_id`.
- `ChangePassword`, `DeleteUser`.

### Integration
The `auth` module is passed as an `inprocClient` to other modules (like `todo`) to allow direct token validation without network overhead.

---

## 5. modules/todo — Domain Service

### Architecture
Follows strict Clean Architecture. The `Todo` aggregate root manages state transitions (`pending` → `in_progress` → `completed`).

### Authentication Middleware
All `TodoService` RPCs are protected by `AuthMiddleware`. It calls `AuthSvc.ValidateToken` for every request.

### Domain Events
Emits events for every state change: `TodoCreated`, `TodoUpdated`, `TodoCompleted`, etc. These are written to the `todo.event` table by the outbox dispatcher within the same transaction.

Each event's `EventType()` returns a semantic domain constant (`"todo.created"`, `"todo.updated"`, etc.) owned by the domain package — not a routing key from contracts. The outbound adapter (`adapters/outbound/events/topics.go`) maps these to Watermill routing keys before writing to the outbox table.

---

## 6. modules/notifications — Event Consumer

### Overview
A reactive module that listens to topics from the `SystemEventBus`.
- Subscribes to `auth` events (e.g., `UserRegistered`).
- Subscribes to `todo` events (e.g., `TodoCreated`).
- In this POC, it logs received events; in production, it would dispatch emails/push notifications.

---

## 7. modules/todo/web — Angular Frontend

### Overview
Angular 17 SPA located in `modules/todo/web/todoapp`.
- Communication via **ConnectRPC TS Client**.
- Shared types from `contracts/gen/ts`.

### Key Components
- `AuthService`: Manages login/token storage.
- `TodoService`: ConnectRPC client with an interceptor to inject the `Authorization: Bearer <token>` header.
- `TimestampPipe`: Converts Protobuf `google.protobuf.Timestamp` to JS `Date`.

---

## 8. Cross-Cutting Concerns

### Error Handling
- Use `rotisserie/eris` for error wrapping: `eris.Wrap(err, "context")`.
- `platform.NewErrorLoggingInterceptor` logs detailed stack traces for RPC failures.
- Domain errors flow through three layers:
  1. **Domain** — sentinel errors (`var ErrNotFound = errors.New("...")`). No proto or Connect dependency.
  2. **Application** — `DomainErrorFor()` maps sentinels to `*platform.DomainError{Code, Message}`. The `ErrorCode` type is owned by the application package with explicit integer values (matching the proto enum so the wire format stays stable).
  3. **Inbound adapter** — `connectErrorFrom()` maps `platform.ErrorCode` to a Connect status code via a lookup table (`authConnectCodeMap`, `domainConnectCodeMap`). Attaches a `commonv1.DomainError` proto detail so TypeScript clients can identify specific errors.

### Architecture Testing
- Each module has an `arch-go.yml` defining allowed dependencies.
- `tools/arch-test` runs these checks.
- **Rule**: No cross-module imports (except via `contracts` or `inproc` interfaces).

### Logging
- Use `slog` with module-specific attributes: `logger.With("module", "Todo")`.
- `oglslog` provides a colorized text handler for dev and JSON for prod.

---

## 9. Event-Driven Architecture

### Transactional Outbox
1. **Application Layer**: Starts transaction.
2. **Repository**: Saves Aggregate + Writes Event to `outbox` table.
3. **Application Layer**: Commits transaction.
4. **Relay (Background)**: Polls `outbox`, publishes to Watermill.
5. **Subscribers**: Receive and process event.

> **Topic resolution:** Domain events carry a semantic `EventType()` string (e.g., `"todo.created"`). The outbox dispatcher translates this to the Watermill routing key (e.g., `"todo.userTasks.created.v1"`) via a `domainTopics` lookup table in `adapters/outbound/events/topics.go`. The relay publishes using the stored routing key directly.

---

## 10. Developer Guide

### Running the Monolith
```bash
go run cmd/mmw/main.go
```

### Adding a New RPC
1. Edit `contracts/proto/<module>/v1/<module>.proto`.
2. Run `buf generate`.
3. Implement handler in `internal/adapters/inbound/connect`.
4. Update application service and repository if needed.

### Running Arch Tests
```bash
./tools/arch-test/arch-test.sh
```

---

## 11. Key Patterns Reference

| Pattern | Location |
| :--- | :--- |
| **Module Entry** | `modules/<name>/<name>.go` |
| **Unit of Work** | `libs/platform/pg/uow` |
| **Outbox Relay** | `libs/platform/db/outbox` |
| **Auth Middleware** | `modules/todo/internal/adapters/inbound/connect/middleware.go` |
| **Config Injection** | `cmd/mmw/main.go` using `config.Load()` |
| **Shared Scripts** | `stow/common/scripts` (symlinked via Stow) |

---

## AI Prompting Tips

- *"Follow the **Transactional Outbox** pattern in `modules/todo` to emit a new `TodoArchived` event."*
- *"Implement a new Connect RPC method in `auth.proto` and regenerate stubs using `buf`."*
- *"Add an architectural rule to `modules/auth/arch-go.yml` to prevent `domain` from importing `slog`."*
- *"Using `github.com/piprim/mmw/platform/pg/uow`, implement a multi-repository operation in the `TodoApplicationService`."*
