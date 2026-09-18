# Channels and Message Passing

Load this when: designing message-passing between threads/tasks; building a worker pool; needing backpressure; implementing an actor; choosing between mpsc vs crossbeam.

## std::sync::mpsc — The Standard Channel

Multi-producer, single-consumer:

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel();
let tx1 = tx.clone();
thread::spawn(move || {
    for i in 0..5 { tx1.send(format!("p1: {i}")).unwrap(); }
});
thread::spawn(move || {
    for i in 0..5 { tx.send(format!("p2: {i}")).unwrap(); }
});
for msg in rx { println!("Received: {msg}"); }  // ends when ALL senders dropped
```

**Properties**:
- **Unbounded** by default (can fill memory if consumer is slow)
- `mpsc::sync_channel(N)` creates a **bounded** channel with backpressure
- `rx.recv()` blocks until a message arrives
- `rx.try_recv()` returns immediately with `Err(Empty)` if nothing ready
- Channel closes when all `Sender`s are dropped

## crossbeam-channel — MPMC, `select!`, timeouts

`std::sync::mpsc` is crossbeam-backed, so "faster" is no longer the reason to switch. Remaining reasons: **multi-consumer (`Receiver: Clone`)**, `select!`, `tick`/`after`. `std::sync::mpmc` exists but is still nightly (`mpmc_channel`).

```rust
use crossbeam_channel::{bounded, unbounded, select};
// Cargo.toml: crossbeam-channel = "0.5"

let (tx, rx) = bounded::<String>(100);  // MPMC, bounded

for id in 0..4 {
    let tx = tx.clone();
    thread::spawn(move || {
        for i in 0..10 { tx.send(format!("w{id}: {i}")).unwrap(); }
    });
}
drop(tx);  // drop original sender so channel can close

// Multiple consumers — not possible with std::sync::mpsc
let rx2 = rx.clone();
let h1 = thread::spawn(move || while let Ok(msg) = rx.recv() { /* ... */ });
let h2 = thread::spawn(move || while let Ok(msg) = rx2.recv() { /* ... */ });
h1.join().unwrap(); h2.join().unwrap();
```

## select! — Listen on Multiple Channels

```rust
use crossbeam_channel::{bounded, tick, after, select};
use std::time::Duration;

let (work_tx, work_rx) = bounded::<String>(10);
let ticker = tick(Duration::from_secs(1));     // periodic tick
let deadline = after(Duration::from_secs(10)); // one-shot timeout

loop {
    select! {
        recv(work_rx) -> msg => match msg {
            Ok(job) => println!("Processing: {job}"),
            Err(_) => break,  // channel closed
        },
        recv(ticker) -> _ => println!("tick"),
        recv(deadline) -> _ => { println!("deadline"); break; }
    }
}
```

This is `crossbeam_channel::select!`, not `tokio::select!` (which cancels losers by drop). Crossbeam's `select!` randomizes order to prevent starvation (like Go).

## Bounded vs Unbounded and Backpressure

| Type | Behavior when full | Memory | Use case |
|---|---|---|---|
| **Unbounded** | Never blocks (grows heap) | Unbounded ⚠️ | Rare — only when producer is slower than consumer |
| **Bounded** | `send()` blocks until space | Fixed | **Production default** — prevents OOM |
| **Rendezvous** (`bounded(0)`) | `send()` blocks until `recv()` called | None | Synchronization / handoff |

**Rule**: Default to bounded channels in production unless you can prove the producer will never outpace the consumer. One-shot **reply** channels (the actor `Get(Sender<i64>)` pattern below) are the usual exception — they carry one value and then close.

## Actor Pattern

Use channels to serialize access to mutable state — no mutexes needed.

```rust
enum CounterMsg {
    Increment,
    Decrement,
    Get(mpsc::Sender<i64>),  // reply channel
}

struct CounterActor { count: i64, rx: mpsc::Receiver<CounterMsg> }

impl CounterActor {
    fn run(mut self) {
        while let Ok(msg) = self.rx.recv() {
            match msg {
                CounterMsg::Increment => self.count += 1,
                CounterMsg::Decrement => self.count -= 1,
                CounterMsg::Get(reply) => { let _ = reply.send(self.count); }
            }
        }
    }
}

#[derive(Clone)]
struct Counter { tx: mpsc::Sender<CounterMsg> }

impl Counter {
    fn spawn() -> Self {
        let (tx, rx) = mpsc::channel();
        thread::spawn(move || CounterActor { count: 0, rx }.run());
        Counter { tx }
    }
    fn increment(&self) { let _ = self.tx.send(CounterMsg::Increment); }
    fn get(&self) -> i64 {
        let (reply_tx, reply_rx) = mpsc::channel();
        self.tx.send(CounterMsg::Get(reply_tx)).unwrap();
        reply_rx.recv().unwrap()
    }
}
```

**Actors vs Mutexes**:
- Actors: complex invariants, long operations, no lock ordering to think about
- Mutexes: simple short critical sections

## Anti-Patterns

- **Unbounded channels in production.** Producer can outpace consumer → OOM.
- **Using `std::sync::mpsc` when you need multi-consumer.** Use `crossbeam-channel`.
- **Channels for everything.** Simple state + short critical section → `Mutex` is clearer.
- **Polling with `try_recv()` in a loop.** Wasteful. Use blocking `recv()` or `select!` with a timeout channel.

## See Also

- [concurrency.md](./concurrency.md) — Mutex/RwLock/atomics as alternatives
- [async.md](./async.md) — `tokio::sync::mpsc` for async contexts
- [smart-pointers.md](./smart-pointers.md) — `Arc` for sharing sender/receiver across threads