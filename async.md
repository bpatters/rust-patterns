# Async/Await Essentials

Load this when: writing async functions; choosing between tokio/async-std/smol; fixing `Send` bound errors; dealing with blocking work in async code; concurrent task execution patterns.

## Core Model

Rust's async is **fundamentally different** from Go's goroutines or Python's asyncio. Three concepts cover 90%:

1. **A `Future` is a lazy state machine** — calling `async fn` doesn't execute anything; it returns a `Future` that must be polled
2. **You need a runtime** to poll futures — `tokio`, `async-std`, `smol`. The stdlib defines `Future` but provides no runtime
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
tokio = { version = "1", features = ["full"] }
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

    let (a, b) = tokio::join!(h1, h2);  // concurrent, NOT sequential
    println!("{}, {}", a.unwrap(), b.unwrap());
}
```

## Common Pitfalls

| Pitfall | Why it happens | Fix |
|---|---|---|
| **Blocking in async** | `std::thread::sleep` or CPU work blocks the executor | `tokio::task::spawn_blocking` or `rayon` |
| **`Send` bound errors** | Future held across `.await` contains `!Send` (e.g., `Rc`, `MutexGuard`) | Restructure to drop before `.await` |
| **Future not polled** | Calling `async fn` without `.await` or spawning — nothing happens | Always `.await` or `tokio::spawn` the future |
| **`MutexGuard` across `.await`** | `std::sync::MutexGuard` is `!Send`; async tasks may resume on a different thread | Use `tokio::sync::Mutex` or drop guard before `.await` |
| **Accidental sequential** | `let a = foo().await; let b = bar().await;` runs sequentially | `tokio::join!` or `tokio::spawn` for concurrency |

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

## Spawning and Structured Concurrency

```rust
use tokio::task;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let h1 = task::spawn(async { sleep(Duration::from_millis(200)).await; "u" });
    let h2 = task::spawn(async { sleep(Duration::from_millis(100)).await; "o" });
    let h3 = task::spawn(async { sleep(Duration::from_millis(150)).await; "r" });

    // Wait for all three concurrently (not sequentially!)
    let (r1, r2, r3) = tokio::join!(h1, h2, h3);
}
```

### `join!` vs `try_join!` vs `select!`

| Macro | Behavior | Use when |
|---|---|---|
| `join!` | Waits for ALL futures | All tasks must complete |
| `try_join!` | Waits for all, short-circuits on first `Err` | Tasks return `Result` |
| `select!` | Returns when FIRST future completes | Timeouts, cancellation |

```rust
use tokio::time::{timeout, Duration};

async fn fetch_with_timeout() -> Result<String, Box<dyn std::error::Error>> {
    let result = timeout(Duration::from_secs(5), async {
        Ok::<_, Box<dyn std::error::Error>>("data".to_string())
    }).await??;  // first ? unwraps Elapsed, second ? unwraps inner Result
    Ok(result)
}
```

## `Send` Bounds and Why Futures Must Be `Send`

When you `tokio::spawn`, the future may resume on a different OS thread. The future must be `Send`.

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
| Async-aware mutual exclusion | `tokio::sync::Mutex` | Doesn't block the executor |
| Async-aware read/write | `tokio::sync::RwLock` | Same |
| Async channels | `tokio::sync::mpsc` / `tokio::sync::broadcast` / `tokio::sync::watch` | Native async support |
| One-time async init | `tokio::sync::OnceCell` | First `.await` initializes |
| CPU-heavy work | `tokio::task::spawn_blocking` | Runs on dedicated blocking pool |

**Note**: in pure async code, prefer `tokio::sync::Mutex` over `std::sync::Mutex` even for short critical sections — the issue isn't blocking duration, it's that the executor may move the task to a different thread mid-lock.

## Cancellation Safety

A future is **cancellation-safe** if dropping it mid-execution leaves the program in a valid state. Common concerns:

- **I/O operations**: usually safe — the request is aborted, partial results dropped
- **Locks held across `.await`**: dropping the guard releases the lock; if the future is the only holder, other tasks resume
- **Background tasks (`tokio::spawn`)**: continue running independently — if you `abort` them, they get killed at the next `.await`
- **Streams**: cancellation can lose buffered items — use `Buffered` if you need to drain

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
- **Holding `std::sync::MutexGuard` across `.await`.** Causes sporadic task panics and deadlocks.
- **Forgetting to `.await` or `spawn` a returned `Future`.** The future does nothing until polled.
- **Using `tokio::sync::Mutex` outside async contexts.** It's designed for `.await`-based access — `std::sync::Mutex` is better in sync code.
- **Spawning for everything.** Some operations are cheap and don't need a separate task — sequential `await`s can be fine.
- **Blocking I/O in async fn.** Use `tokio::fs`, `tokio::net`, or `spawn_blocking` for sync I/O.
- **`tokio::spawn` of a closure that captures `!Send` types.** Compile error — restructure to drop before spawn, or run sequentially.

## See Also

- [concurrency.md](./concurrency.md) — OS threads, Mutex/RwLock, atomics, rayon
- [channels.md](./channels.md) — sync channel patterns; tokio has its own channels for async
- [smart-pointers.md](./smart-pointers.md) — `Pin` for futures; `Send`/`Sync` auto-traits
- [error-handling.md](./error-handling.md) — error propagation across `.await`, `JoinError`