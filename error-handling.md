# Error Handling Patterns

Load this when: designing error types; choosing between `thiserror` and `anyhow`; deciding panic vs `Result`; adding context to errors; using `catch_unwind`; handling `Send` errors across threads.

## thiserror vs anyhow

| | `thiserror` | `anyhow` |
|---|---|---|
| Use in | Libraries, shared crates | Applications, binaries |
| Error types | Concrete enums (callers can match) | `anyhow::Error` (opaque) |
| Effort | Define your enum | Just use `Result<T>` |
| Downcasting | Not needed — pattern match | `error.downcast_ref::<MyError>()` |
| `.context()` | Not available (that's an anyhow trait) | `with_context` / `context` on any `Result` |

### thiserror (Libraries)

`thiserror = "2"` is current (`cargo add thiserror`). The derive API below is unchanged from 1.x.

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DatabaseError {
    #[error("connection failed: {0}")]
    ConnectionFailed(String),

    #[error("query error: {source}")]
    QueryError {
        #[source]
        source: sqlx::Error,
    },

    #[error("record not found: table={table} id={id}")]
    NotFound { table: String, id: u64 },

    #[error(transparent)]  // delegate Display to inner
    Io(#[from] std::io::Error),  // auto-generates From<io::Error>
}
```

### anyhow (Applications)

```rust
use anyhow::{Context, Result, bail, ensure};

fn read_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {path}"))?;

    let config: Config = serde_json::from_str(&content)
        .context("failed to parse config JSON")?;

    ensure!(config.port > 0, "port must be positive, got {}", config.port);

    Ok(config)
}

fn main() -> Result<()> {
    let config = read_config("config.json")?;
    if config.name.is_empty() { bail!("server name cannot be empty"); }
    Ok(())
}
```

## Error Conversion Chains with `#[from]`

```rust
#[derive(Error, Debug)]
enum AppError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
    #[error("JSON error: {0}")]
    Json(#[from] serde_json::Error),
    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),
}

// Now ? automatically converts:
fn fetch_and_parse(url: &str) -> Result<Config, AppError> {
    let body = reqwest::blocking::get(url)?.text()?;  // reqwest::Error → Http
    let config: Config = serde_json::from_str(&body)?; // serde_json::Error → Json
    Ok(config)
}
```

## Context and Error Wrapping

Add human-readable context without losing the original:

```rust
fn process_file(path: &str) -> Result<Data> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read {path}"))?;
    let data = parse_content(&content)
        .with_context(|| format!("failed to parse {path}"))?;
    validate(&data).context("validation failed")?;
    Ok(data)
}

// Output:
// Error: validation failed
//
// Caused by:
//    0: failed to parse config.json
//    1: expected ',' at line 5 column 12
```

## The `?` Operator in Depth

`?` is `match` + `From` conversion + early return:

```rust
// This:
let value = operation()?;
// Desugars to:
let value = match operation() {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),
};
```

`?` also works with `Option` in functions returning `Option`:

```rust
fn find_user_email(users: &[User], name: &str) -> Option<String> {
    let user = users.iter().find(|u| u.name == name)?;
    let email = user.email.as_ref()?;
    Some(email.to_uppercase())
}
```

## Panic vs Result vs catch_unwind

| | Use for |
|---|---|
| `Result<T, E>` | **Expected failures** — file not found, network timeout, parse error |
| `panic!()` | **Bugs** — index out of bounds, invariant violated, "this can't happen" |
| `process::abort()` | Unrecoverable — security violation, corrupt data |
| `catch_unwind` | FFI boundaries, thread pools — isolate panic from caller. No-op if `panic = abort`. Closure must be `UnwindSafe` (often `AssertUnwindSafe`). Unwinding from Rust into C is UB — catch **inside** `extern "C"` exports |

```rust
fn get_element(data: &[i32], index: usize) -> &i32 {
    // If this panics, it's a programming error — fix the caller, don't "handle" it.
    &data[index]
}

use std::panic;
let result = panic::catch_unwind(|| risky_operation());
match result {
    Ok(v) => println!("Success: {v:?}"),
    Err(_) => eprintln!("Operation panicked — continuing safely"),
}
```

## Send + 'static Errors Across Threads

`thread::spawn` requires `F: Send + 'static`. `Box<dyn Error>` is **not** `Send`. Returning errors from threads needs `E: Send + 'static`:

```rust
// ❌ Doesn't compile — Box<dyn Error> is not Send
fn load() -> Result<String, Box<dyn std::error::Error>> {
    Ok(std::fs::read_to_string("/tmp/config")?)
}
std::thread::spawn(|| load()); // error: `dyn Error` cannot be sent between threads

// ✅ Thread-safe error type
fn load() -> Result<String, Box<dyn std::error::Error + Send + Sync>> { /* ... */ }
std::thread::spawn(|| load()); // OK
```

`anyhow::Error` is `Send + Sync + 'static` — works across threads. Do not put `anyhow::Error` or `Box<dyn Error>` in a **library public** API (callers cannot match).

## From Conversions for Crate Boundaries

Implement `From<X>` for your error type to enable `?` across boundaries:

```rust
impl From<std::io::Error> for MyError {
    fn from(e: std::io::Error) -> Self { MyError::Io(e) }
}
impl From<serde_json::Error> for MyError {
    fn from(e: serde_json::Error) -> Self { MyError::Parse(e) }
}
```

Or use `#[from]` in `thiserror` to auto-generate.

## Anti-Patterns

- **`.unwrap()` / `.expect()` in library code.** Return `Result`; let the caller decide. Use only in tests and prototypes.
- **`panic!` for expected errors.** File-not-found, network timeout, parse error → `Result`.
- **Stringly-typed errors.** `Result<T, String>` throws away type information — use `thiserror`.
- **Silently swallowing errors.** `let _ = fallible_op();` discards `Err`. (`let _ = fallible_op()?;` still propagates — it only discards `Ok`.) Log it, propagate with `?`, or handle.
- **Catching all errors with `Err(e) => eprintln!("{e}")`.** Sometimes correct, but be intentional — you may be hiding bugs.
- **Putting `anyhow::Error` / `Box<dyn Error>` in a library public API.** Callers cannot match; `Box<dyn Error>` is not `Send`. Wrap foreign errors in *your* type via `#[from]` or `.map_err()` when you do not want implicit `?` conversion. You **can** `impl From<Foreign> for Local`; you **cannot** `impl From<ForeignA> for ForeignB`.
- **Generic `Result<T, Box<dyn Error>>` in library APIs.** Callers can't match. Define a real error type.
- **Using `catch_unwind` as control flow.** It's for boundary safety, not error handling.
- **Huge error enums with one variant per failure point.** Group related failures; use `#[source]` for the underlying cause.

## See Also

- [api-design.md](./api-design.md) — designing public error types; `TryFrom` for parse-don't-validate
- [serialization.md](./serialization.md) — serde error handling
- [async.md](./async.md) — error propagation across `.await`, `JoinHandle` result unwrapping
- [testing.md](./testing.md) — testing error paths