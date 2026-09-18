# Smart Pointers and Interior Mutability

Load this when: choosing between Box/Rc/Arc/Weak/Cell/RefCell/Cow/Pin; breaking reference cycles; implementing interior mutability; working with self-referential types; controlling drop order.

## Box, Rc, Arc — Heap and Sharing

```rust
// Box<T>: single owner, heap allocation
let boxed: Box<i32> = Box::new(42);
enum List<T> { Cons(T, Box<List<T>>), Nil }  // recursive types require Box
let writer: Box<dyn std::io::Write> = Box::new(std::io::stdout());

// Rc<T>: multiple owners, single-threaded
use std::rc::Rc;
let a = Rc::new(vec![1, 2, 3]);
let b = Rc::clone(&a);  // increments refcount, NOT deep clone
let c = Rc::clone(&a);

// Arc<T>: multiple owners, thread-safe
use std::sync::Arc;
let shared = Arc::new(String::from("data"));
let h = std::thread::spawn({
    let shared = Arc::clone(&shared);
    move || println!("{shared}")
});
```

## Weak References — Breaking Cycles

`Rc`/`Arc` cannot free cycles (A→B→A). `Weak<T>` is non-owning — doesn't increment strong count:

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,        // doesn't keep parent alive
    children: RefCell<Vec<Rc<Node>>>,
}

let parent = Rc::new(Node { /* ... */ });
let child = Rc::new(Node { /* ... */ });
child.parent.borrow_mut().replace(Rc::downgrade(&parent));

if let Some(p) = child.parent.borrow().upgrade() {
    println!("parent: {}", p.value);
}
// When `parent` dropped → strong_count → 0 → memory freed.
```

**Rule**: `Rc`/`Arc` for ownership edges; `Weak` for back-references and caches. For threads: `sync::Weak<T>`.

## Cell and RefCell — Interior Mutability

Mutate data behind a shared (`&`) reference. Runtime borrow checking.

```rust
use std::cell::{Cell, RefCell};

// Cell<T>: never panics. get() needs Copy; replace/take work for non-Copy
struct Counter { count: Cell<u32> }
impl Counter {
    fn increment(&self) {  // &self, not &mut self!
        self.count.set(self.count.get() + 1);
    }
}

// RefCell<T>: any type — panics on double-mutable-borrow
struct Cache { data: RefCell<Vec<String>> }
impl Cache {
    fn add(&self, item: String) {  // &self — looks immutable
        self.data.borrow_mut().push(item);  // runtime-checked &mut
    }
}
```

| | `Cell<T>` | `RefCell<T>` |
|---|---|---|
| Works with | Any `T`. `get()` needs `Copy`; `set`/`replace`/`swap` for any `T`; `take` needs `T: Default` | Any type |
| Panics | Never | On double-mutable-borrow |
| Thread-safe | ❌ | ❌ |

Neither is `Sync` — for multithreaded interior mutability, use `Mutex`/`RwLock`.

## Cow — Clone on Write

Holds borrowed OR owned value. Clones only when mutation needed.

```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.contains('\t') {
        Cow::Owned(input.replace('\t', "    "))
    } else {
        Cow::Borrowed(input)  // no allocation
    }
}

// Byte-oriented API: zero allocation on common path
fn pad_frame(frame: &[u8], min_len: usize) -> Cow<'_, [u8]> {
    if frame.len() >= min_len { Cow::Borrowed(frame) }
    else {
        let mut padded = frame.to_vec();
        padded.resize(min_len, 0x00);
        Cow::Owned(padded)
    }
}
```

Also useful for function parameters that MIGHT need ownership.

## Decision Table — Which Pointer

| Pointer | Owners | Thread-safe | Mutability | Use when |
|---|---|---|---|---|
| `Box<T>` | 1 | ✅ if T: Send | via `&mut` | Heap, trait objects, recursive types |
| `Rc<T>` | N | ❌ | None (wrap in Cell/RefCell) | Shared ownership, single thread |
| `Arc<T>` | N | ✅ | None (wrap in Mutex/RwLock) | Shared across threads |
| `Cell<T>` | — | ❌ | `.get()`/`.set()` (Copy); `.replace`/`.take` otherwise | Interior mutability; `get()` needs `Copy` |
| `RefCell<T>` | — | ❌ | `.borrow()`/`.borrow_mut()` | Interior mutability, single thread |
| `Cow<'_, T>` | 0 or 1 | ✅ if T: Send | clone-on-write | Avoid alloc when data usually unchanged |
| `Pin<Box<T>>` | 1 | depends | self-ref types, Futures | Prevents moving |

## Pin and Self-Referential Types

`Pin<P>` prevents a value from being moved in memory. Essential for self-referential structs and `Future`s.

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

struct SelfRef {
    data: String,
    ptr: *const String,
    _pin: PhantomPinned,  // opts out of Unpin
}

