# Concurrency vs Parallelism vs Threads

Load this when: choosing between threads/rayon/async; shared mutable state across threads; lock-free patterns; lazy initialization; scoped threads.

## Terminology

| Concurrency | Parallelism |
|---|---|
| Managing multiple tasks that can make progress | Executing multiple tasks simultaneously |
| One core is enough | Requires multiple cores |
| One cook, multiple dishes | Multiple cooks, each on a dish |
| `async/await`, channels, `select!` | `rayon`, `thread::spawn`, `par_iter()` |

## std::thread — OS Threads

1:1 with OS threads. Each gets its own stack (~2-8 MB).

```rust
let handle = thread::spawn(|| {
    for i in 0..5 { println!("spawned: {i}"); }
    42  // return value
});
let result = handle.join().unwrap();
```

**Closure requirements**: `Send` + `'static` + `FnOnce`. Use `move ||` to take ownership:

```rust
let data = vec![1, 2, 3];
thread::spawn(move || println!("{data:?}"));  // moves data
```

## Scoped Threads

Threads that can **borrow** from the parent scope — no `Arc::clone()` needed:

```rust
let mut data = vec![1, 2, 3, 4, 5];

thread::scope(|s| {
    s.spawn(|| { let sum: i32 = data.iter().sum(); /* ... */ });
    s.spawn(|| { let max = data.iter().max().unwrap(); /* ... */ });
    // ❌ Can't mutably borrow while shared borrows exist:
    // s.spawn(|| data.push(6));
});
// ALL scoped threads joined here — guaranteed before scope returns.

data.push(6);  // safe to mutate now
```

Huge ergonomic win over manual `Arc` for short-lived parallel work.

## rayon — Data Parallelism

Parallel iterators that distribute work across a thread pool. One-method conversion.

```rust
use rayon::prelude::*;
// Cargo.toml: rayon = "1"

let data: Vec<u64> = (0..1_000_000).collect();
let sum: u64 = data.par_iter().map(|x| x * x).sum();
let mut numbers = vec![5, 2, 8, 1, 9, 3];
numbers.par_sort();  // parallel sort
```

| Use | When |
|---|---|
| `rayon::par_iter()` | Processing collections in parallel (map/filter/reduce) |
| `thread::spawn` | Long-running background tasks, I/O workers |
| `thread::scope` | Short-lived parallel tasks that borrow local data |
| `async` + `tokio` | I/O-bound concurrency (networking, file I/O) |

## Shared State: Arc, Mutex, RwLock, Atomics

### Arc<Mutex<T>> — Shared + Exclusive

```rust
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0u64));
for _ in 0..10 {
    let counter = Arc::clone(&counter);
    thread::spawn(move || {
        for _ in 0..1000 {
            let mut guard = counter.lock().unwrap();
            *guard += 1;
        }
    });
}
```

### Arc<RwLock<T>> — Many Readers OR One Writer

```rust
let config = Arc::new(RwLock::new(String::from("initial")));

// Many readers — don't block each other
for id in 0..5 {
    let config = Arc::clone(&config);
    thread::spawn(move || { let g = config.read().unwrap(); println!("R{id}: {g}"); });
}

// Writer — blocks and waits for all readers
{ let mut g = config.write().unwrap(); *g = "updated".into(); }
```

### Atomics — Lock-Free for Simple Values

```rust
use std::sync::atomic::{AtomicU64, Ordering};

let counter = Arc::new(AtomicU64::new(0));
for _ in 0..10 {
    let counter = Arc::clone(&counter);
    thread::spawn(move || {
        for _ in 0..1000 { counter.fetch_add(1, Ordering::Relaxed); }
    });
}
```

### Quick Comparison

| Primitive | Use case | Cost |
|---|---|---|
| `Mutex<T>` | Short critical sections | Lock + unlock |
| `RwLock<T>` | Read-heavy, rare writes | Reader-writer lock |
| `AtomicU64` etc. | Counters, flags | Hardware CAS, lock-free |
| Channels | Message passing | Queue ops, decouples prod/consumer |

## Condvar — Wait Without Busy-Looping

Always paired with a `Mutex`. Wait until another thread signals.

