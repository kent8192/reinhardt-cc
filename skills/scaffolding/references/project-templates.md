# Reinhardt Project Templates Reference

## Template Types

The `reinhardt-admin startproject` command supports two template types via flags:

### `--restful` (Default)

REST API backend project. This is the default when no flag is specified.

- Generates a project structured for JSON API development
- Includes `src/settings.rs` with REST-oriented defaults
- Pre-configures JSON renderer and parser
- No frontend asset pipeline

### `--with-pages`

Full-stack application with WASM + SSR support.

- Generates a project with both server and client-side code
- Includes Pages component infrastructure for WASM rendering
- Configures server-side rendering (SSR) with hydration
- Includes server function support for RPC-style client-server communication
- Adds `trunk` build integration for WASM compilation

## Template Variables

The scaffolding engine substitutes these variables in generated files:

| Variable | Description | Example |
|----------|-------------|---------|
| `project_name` | The project name as provided by the user | `my_blog` |
| `crate_name` | Sanitized crate name (same as `project_name`, validated) | `my_blog` |
| `camel_case_project_name` | PascalCase version for struct/type names | `MyBlog` |
| `secret_key` | Auto-generated cryptographic secret key for sessions/signing | `a3f8...` (64 hex chars) |
| `reinhardt_version` | Current reinhardt crate version used in `Cargo.toml` | `0.1.0-alpha` |
| `is_restful` | Boolean: true when `--restful` or no flag | `true` / `false` |
| `with_pages` | Boolean: true when `--with-pages` | `true` / `false` |

## Post-Scaffolding Checklist

After running `reinhardt-admin startproject <name>`, complete these steps:

1. **Review `Cargo.toml`** — Verify feature flags match your requirements. Add or remove features based on your database backend, auth method, and component needs. See `feature-flags.md` for details.

2. **Review `src/settings.rs`** — Confirm database connection settings, secret key configuration, installed apps list, and middleware stack. Update `DATABASES`, `ALLOWED_HOSTS`, and `DEBUG` settings as needed.

3. **Convert `mod.rs` files (if present)** — The Rust 2024 Edition module system uses `module.rs` + `module/` directory structure. If any generated template contains `mod.rs` files, convert them:
   - Rename `src/foo/mod.rs` to `src/foo.rs`
   - Keep `src/foo/` directory with its submodule files intact
   - Update any `#[path]` attributes if present

4. **Run `cargo check`** — Verify the project compiles with the selected feature flags:
   ```bash
   cargo check --all-features
   ```

5. **Run `cargo fmt`** — Ensure generated code follows standard Rust formatting:
   ```bash
   cargo fmt --all
   ```

6. **Initialize Git** (if not already):
   ```bash
   git init
   git add .
   git commit -m "chore: initialize reinhardt project"
   ```
