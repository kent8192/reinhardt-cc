# Reinhardt Project Templates Reference

## Template Types

The `reinhardt-admin startproject` command supports two template types via the `-t` / `--template-type` flag:

### `restful` (Default)

REST API backend project. This is the default when no flag is specified.

- Generates a project structured for JSON API development
- Includes `config/settings.rs` with TOML-based environment configuration
- Pre-configures JSON renderer and parser
- No frontend asset pipeline

### `mtv` (Model-Template-View)

Full-stack application with WASM + SSR support (reinhardt-pages).

- Generates a project with both server and client-side code
- Includes Pages component infrastructure for WASM rendering
- Configures server-side rendering (SSR) with hydration
- Includes server function support for RPC-style client-server communication
- Adds `build.rs` and `index.html` for WASM compilation
- Includes `client/` module for WASM routing and state management
- Includes `shared/` module for types shared between server and client

## CLI Usage

```bash
# Create a RESTful API project (default)
reinhardt-admin startproject my_project

# Explicitly specify RESTful template
reinhardt-admin startproject my_project -t restful

# Create a Pages (MTV) project
reinhardt-admin startproject my_project -t mtv

# Create an app (RESTful, default)
reinhardt-admin startapp my_app

# Create a Pages app
reinhardt-admin startapp my_app -t mtv
```

## Template Variables

The scaffolding engine substitutes these variables in generated files:

| Variable | Description | Example |
|----------|-------------|---------|
| `project_name` | The project name as provided by the user | `my_blog` |
| `crate_name` | Sanitized crate name (same as `project_name`, validated) | `my_blog` |
| `camel_case_project_name` | PascalCase version for struct/type names | `MyBlog` |
| `app_name` | App name (for `startapp`) | `user` |
| `camel_case_app_name` | PascalCase app name | `User` |

## Generated Project Structure

### RESTful Template

```
my_project/
├── .gitignore
├── bacon.toml
├── Cargo.toml
├── Makefile.toml
├── README.md
├── settings/
│   ├── .gitignore
│   ├── base.example.toml
│   ├── local.example.toml
│   ├── production.example.toml
│   └── staging.example.toml
└── src/
    ├── lib.rs               # pub mod config; pub mod apps;
    ├── bin/
    │   └── manage.rs        # Management CLI entry point
    ├── apps.rs              # App module declarations (initially empty)
    ├── config.rs            # pub mod urls; pub mod settings;
    └── config/
        ├── settings.rs      # TOML-based settings with environment profiles
        └── urls.rs          # #[routes] fn routes() -> UnifiedRouter
```

### MTV (Pages) Template

```
my_project/
├── .gitignore
├── bacon.toml
├── build.rs                 # WASM build configuration
├── Cargo.toml
├── index.html               # HTML shell for WASM app
├── Makefile.toml
├── README.md
├── settings/
│   └── (same as RESTful)
└── src/
    ├── lib.rs
    ├── bin/
    │   └── manage.rs
    ├── apps.rs
    ├── config.rs
    ├── config/
    │   ├── settings.rs
    │   └── urls.rs
    ├── client.rs            # WASM client module
    ├── client/
    │   ├── router.rs        # Client-side routing
    │   └── state.rs         # Client state management
    ├── server/
    │   └── server_fn.rs     # Server functions
    ├── shared.rs            # Shared types module
    └── shared/
        ├── errors.rs        # Shared error types
        └── types.rs         # Shared data types
```

## Generated App Structure

### RESTful App

```
<app_name>/
├── lib.rs           # #[app_config] + module declarations
├── admin.rs         # Admin configuration
├── admin/           # (gitkeep for future admin files)
├── models.rs        # Model definitions
├── models/          # (gitkeep for future model files)
├── serializers.rs   # Serializer definitions
├── serializers/     # (gitkeep for future serializer files)
├── urls.rs          # URL routing (ServerRouter)
├── views.rs         # View functions
├── views/           # (gitkeep for future view files)
├── tests.rs         # App tests
└── tests/           # (gitkeep for future test files)
```

## Post-Scaffolding Checklist

After running `reinhardt-admin startproject <name>`, complete these steps:

1. **Review `Cargo.toml`** — Verify feature flags match your requirements. Add or remove features based on your database backend, auth method, and component needs. See `feature-flags.md` for details.

2. **Set up settings files** — Copy example TOML files and configure:
   ```bash
   cp settings/base.example.toml settings/base.toml
   cp settings/local.example.toml settings/local.toml
   ```
   Configure database connection, secret key, and other environment-specific settings.

3. **Verify compilation**:
   ```bash
   cargo check --all-features
   ```

4. **Format generated code**:
   ```bash
   cargo fmt --all
   ```

5. **Initialize Git** (if not already):
   ```bash
   git init
   git add .
   git commit -m "chore: initialize reinhardt project"
   ```
