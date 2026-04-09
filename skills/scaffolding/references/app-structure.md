# Reinhardt App Structure Reference

## Standard Project Layout

A reinhardt project uses the Rust 2024 Edition module system (`module.rs` + `module/` directory, NEVER `mod.rs`).

```
my_project/
├── Cargo.toml
├── src/
│   ├── main.rs                  # Entry point: starts the server
│   ├── settings.rs              # Project settings (DATABASES, INSTALLED_APPS, etc.)
│   ├── urls.rs                  # Root URL configuration (UnifiedRouter)
│   ├── apps.rs                  # App registration (mod declarations)
│   └── apps/
│       ├── user.rs              # User app entry point
│       ├── user/
│       │   ├── models.rs        # Model definitions
│       │   ├── views.rs         # View functions / ViewSets
│       │   ├── serializers.rs   # Serializer definitions
│       │   ├── urls.rs          # App-level URL routing (ServerRouter)
│       │   └── tests.rs         # App-level tests
│       ├── post.rs              # Post app entry point
│       └── post/
│           ├── models.rs
│           ├── views.rs
│           ├── serializers.rs
│           ├── urls.rs
│           └── tests.rs
└── tests/
    └── integration/
        └── api_tests.rs         # Cross-app integration tests
```

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

## App Registration Pattern

Each app must be declared in `src/apps.rs` and registered in `src/settings.rs`:

```rust
// src/apps.rs — App module declarations
pub mod user;
pub mod post;
```

```rust
// src/settings.rs — App registration
use crate::apps;

pub fn installed_apps() -> Vec<AppConfig> {
    vec![
        AppConfig::new::<apps::user::UserApp>(),
        AppConfig::new::<apps::post::PostApp>(),
    ]
}
```

```rust
// src/apps/user.rs — App entry point with re-exports
pub mod models;
pub mod views;
pub mod serializers;
pub mod urls;

#[cfg(test)]
mod tests;

// Re-export the app's public API
pub use models::User;
pub use views::UserViewSet;

/// App configuration
pub struct UserApp;

impl AppConfig for UserApp {
    fn name(&self) -> &str {
        "user"
    }

    fn label(&self) -> &str {
        "user"
    }
}
```

## Adding a New App

Follow this procedure to add a new app to an existing project:

1. **Generate the app scaffold**:
   ```bash
   reinhardt-admin startapp <name>
   ```

2. **Verify the generated structure** matches the layout above. Convert any `mod.rs` files to the `module.rs` + `module/` pattern if needed.

3. **Create the app entry point** (`src/apps/<name>.rs`):
   - Declare submodules: `pub mod models;`, `pub mod views;`, etc.
   - Define `struct <Name>App` implementing `AppConfig`
   - Add `pub use` re-exports for the app's public API

4. **Register the app module** in `src/apps.rs`:
   ```rust
   pub mod <name>;
   ```

5. **Register the app config** in `src/settings.rs`:
   ```rust
   AppConfig::new::<apps::<name>::<Name>App>()
   ```

6. **Add app routes** to the root URL configuration in `src/urls.rs`:
   ```rust
   router.include("/api/<name>/", apps::<name>::urls::router());
   ```

7. **Verify compilation**:
   ```bash
   cargo check
   ```
