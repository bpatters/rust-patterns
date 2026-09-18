# Async/Await Essentials

Load this when: writing async functions; choosing a runtime; writing `async fn` in traits; accepting async closures (`AsyncFn`); fixing `Send` bound errors; dealing with blocking work in async code.

## Core Model

Rust's async is **fundamentally different** from Go's goroutines or Python's asyncio. Three concepts cover 90%:

1. **A `Future` is a lazy state machine** — calling `async fn` doesn't execute anything; it returns a `Future` that must be polled
2. **You need a runtime** to poll futures. **Tokio is the default** (axum, reqwest, sqlx, tonic). `async-std` is discontinued (maintainers recommend `smol` or Tokio). The stdlib defines `Future` but provides no runtime.
3. **`async fn` is sugar** — the compiler transforms it into a state machine implementing `Future`

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

// async fn desugars to: fn fetch_data(url: &str) -> impl Future<Output = Result<Vec<u8>, Error>>
async fn fetch_data(url: &str) -> Result<Vec<u8>, reqwest::Error> {
    let response = reqwest::get(url).await?;
    let bytes = response.bytes().await?;
    Ok(bytes.to_vec())
}
```

## Tokio Quick Start

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }  # binaries. Libraries: enable only rt, macros, net, time, sync as needed.
```

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let h1 = tokio::task::spawn(async {
        sleep(Duration::from_millis(100)).await;
        "task A done"
    });
    let h2 = tokio::task::spawn(async {
        sleep(Duration::from_millis(50)).await;
        "task B done"
    });

    // spawn already runs them concurrently; join! just waits for both
    let (a, b) = tokio::join!(h1, h2);
    println!("{}, {}", a.unwrap(), b.unwrap());
}
```

## Common Pitfalls

| Pitfall | Why it happens | Fix |
|---|---|---|
| **Blocking in async** | `std::thread::sleep` or CPU work blocks the executor | `tokio::task::spawn_blocking` or `rayon` |
| **`Send` bound errors** | Future held across `.await` contains `!Send` (e.g., `Rc`, `MutexGuard`) | Nested `{ ... }` block so the value drops before `.await`. `drop(guard)` is not enough — Send analysis is scope-based |
| **Future not polled** | Calling `async fn` without `.await` or spawning — nothing happens | Always `.await` or `tokio::spawn` the future |
| **`MutexGuard` across `.await`** | `std::sync::MutexGuard` is `!Send`. On `tokio::spawn` this is a **compile error**, not a panic | Nested block so the guard drops before `.await`. Short critical section, no `.await` while held → `std::sync::Mutex`. Must hold across `.await` → `tokio::sync::Mutex` |
| **Accidental sequential** | `let a = foo().await; let b = bar().await;` on **unspawned** futures runs sequentially | `tokio::join!` or `tokio::spawn`. Spawned tasks already run concurrently — awaiting their handles sequentially still waits concurrently |

```rust
// ❌ Blocking the async executor
async fn bad() {
    std::thread::sleep(std::time::Duration::from_secs(5));  // blocks entire thread
}

// ✅ Offload blocking work
async fn good() {
    tokio::task::spawn_blocking(|| {
        std::thread::sleep(std::time::Duration::from_secs(5));  // runs on blocking pool
    }).await.unwrap();
}
```

## Spawning vs Joining

```rust
use tokio::task;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let h1 = task::spawn(async { sleep(Duration::from_millis(200)).await; "u" });
    let h2 = task::spawn(async { sleep(Duration::from_millis(100)).await; "o" });
    let h3 = task::spawn(async { sleep(Duration::from_millis(150)).await; "r" });

    // spawn already runs them; join! waits. Sequential h1.await; h2.await is
    // still concurrent. Dropping a JoinHandle detaches — it does not abort.
    let (r1, r2, r3) = tokio::join!(h1, h2, h3);
}
```

### `join!` vs `try_join!` vs `select!`

| Macro | Behavior | Use when |
|---|---|---|
| `join!` | Polls all futures concurrently on **this** task; waits for all | Unspawned futures that should run together |
| `try_join!` | Like `join!`, short-circuits on first `Err` | Futures that return `Result`. On `JoinHandle<Result<T, E>>` the inner `Err` is **not** a `try_join!` failure — you get `Result<Result<T, E>, JoinError>` |
| `select!` | Returns when FIRST future completes; **drops the losers** (cancellation) | Timeouts, racing. This is `tokio::select!`, not `crossbeam_channel::select!` |

```rust
use tokio::time::{timeout, Duration};

async fn fetch_with_timeout() -> Result<String, Box<dyn std::error::Error>> {
    let result = timeout(Duration::from_secs(5), async {
        Ok::<_, Box<dyn std::error::Error>>("data".to_string())
    }).await??;  // first ? unwraps Elapsed, second ? unwraps inner Result
    Ok(result)
}
```

## `async fn` in Traits (stable since 1.75)

Write `async fn` directly in traits. No `async-trait` crate for static dispatch.

```rust
trait Store {
    async fn get(&self, key: &str) -> Option<String>;
}
```

**Not dyn-compatible** — the returned future is an opaque type. For `dyn Store`, erase it:

```rust
fn get(&self, key: &str) -> Pin<Box<dyn Future<Output = Option<String>> + Send + '_>>;
```

**`Send` on public traits**: bare `async fn` does not promise `Send`. Multi-thread `tokio::spawn` needs it. Either desugar and add the bound, or generate both variants:

```rust
fn get(&self, key: &str) -> impl Future<Output = Option<String>> + Send;

