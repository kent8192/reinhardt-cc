# Routing, SSR & Hydration Reference

## Client-Side Router

Django-style URL patterns with History API integration.

### Setup

Routes take closures that return `Page`:

```rust
use reinhardt::pages::router::Router;

fn init_router() -> Router {
    Router::new()
        .route("/", || dashboard_page())
        .route("/login", || login_page())
        .route("/users/", || user_list_page())
        .route("/users/{id}/", || user_detail_page())
        .named_route("user_detail", "/users/{id}/", || user_detail_page())
        .not_found(|| not_found_page())
}
```

### Thread-Local Router (Recommended Pattern)

Store the router in a `thread_local!` for SPA access:

```rust
use std::cell::RefCell;
use reinhardt::pages::router::Router;

thread_local! {
    static ROUTER: RefCell<Option<Router>> = const { RefCell::new(None) };
}

pub fn init_global_router() {
    ROUTER.with(|r| {
        *r.borrow_mut() = Some(init_router());
    });
}

pub fn with_router<F, R>(f: F) -> R
where
    F: FnOnce(&Router) -> R,
{
    ROUTER.with(|r| {
        f(r.borrow().as_ref().expect("Router not initialized"))
    })
}
```

### Components

| Component | Description | Import |
|-----------|-------------|--------|
| `Router` | Route registry with pattern matching | prelude |
| `Route` | Single route definition | prelude |
| `Link` | Client-side navigation (no full page reload) | prelude |
| `RouterOutlet` | Renders the current matched route | prelude |
| `Redirect` | Programmatic redirect component | `reinhardt::pages::router::Redirect` |

### RouterOutlet

```rust
let outlet = RouterOutlet::new(router.clone());
// Renders the component matching the current URL
```

### Link Component

```rust
// In page! macro
Link(to: "/users/42/") { "View User" }

// With named route (reverse returns Result)
Link(to: router.reverse("user_detail", &[("id", "42")]).unwrap()) { "View User" }
```

### Programmatic Navigation

```rust
router.push("/users/42/");
```

### Route Parameters

Use `PathParams<T>` as an extractor with destructuring:

```rust
use reinhardt::pages::router::PathParams;

// PathParams<T> is a generic wrapper — destructure in the function signature
fn user_detail(PathParams(id): PathParams<i64>) -> Page {
    // id is i64, extracted from the URL
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

Django-style URL patterns:

```rust
use reinhardt::pages::prelude::PathPattern;

let pattern = PathPattern::new("/users/{id}/posts/{post_id}/");

// matches() returns Option<(HashMap<String, String>, Vec<String>)>
if let Some((params, _)) = pattern.matches("/users/42/posts/7/") {
    // params.get("id") == Some(&"42".to_string())
    // params.get("post_id") == Some(&"7".to_string())
}
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

// Full page rendering (takes component only)
let mut renderer = SsrRenderer::with_options(SsrOptions::default());
let page_html = renderer.render_page(&my_component);
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