```rust
let pair = Arc::new((Mutex::new(false), Condvar::new()));
let pair2 = Arc::clone(&pair);

thread::spawn(move || {
    let (lock, cvar) = &*pair2;
    let mut ready = lock.lock().unwrap();
    while !*ready { ready = cvar.wait(ready).unwrap(); }  // spurious wakeups
    println!("Worker: proceeding");
});

let (lock, cvar) = &*pair;
let mut ready = lock.lock().unwrap();
*ready = true;
cvar.notify_one();
```

**Always re-check the condition in a `while` loop after `wait()` — spurious wakeups are allowed.**

## Lazy Initialization: OnceLock and LazyLock

```rust
use std::sync::{OnceLock, LazyLock};

// OnceLock — initialize on first use via get_or_init (init can depend on runtime args)
static CONFIG: OnceLock<HashMap<String, String>> = OnceLock::new();

fn get_config() -> &'static HashMap<String, String> {
    CONFIG.get_or_init(|| {
        let mut m = HashMap::new();
        m.insert("log_level".into(), "info".into());
        m
    })
}

// LazyLock — initialize on first access, closure at definition site
static REGEX: LazyLock<regex::Regex> = LazyLock::new(|| {
    regex::Regex::new(r"^[a-zA-Z0-9_]+$").unwrap()
});
```

| Type | Stabilized | Init timing | Use when |
|---|---|---|---|
| `OnceLock<T>` | 1.70 | Call-site (`get_or_init`) | Init depends on runtime args |
| `LazyLock<T>` | 1.80 | Definition-site | Init is self-contained |
| `lazy_static!` | — | Definition-site (macro) | Pre-1.80 codebases — **migrate away** |
| `const fn` + `static` | Always | Compile-time | Value is computable at compile time |

**Migration**: replace `lazy_static! { static X: T = expr; }` with `static X: LazyLock<T> = LazyLock::new(|| expr);`.

## Lock-Free Patterns

**Practical advice**: Lock-free code is hard. Use `Mutex`/`RwLock` unless profiling shows lock contention is your bottleneck. When you need lock-free, reach for proven crates (`crossbeam`, `arc-swap`, `dashmap`) — don't roll your own.

Common patterns:
- **Atomic flag/CAS spin** — for very short critical sections, contested rarely. Counters whose only read is after `join()` can use `Ordering::Relaxed`. A **flag that publishes other data** needs `Release`/`Acquire` (or `SeqCst` if you do not want to think)
- **Bounded MPMC queue** — `crossbeam::queue::ArrayQueue` (not SPSC)
- **Sequence lock (SeqLock)** — for single-writer / multi-reader of small values; prefer a crate, don't roll your own
- **RCU** — via `arc-swap` or `crossbeam-epoch`

## Decision Tree

```text
Need shared mutable state?
├── No  → Channels
└── Yes → How much contention?
     ├── Read-heavy    → RwLock
     ├── Short critical → Mutex
     ├── Simple counter  → Atomics
     └── Complex state   → Actor + channels

Need parallelism?
├── Collection processing → rayon::par_iter
├── Background task      → thread::spawn
└── Borrow local data    → thread::scope
```

## Anti-Patterns

- **`lazy_static!` in new code.** Use `OnceLock` / `LazyLock`.
- **Manual spin locks in production.** Use `std::sync::Mutex` or `parking_lot::Mutex`.
- **Sharing `Rc` across threads.** `Rc` is `!Send`. Use `Arc`.
- **Holding a `std::sync::MutexGuard` across `.await`.** Compile error on `tokio::spawn` (`!Send`). Nested `{ ... }` block so the guard drops before `.await` — `drop(guard)` is not enough. Short critical section with no `.await` while held → `std::sync::Mutex`. Must hold across `.await` → `tokio::sync::Mutex`.
- **Polling with `try_lock` in a loop.** Wasteful — use `Condvar` or channels.
- **`.unwrap()` on lock acquisition.** Decide whether to recover from poisoned locks or propagate the error.
- **Rolling your own lock-free structure.** Reach for `crossbeam`, `arc-swap`, `dashmap`.

## See Also

- [channels.md](./channels.md) — message passing alternative
- [async.md](./async.md) — async concurrency primitives
- [smart-pointers.md](./smart-pointers.md) — `Arc` for sharing, `Send`/`Sync` auto-traits