// or: cargo add trait-variant
#[trait_variant::make(Store: Send)]
trait LocalStore {
    async fn get(&self, key: &str) -> Option<String>;
}
```

`async-trait` is only for `dyn` dispatch or old MSRV. New code: native `async fn` + `trait-variant` when callers spawn.

## Async Closures (stable since 1.85)

`async || { ... }` returns a new `Future` each call and can borrow captures (unlike `|| async { ... }`). Bounds: `AsyncFn` / `AsyncFnMut` / `AsyncFnOnce` (prelude).

```rust
async fn retry<F>(mut f: F) -> Result<String, Error>
where
    F: AsyncFnMut() -> Result<String, Error>,
{
    f().await
}

let token = fetch_token().await;
retry(async || fetch_with(&token).await).await?;
```

Prefer `async ||` over `|| async move { ... }` for callbacks and middleware.

## `Send` Bounds and Why Futures Must Be `Send`

When you `tokio::spawn`, the future may resume on a different OS thread. The future must be `Send + 'static` (no borrows of locals). `spawn_local` drops the `Send` requirement but still needs `'static`.

```rust
use std::rc::Rc;

async fn not_send() {
    let rc = Rc::new(42);  // Rc is !Send
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{rc}");  // rc held across .await — future is !Send
}

// Fix 1: Drop before .await
async fn fixed_drop() {
    let data = { let rc = Rc::new(42); *rc };  // copy i32 out, rc dropped
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{data}");  // ✅
}

// Fix 2: Use Arc instead of Rc
async fn fixed_arc() {
    let arc = std::sync::Arc::new(42);  // Arc is Send
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{arc}");  // ✅
}
```

## Choosing the Right Sync Primitive in Async

| Need | Use | Why |
|---|---|---|
| Short critical section, **no** `.await` while held | `std::sync::Mutex` | Tokio's default recommendation; cheaper; `!Send` guard is a compile-time safety net |
| Must hold the lock across `.await` | `tokio::sync::Mutex` | Guard is `Send`; lock is async-aware |
| Async-aware read/write | `tokio::sync::RwLock` | Same split as above vs `std::sync::RwLock` |
| Async channels | `tokio::sync::mpsc` / `broadcast` / `watch` | Native async support |
| One-time async init | `tokio::sync::OnceCell` | First `.await` initializes. Sync equivalent: `std::sync::OnceLock` |
| CPU-heavy / blocking work | `tokio::task::spawn_blocking` | Dedicated blocking pool; **cannot be aborted** once running |

## Cancellation Safety

A future is **cancel-safe** if dropping it (the way `select!` and `timeout` cancel losers) loses no unrecoverable state.

- `select!` / `timeout` **drop the losing branches**. That is why this matters.
- Safe in `select!`: `recv()`, `read()`, awaiting `&mut JoinHandle`.
- **Not** safe: `read_exact`, `read_line`, `write_all` — partial I/O is lost.
- `abort()` runs `Drop` at the next `.await`; it does not stop `spawn_blocking`.
- Dropping a `JoinHandle` **detaches** (task keeps running). Call `abort()` if you need cancellation.
- Do not use `StreamExt::buffered` / `buffer_unordered` to "drain" — dropping the stream drops in-flight work.

## Async and FFI / Blocking Code

```rust
// ❌ Calling a blocking C function in async context
async fn bad() {
    unsafe { some_blocking_c_function(); }  // blocks executor
}

// ✅ Offload to blocking pool
async fn good() {
    tokio::task::spawn_blocking(|| unsafe { some_blocking_c_function() })
        .await
        .unwrap();
}
```

## Anti-Patterns

- **Sequential `await`s when concurrent is needed.** `let a = foo().await; let b = bar().await;` — use `tokio::join!`.
- **Holding `std::sync::MutexGuard` across `.await`.** Compile error on `tokio::spawn` (`!Send`). Nested `{ ... }` block, not `drop(guard)`. Don't reach for `tokio::sync::Mutex` unless you must hold the lock across `.await`.
- **Forgetting to `.await` or `spawn` a returned `Future`.** The future does nothing until polled.
- **Using `tokio::sync::Mutex` for short critical sections that don't span `.await`.** `std::sync::Mutex` is the right default there. `tokio::sync::Mutex` is for holding across `.await`.
- **Spawning for everything.** Some operations are cheap and don't need a separate task — sequential `await`s can be fine.
- **Blocking I/O in async fn.** Use `tokio::fs`, `tokio::net`, or `spawn_blocking` for sync I/O.
- **`tokio::spawn` of a closure that captures `!Send` types.** Compile error — restructure to drop before spawn, or run sequentially.
- **New `async-std` projects.** The crate is discontinued. Use Tokio, or `smol` if you need a minimal composable runtime.
- **`#[async_trait]` on a new trait that is only used with static dispatch.** Native `async fn` in traits is zero-cost. Keep `async-trait` only for `dyn` or old MSRV.

## See Also

- [concurrency.md](./concurrency.md) — OS threads, Mutex/RwLock, atomics, rayon
- [channels.md](./channels.md) — sync channel patterns; tokio has its own channels for async
- [smart-pointers.md](./smart-pointers.md) — `Pin` for futures; `Send`/`Sync` auto-traits
- [error-handling.md](./error-handling.md) — error propagation across `.await`, `JoinError`