# Reactive System & Hooks Reference

## Core Primitives

### Signal

A reactive value with automatic dependency tracking.

```rust
use reinhardt::pages::prelude::*;

let count = Signal::new(0);

// Read value (registers dependency)
let value = count.get();

// Set value (notifies dependents)
count.set(5);

// Update in place
count.update(|n| *n += 1);
```

### Effect

A side effect that reruns when dependencies change.

```rust
let count = Signal::new(0);
let name = Signal::new("Alice".to_string());

Effect::new(move || {
    println!("{}: count = {}", name.get(), count.get());
});
// Prints: "Alice: count = 0"

count.set(5);
// Prints: "Alice: count = 5"
```

### Memo

A cached derived computation.

```rust
let count = Signal::new(0);
let doubled = Memo::new(move || count.get() * 2);

assert_eq!(doubled.get(), 0);
count.set(5);
assert_eq!(doubled.get(), 10);
```

## Context System

Share data through the component tree without prop drilling.

```rust
use reinhardt::pages::prelude::*;

// Provide context
let theme = Signal::new("dark".to_string());
provide_context("theme", theme.clone());

// Consume context (anywhere in the subtree)
let theme: Signal<String> = get_context("theme").unwrap();
```

| Function | Description |
|----------|-------------|
| `create_context(key, value)` | Create a new context |
| `provide_context(key, value)` | Provide a context value |
| `get_context::<T>(key)` | Get a context value |
| `remove_context(key)` | Remove a context |

## Hooks API

### State Hooks

| Hook | Signature | Description |
|------|-----------|-------------|
| `use_state` | `use_state(initial: T) -> (Signal<T>, SetState<T>)` | Local reactive state (takes value, not closure) |
| `use_reducer` | `use_reducer(reducer, init) -> (Signal<S>, Dispatch<A>)` | State with reducer pattern |
| `use_shared_state` | `use_shared_state(initial: T) -> (SharedSignal<T>, SharedSetState<T>)` | Shared state across components |
| `use_optimistic` | `use_optimistic(initial: T) -> OptimisticState<T>` | Optimistic UI updates |

```rust
// use_state takes a value directly (NOT a closure)
let (count, set_count) = use_state(0);
set_count(5);
```

### Effect Hooks

| Hook | Signature | Description |
|------|-----------|-------------|
| `use_effect` | `use_effect(closure)` | Side effect (async-safe) |
| `use_layout_effect` | `use_layout_effect(closure)` | Synchronous effect before paint |
| `use_effect_event` | `use_effect_event(closure) -> Callback<EventArg, ()>` | Event handler that reads latest values |
| `use_effect_event_with` | `use_effect_event_with(closure) -> Callback<Args, Ret>` | Generic event handler variant |

```rust
use_effect(move || {
    // Runs when dependencies change
    log!("Count is: {}", count.get());
});
```

**When to use `use_layout_effect`**: DOM measurements, preventing visual flicker.
**When to use `use_effect`** (preferred): Data fetching, subscriptions, logging.

### Derived Value Hooks

| Hook | Signature | Description |
|------|-----------|-------------|
| `use_memo` | `use_memo(closure) -> Memo<T>` | Cached computation |
| `use_callback` | `use_callback(closure) -> Callback<EventArg, ()>` | Stable event callback |
| `use_callback_with` | `use_callback_with(closure) -> Callback<Args, Ret>` | Generic stable callback |
| `use_deferred_value` | `use_deferred_value(signal) -> Signal<T>` | Deferred update for low-priority UI |

### Ref and Identity Hooks

| Hook | Signature | Description |
|------|-----------|-------------|
| `use_ref` | `use_ref(init) -> Ref<T>` | Mutable reference (no re-render on change) |
| `use_id` | `use_id() -> String` | Unique ID for accessibility |
| `use_id_with_prefix` | `use_id_with_prefix(prefix) -> String` | Unique ID with custom prefix |

### Async Hooks

| Hook | Signature | Description |
|------|-----------|-------------|
| `use_transition` | `use_transition() -> TransitionState` | Non-blocking state updates |
| `use_action` | `use_action(action_fn) -> Action<T, E>` | Async action with loading/error state |
| `use_action_state` | *(deprecated)* | Use `use_action` instead |

```rust
let save_action = use_action(|data: FormData| async move {
    save_to_server(data).await
});

// Trigger
save_action.dispatch(form_data);

// Check state
match save_action.phase().get() {
    ActionPhase::Idle => { /* ready */ },
    ActionPhase::Pending => { /* loading */ },
    ActionPhase::Resolved(result) => { /* done */ },
}
```

### External Integration Hooks

| Hook | Signature | Description |
|------|-----------|-------------|
| `use_sync_external_store` | `use_sync_external_store(subscribe, get_snapshot)` | Integrate external stores |
| `use_sync_external_store_with_server` | `...(subscribe, get_snapshot, get_server_snapshot)` | SSR-compatible variant |
| `use_websocket` | `use_websocket(url, options) -> WebSocketHandle` | Reactive WebSocket |
| `use_context` | `use_context(ctx: &Context<T>) -> Option<T>` | Read context value (takes `&Context<T>`, returns `Option`) |

### Debug Hooks

| Hook | Description |
|------|-------------|
| `use_debug_value` | Custom label in dev tools (requires `debug-hooks` feature) |

## Resource (WASM Only)

Async data loading with reactive dependencies.

```rust
#[cfg(wasm)]
{
    let user_id = Signal::new(1);
    let user = create_resource(move || async move {
        get_user(user_id.get()).await
    });

    // With dependencies
    let user = create_resource_with_deps(
        move || user_id.get(),
        |id| async move { get_user(id).await },
    );

    // Check state
    match user.state().get() {
        ResourceState::Loading => { /* show spinner */ },
        ResourceState::Ready(data) => { /* render data */ },
        ResourceState::Error(err) => { /* show error */ },
    }
}
```

## Platform Event Type

Platform-agnostic event type for cross-target code:

```rust
use reinhardt::pages::prelude::*;

fn handle_click(_event: Event) {
    // Works on both WASM and native
}
```

## Architecture Notes

- **Fine-grained reactivity**: Only DOM nodes depending on changed Signals update (not entire component trees)
- **Pull-based model**: Signals track dependencies automatically via `.get()` calls
- **Batching**: Multiple Signal changes batch into a single update cycle via micro-tasks
- **Memory management**: All reactive nodes auto-cleanup when dropped
