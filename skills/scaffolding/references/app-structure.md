# Reinhardt App Structure Reference

## Standard Project Layout (RESTful)

A reinhardt project uses `lib.rs` + `bin/manage.rs` as entry points, NOT `main.rs`. The Rust 2024 Edition module system (`module.rs` + `module/` directory, NEVER `mod.rs`) is used throughout.

```
my_project/
├── Cargo.toml
├── settings/                    # TOML-based configuration files
│   ├── base.toml                # Common settings across all environments
│   ├── local.toml               # Local development settings
│   ├── staging.toml             # Staging environment settings
│   └── production.toml          # Production environment settings
├── src/
│   ├── lib.rs                   # Library crate entry point
│   ├── bin/
│   │   └── manage.rs            # Management CLI (equivalent to Django's manage.py)
│   ├── apps.rs                  # App module declarations
│   ├── config.rs                # Configuration module entry point
│   ├── config/
│   │   ├── settings.rs          # Project settings (TOML-based, environment-specific)
│   │   └── urls.rs              # Root URL configuration (#[routes] + UnifiedRouter)
│   └── apps/
│       ├── user.rs              # User app entry point (#[app_config])
│       ├── user/
│       │   ├── admin.rs         # Admin configuration
│       │   ├── models.rs        # Model definitions
│       │   ├── views.rs         # View functions / ViewSets
│       │   ├── serializers.rs   # Serializer definitions
│       │   ├── urls.rs          # App-level URL routing (ServerRouter)
│       │   └── tests.rs         # App-level tests
│       ├── post.rs              # Post app entry point
│       └── post/
│           ├── admin.rs
│           ├── models.rs
│           ├── views.rs
│           ├── serializers.rs
│           ├── urls.rs
│           └── tests.rs
└── tests/
    └── integration/
        └── api_tests.rs         # Cross-app integration tests
```

### Pages (MTV) Project Layout

Pages projects add WASM client-side code and shared modules:

```
my_project/
├── Cargo.toml
├── build.rs                     # WASM build configuration
├── index.html                   # HTML shell for WASM app
├── settings/                    # Same as RESTful
├── src/
│   ├── lib.rs                   # Library crate entry point
│   ├── bin/
│   │   └── manage.rs            # Management CLI
│   ├── apps.rs                  # App module declarations
│   ├── config.rs                # Configuration module
│   ├── config/
│   │   ├── settings.rs          # Project settings
│   │   └── urls.rs              # Root URL configuration
│   ├── client.rs                # WASM client module
│   ├── client/
│   │   ├── router.rs            # Client-side routing
│   │   └── state.rs             # Client state management
│   ├── server/
│   │   └── server_fn.rs         # Server functions
│   ├── shared.rs                # Shared types module
│   ├── shared/
│   │   ├── errors.rs            # Shared error types
│   │   └── types.rs             # Shared data types
│   └── apps/
│       └── ...                  # Same as RESTful + client/ modules
```

## Entry Points

### `lib.rs` — Library crate entry point

```rust
//! my_project library

pub mod config;
pub mod apps;

// Re-export commonly used items
pub use config::settings::get_settings;
pub use config::urls::routes;
```

### `bin/manage.rs` — Management CLI

```rust
use my_project as _;
use reinhardt::commands::execute_from_command_line;
use std::process;

#[tokio::main]
async fn main() {
    unsafe {
        std::env::set_var("REINHARDT_SETTINGS_MODULE", "my_project.config.settings");
    }

    if let Err(e) = execute_from_command_line().await {
        eprintln!("Error: {}", e);
        process::exit(1);
    }
}
```

**Key points:**
- `use my_project as _` imports the library crate to register `#[routes]` and `#[app_config]` macros
- `REINHARDT_SETTINGS_MODULE` env var tells the framework where to find settings
- Router registration happens automatically via the `#[routes]` attribute macro

## Module System Rules