impl SelfRef {
    fn new(s: &str) -> Pin<Box<Self>> {
        let val = SelfRef { data: s.into(), ptr: std::ptr::null(), _pin: PhantomPinned };
        let mut boxed = Box::pin(val);
        let self_ptr: *const String = &boxed.data;
        unsafe {
            let mut_ref = Pin::as_mut(&mut boxed);
            Pin::get_unchecked_mut(mut_ref).ptr = self_ptr;
        }
        boxed
    }
    fn data(&self) -> &str { &self.data }
    fn ptr_data(&self) -> &str { unsafe { &*self.ptr } }
}
```

### Key Concepts

| Concept | Meaning |
|---|---|
| `Unpin` (auto-trait) | "Moving this type is safe." Most types are `Unpin` by default. |
| `!Unpin` / `PhantomPinned` | "I have internal pointers — don't move me." |
| `Pin<&mut T>` | Mutable reference guaranteeing `T` won't move |
| `Pin<Box<T>>` | Owned, heap-pinned value |

**Why async**: every `async fn` desugars to a `Future` that may hold references across `.await` points — making it self-referential. The runtime pins before polling.

**Crate alternatives**: prefer `self_cell`. Avoid `ouroboros` (open soundness issues; last release 2025-01).

### pin-project — Safe Pin Projections

```rust
use pin_project::pin_project;

#[pin_project]
struct TimedFuture<F: Future> {
    #[pin]                                // structurally pinned
    inner: F,
    started_at: std::time::Instant,        // NOT pinned
}

impl<F: Future> Future for TimedFuture<F> {
    type Output = (F::Output, std::time::Duration);
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();           // safe! generated by pin_project
        // this.inner : Pin<&mut F>
        // this.started_at : &mut Instant
        match this.inner.poll(cx) {
            Poll::Ready(o) => Poll::Ready((o, this.started_at.elapsed())),
            Poll::Pending => Poll::Pending,
        }
    }
}
```

Use `pin-project` (attribute, as above) or, in libraries, `pin-project-lite` (`pin_project! { ... }`, no syn) when wrapping a `Future` or `Stream`.

## Drop Ordering and ManuallyDrop

**Drop order rules**:

| What | Order | Rationale |
|---|---|---|
| Local variables | Reverse declaration order | Later vars might reference earlier ones |
| Struct fields | Declaration order (top to bottom) | Stable since RFC 1857 |
| Tuple elements | Declaration order (left to right) | `(a, b, c)` drops a, then b, then c |

**Practical impact**: Fields drop in declaration order, but `std::thread::JoinHandle` and `tokio::task::JoinHandle` **detach on drop** — they do not `join()` or `abort()`. For "close the channel, then wait": put `Option<Sender>` first, and in `Drop` do `self.tx.take(); self.handle.take().unwrap().join()...`.

### ManuallyDrop<T>

Suppresses automatic Drop. You take responsibility for dropping.

```rust
use std::mem::ManuallyDrop;

// Use case: prevent double-free in unsafe code
struct TwoPhaseBuffer {
    data: ManuallyDrop<Vec<u8>>,
    committed: bool,
}

impl Drop for TwoPhaseBuffer {
    fn drop(&mut self) {
        if !self.committed { println!("rolling back"); }
        unsafe { ManuallyDrop::drop(&mut self.data); }
    }
}

// Use case: union fields (only one variant is valid at a time)
union IntOrString {
    i: u64,
    s: ManuallyDrop<String>,  // String has Drop, must wrap
}
```

`ManuallyDrop` vs `mem::forget`:

| | `ManuallyDrop<T>` | `mem::forget(value)` |
|---|---|---|
| When | Wrap at construction | Consume later |
| Access inner | `&*md` / `&mut *md` | Value is gone |
| Drop later | `ManuallyDrop::drop(&mut md)` | Not possible |
| Use case | Fine-grained lifecycle | Fire-and-forget leak |

**Rule**: Use `ManuallyDrop` in unsafe abstractions where you need to control exactly when a destructor runs. In safe application code, you almost never need it.

## Anti-Patterns

- **Using `Rc` across threads.** `Rc` is `!Send`. Use `Arc`.
- **Deep nested `Box`/`Rc` when `Vec<T>` works.** Lists of known length → `Vec`.
- **`Cell::get()` on a non-Copy type.** `get()` needs `Copy`. For non-Copy, `replace`/`take` (often `Cell<Option<T>>`). Use `RefCell` when you need `&`/`&mut` to the interior.
- **`RefCell` across threads.** `RefCell` is `!Sync`. Use `Mutex` or `RwLock`.
- **Reaching for `unsafe` and `Pin` when a simpler design works.** Self-referential types are hard. Prefer `self_cell` or restructure.
- **Holding a `std::sync::MutexGuard` across `.await`.** Compile error on `tokio::spawn`. Nested `{ ... }` block, not `drop(guard)`. `std::sync::Mutex` is correct when the lock is not held across `.await`.
- **Assuming `JoinHandle` drop joins the thread.** It detaches. Join or abort explicitly in `Drop`.
- **`mem::forget` for "I don't want to drop this".** Use `ManuallyDrop` if you need later access, otherwise document the leak.

## See Also

- [concurrency.md](./concurrency.md) — `Arc` + `Mutex`/`RwLock` patterns
- [phantomdata.md](./phantomdata.md) — variance + drop-check interactions
- [async.md](./async.md) — pinning Futures, `Pin<&mut Self>`
- [unsafe.md](./unsafe.md) — raw pointers, `ManuallyDrop` in unsafe code