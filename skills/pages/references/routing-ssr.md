# Routing, SSR & Hydration Reference

## Client-Side Router

Django-style URL patterns with History API integration.

### Setup

```rust
use reinhardt::pages::prelude::*;
use std::sync::Arc;

let router = Arc::new(Router::new()
    .route("/", home_page)
    .route("/users/", user_list)
    .route("/users/{id}/", user_detail)
    .named_route("user_detail", "/users/{id}/", user_detail));
```

### Components

| Component | Description |
|-----------|-------------|
| `Router` | Route registry with pattern matching |
| `Route` | Single route definition |
| `Link` | Client-side navigation (no full page reload) |
| `RouterOutlet` | Renders the current matched route |
| `Redirect` | Programmatic redirect component |

### RouterOutlet

```rust
let outlet = RouterOutlet::new(router.clone());
// Renders the component matching the current URL
```

### Link Component

```rust
// In page! macro
Link(to: "/users/42/") { "View User" }

// With named route
Link(to: router.reverse("user_detail", &[("id", "42")])) { "View User" }
```

### Programmatic Navigation

```rust
router.push("/users/42/");
```

### Route Parameters

```rust
use reinhardt::pages::router::{PathParams, FromPath};

fn user_detail(params: PathParams) -> Page {
    let id: i64 = params.get("id").unwrap();
    // ...
}
```

### Route Guards

```rust
use reinhardt::pages::router::{guard, guard_or};

// Redirect to login if not authenticated
let protected_route = guard(is_authenticated, "/login");
```

### PathPattern

Django-style URL patterns with compile-time validation:

```rust
use reinhardt::pages::prelude::PathPattern;

let pattern = PathPattern::new("/users/{id}/posts/{post_id}/");
let matched = pattern.match_path("/users/42/posts/7/");
// matched.get("id") == Some("42")
// matched.get("post_id") == Some("7")
```

## Server-Side Rendering (SSR)

### SsrRenderer

```rust
use reinhardt::pages::prelude::*;

// Simple rendering
let html = SsrRenderer::render(&my_component);

// With options
let html = SsrRenderer::with_options(SsrOptions::default())
    .render(&my_component);

// Full page rendering (includes head, layout)
let page = SsrRenderer::render_page(
    &my_component,
    "My Page Title",
    Some("<!DOCTYPE html>..."),
);
```

### Head Section in SSR

```rust
let page_head = head!(|| {
    title { "My App" }
    meta { name: "description", content: "..." }
});

let page = page! {
    #head: page_head,
    || { div { "Content" } }
}();

// SsrRenderer includes the head in the HTML output
let html = SsrRenderer::render(&page);
```

## Hydration

Client-side activation of server-rendered HTML.

### Setup

```rust
use reinhardt::pages::prelude::*;

// Initialize hydration state (call once on WASM startup)
init_hydration_state();

// Hydrate a component
hydrate(&my_component);

// Check completion
if is_hydration_complete() {
    // All components hydrated
}

// Callback on completion
on_hydration_complete(|| {
    log!("Hydration complete");
});
```

### HydrationContext

```rust
use reinhardt::pages::prelude::HydrationContext;

// Access hydration context for state restoration
let ctx = HydrationContext::current();
```

### Island Hydration

Selective hydration of interactive sections within static HTML. Only marked "islands" are hydrated on the client, reducing JavaScript and improving performance.

## Static File Resolution

```rust
use reinhardt::pages::prelude::*;

// Initialize (once, on startup)
init_static_resolver(static_manifest);

// Resolve hashed URL
let css_url = resolve_static("css/main.css");
// Returns: "/static/css/main.abc123.css"

// Check if initialized
if is_initialized() {
    // Safe to resolve
}
```

Compatible with reinhardt's collectstatic system for cache-busted asset URLs.

## cfg_aliases Setup

Required in `build.rs` for platform-conditional code:

```rust
// build.rs
use cfg_aliases::cfg_aliases;

fn main() {
    println!("cargo::rustc-check-cfg=cfg(wasm)");
    println!("cargo::rustc-check-cfg=cfg(native)");

    cfg_aliases! {
        wasm: { target_arch = "wasm32" },
        native: { not(target_arch = "wasm32") },
    }
}
```

```toml
# Cargo.toml
[build-dependencies]
cfg_aliases = "0.2"
```

Then use `#[cfg(wasm)]` and `#[cfg(native)]` instead of `#[cfg(target_arch = "wasm32")]`.