| Pattern | Correct | Wrong |
|---------|---------|-------|
| Module with submodules | `apps.rs` + `apps/` directory | `apps/mod.rs` |
| Nested module | `apps/user.rs` + `apps/user/` directory | `apps/user/mod.rs` |
| Simple module (no children) | `settings.rs` (single file) | `settings/mod.rs` |
| Re-exports | `pub use models::User;` | `pub use models::*;` |

### Key Rules

- **ALWAYS** use `module.rs` + `module/` directory for modules with submodules
- **NEVER** use `mod.rs` files (deprecated in Rust 2024 Edition)
- **NEVER** use glob re-exports (`pub use module::*`) except `use super::*` in test modules
- Maximum nesting depth: 4 levels
- Use `pub use` in module entry points to control the public API surface

## App Configuration

### `#[app_config]` Macro

Each app defines a config struct using the `#[app_config]` attribute macro:

```rust
// src/apps/user.rs — App entry point
use reinhardt::app_config;

pub mod admin;
pub mod models;
pub mod serializers;
pub mod urls;
pub mod views;

#[cfg(test)]
mod tests;

#[app_config(name = "user", label = "user")]
pub struct UserConfig;
```

### `installed_apps!` Macro

Register apps in `src/config/apps.rs` (or equivalent) using the `installed_apps!` macro:

```rust
// src/config/apps.rs
use reinhardt::installed_apps;

// Register user-defined apps for discovery and configuration.
// Framework features (auth, sessions, etc.) are enabled via Cargo feature flags.
installed_apps! {
    user: "user",
    post: "post",
}

/// Get the list of installed applications
pub fn get_installed_apps() -> Vec<String> {
    InstalledApp::all_apps()
}
```

**Important:**
- `installed_apps!` is for **user applications only** — framework features are enabled via Cargo feature flags
- The macro generates an `InstalledApp` enum with `all_apps()` and `path()` methods
- App names in `installed_apps!` must match the `name` in `#[app_config]`

## Adding a New App

Follow this procedure to add a new app to an existing project:

1. **Generate the app scaffold**:
   ```bash
   reinhardt-admin startapp <name> [-t restful|mtv]
   ```

2. **Verify the generated structure** matches the layout above. The generated app includes:
   - `lib.rs` — App entry point with `#[app_config]` macro
   - `admin.rs` — Admin configuration
   - `models.rs` — Model definitions
   - `serializers.rs` — Serializer definitions
   - `views.rs` — View functions
   - `urls.rs` — URL routing (`ServerRouter`)
   - `tests.rs` — App tests

3. **Register the app module** in `src/apps.rs`:
   ```rust
   pub mod <name>;
   ```

4. **Register the app** in `installed_apps!` macro (in `src/config/apps.rs` or equivalent):
   ```rust
   installed_apps! {
       // existing apps...
       <name>: "<name>",
   }
   ```

5. **Mount app routes** in the root URL configuration (`src/config/urls.rs`):
   ```rust
   #[routes]
   pub fn routes() -> UnifiedRouter {
       UnifiedRouter::new()
           .mount("/api/", crate::apps::<name>::urls::url_patterns())
   }
   ```

6. **Verify compilation**:
   ```bash
   cargo check
   ```

### Pages App Modules

Pages (MTV) apps include additional modules for WASM support:

```rust
// src/apps/my_app.rs — Pages app entry point
#[cfg(native)]
use reinhardt::app_config;

#[cfg(native)]
pub mod admin;
#[cfg(wasm)]
pub mod client;          // WASM client components
#[cfg(native)]
pub mod models;
#[cfg(native)]
pub mod serializers;
pub mod server;          // Server functions (available on both native and WASM)
pub mod shared;          // Shared types (available on both)
#[cfg(native)]
pub mod urls;
#[cfg(native)]
pub mod views;

#[cfg(native)]
#[app_config(name = "my_app", label = "my_app")]
pub struct MyAppConfig;
```

- `#[cfg(native)]` — Server-only modules (models, views, admin, etc.)
- `#[cfg(wasm)]` — WASM-only modules (client components)
- No annotation — Available on both platforms (server functions, shared types)
