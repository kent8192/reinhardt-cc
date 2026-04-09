# Reinhardt Feature Flags Reference

## Feature Presets

Reinhardt provides curated feature presets for common project types. Select a preset as a starting point, then add or remove individual features as needed.

| Preset | Features Included | Use Case |
|--------|-------------------|----------|
| `minimal` | `core`, `di`, `server` | Lightweight microservice with no database or auth |
| `standard` | `core`, `di`, `server`, `rest`, `api`, `middleware`, `sessions`, `commands` | Typical REST API backend with session support |
| `api-only` | `core`, `di`, `server`, `rest`, `api`, `middleware`, `commands`, `auth-jwt` | Stateless API with JWT authentication |
| `full` | All features enabled | Full-stack application with all capabilities |

## Database Features

Enable exactly one database backend feature. Each activates `reinhardt-db` with the appropriate driver.

| Feature | Database | Notes |
|---------|----------|-------|
| `db-postgres` | PostgreSQL | Recommended for production. Best ecosystem support. |
| `db-mysql` | MySQL / MariaDB | Compatible with MySQL 8.0+ and MariaDB 10.6+. |
| `db-sqlite` | SQLite | File-based. Good for development, prototyping, and embedded use. |
| `db-cockroachdb` | CockroachDB | PostgreSQL wire-compatible. For distributed SQL workloads. |

## Authentication Features

Auth features can be combined (e.g., JWT for API clients + Session for admin panel).

| Feature | Description |
|---------|-------------|
| `auth-jwt` | JSON Web Token authentication. Stateless, suitable for APIs and mobile clients. |
| `auth-session` | Cookie-based session authentication. Requires a session backend. |
| `auth-oauth` | OAuth 2.0 / OpenID Connect provider integration. |
| `auth-token` | Persistent token-based authentication (database-backed API keys). |
| `argon2-hasher` | Argon2id password hashing (recommended). Adds `argon2` dependency. |

## Individual Component Features

### Core Infrastructure

| Feature | Description |
|---------|-------------|
| `core` | Core framework types, configuration, and application lifecycle. Always required. |
| `di` | Dependency injection container (`InjectionContext`, `#[inject]`). |
| `server` | HTTP server (hyper-based) with graceful shutdown and signal handling. |
| `middleware` | Middleware pipeline (logging, CORS, security headers, compression). |
| `commands` | Management command framework (`reinhardt-admin` CLI integration). |

### API & Web

| Feature | Description |
|---------|-------------|
| `rest` | REST framework: serializers, parsers, renderers, content negotiation. |
| `api` | API views, ViewSets, routers, and pagination support. |
| `forms` | Form handling and validation (HTML form processing). |
| `graphql` | GraphQL schema, queries, mutations via `async-graphql` integration. |
| `pages` | Full-stack Pages support: WASM components, SSR, hydration, server functions. |
| `websockets` | WebSocket support for real-time communication. |

### Data & Caching

| Feature | Description |
|---------|-------------|
| `cache` | Cache framework with pluggable backends. |
| `redis-backend` | Redis cache and session backend. Requires a running Redis instance. |
| `sessions` | Server-side session framework. Requires a session backend (DB, Redis, or cookie). |

### Utilities

| Feature | Description |
|---------|-------------|
| `i18n` | Internationalization: message catalogs, locale detection, translation macros. |
| `mail` | Email sending framework with template support. |
| `admin` | Auto-generated admin interface for registered models. |

### Testing

| Feature | Description |
|---------|-------------|
| `test` | Test utilities: `ReinhardtTestCase`, request factories, assertion helpers. |
| `testcontainers` | TestContainers integration for database testing with real database instances. |

## Cargo.toml Examples

### API-Only with PostgreSQL + JWT

```toml
[dependencies]
reinhardt = {
    version = "0.1.0-alpha",
    default-features = false,
    features = [
        "core",
        "di",
        "server",
        "rest",
        "api",
        "middleware",
        "commands",
        "db-postgres",
        "auth-jwt",
        "argon2-hasher",
    ]
}

[dev-dependencies]
reinhardt = {
    version = "0.1.0-alpha",
    features = ["test", "testcontainers"]
}
```

### Full-Stack with Pages

```toml
[dependencies]
reinhardt = {
    version = "0.1.0-alpha",
    default-features = false,
    features = [
        "core",
        "di",
        "server",
        "rest",
        "api",
        "forms",
        "pages",
        "middleware",
        "sessions",
        "commands",
        "db-postgres",
        "auth-session",
        "argon2-hasher",
        "cache",
        "redis-backend",
        "i18n",
        "admin",
    ]
}

[dev-dependencies]
reinhardt = {
    version = "0.1.0-alpha",
    features = ["test", "testcontainers"]
}
```

### Minimal Microservice

```toml
[dependencies]
reinhardt = {
    version = "0.1.0-alpha",
    default-features = false,
    features = [
        "core",
        "di",
        "server",
    ]
}

[dev-dependencies]
reinhardt = {
    version = "0.1.0-alpha",
    features = ["test"]
}
```
