---
name: rust-patterns
description: Use when writing, reviewing, or modifying Rust code in a Rust project (Cargo.toml or .rs files present) — specifically when working with generics, traits, lifetimes, ownership, error handling (thiserror 2/anyhow/?), async/await (tokio, async fn in traits, AsyncFn), type-state, newtype, PhantomData, smart pointers (Box/Rc/Arc/Cell/RefCell/Cow/Pin), monomorphization, dyn Trait / object-safety (E0038), trait upcasting, concurrency (Mutex/RwLock/atomics/channels/rayon), serde, postcard, unsafe (edition 2024 FFI), macros, or public API design. Triggers when choosing between enum/generics/dyn Trait, deciding thiserror vs anyhow, designing compile-time state machines, writing Send-safe async code, applying parse-don't-validate, or picking between Mutex/RwLock/atomics/channels/actors.
---

# Rust Patterns

Idiomatic Rust patterns distilled from Microsoft's Rust Patterns & Engineering How-Tos. This skill orients you to the right pattern; per-chapter reference files (in this directory) give the full details.

## Core Principles

1. **Make illegal states unrepresentable.** Move checks from runtime to compile time using newtypes, type-state, and the type system. If a value exists, it is valid.
2. **Parse, don't validate.** Don't accept data and check it later. Convert raw input into a validated type at the boundary (`TryFrom`, `FromStr`); the validated type then carries its invariants forever.
3. **Zero-cost abstractions.** Generics, traits, and lifetimes compile to the same code as hand-written specializations. Use generics on hot paths; use `dyn Trait` on cold paths (logging, errors, config) to avoid monomorphization copies.
4. **Static dispatch by default; dynamic only when needed.** Prefer generics/`impl Trait` over `dyn Trait` unless you need heterogeneous collections or runtime polymorphism.
5. **Errors are values, not exceptions.** `Result<T, E>` for expected failures; `panic!` only for bugs (violated invariants). Libraries: structured `thiserror` enums. Applications: `anyhow` for ergonomic propagation.
6. **The borrow checker is your friend.** If lifetimes fight you, the design probably needs a newtype, an owned value, or a restructure — not a workaround.

## Top Decision Trees

### Dispatch: Enum vs Generics vs dyn Trait

```
Known set of types?
├── Closed set (variants never added by users) → enum (exhaustive match, zero cost)
├── Open set, hot path (millions of calls)     → generics/<T: Trait> (inlined)
├── Open set, cold path (logging, errors, cfg) → dyn Trait (one vtable indirection)
└── Need heterogeneous collection              → Vec<Box<dyn Trait>> or enum dispatch
```

See: [traits.md](./traits.md), [generics.md](./generics.md)

### Shared State Across Threads

```
Simple counter/flag?            → AtomicU64 / AtomicBool (lock-free)
Short critical section?         → Mutex<T> (std or parking_lot)
Many readers, rare writers?     → RwLock<T>
Lazy one-time initialization?   → OnceLock<T> or LazyLock<T>
Complex state with invariants?  → Actor + channels (serialize access)
Collection processing?          → rayon::par_iter()
```

See: [concurrency.md](./concurrency.md), [channels.md](./channels.md)

### Error Handling

```
Library public API?     → thiserror with #[derive(Error)] + #[from]
Application binary?     → anyhow::Result<T> + .context()
Propagating through ?   → make sure From<E> is implemented (#[from])
Expected failure?       → Result<T, E>
Bug / violated invariant? → panic! (or assert!)
FFI boundary / thread pool? → catch_unwind to isolate panic
```

See: [error-handling.md](./error-handling.md)

### Async Pitfalls (most common bugs)

| Bug | Cause | Fix |
|---|---|---|
| Blocking the executor | `std::thread::sleep` or CPU work in async fn | `tokio::task::spawn_blocking` |
| Sequential when meant concurrent | `let a = foo().await; let b = bar().await;` | `tokio::join!` or `tokio::spawn` |
| `Send` bound error | `Rc` or `std::sync::MutexGuard` still in scope across `.await` | nested `{ ... }` block so the guard drops before `.await`; or `Arc` / `tokio::sync::Mutex` |
| Future does nothing | Called `async fn` but never `.await`ed or `spawn`ed | always `.await` or `tokio::spawn` the future |

See: [async.md](./async.md)

## Top 10 Anti-Patterns

1. **`Deref` for API convenience.** If your newtype has invariants, do NOT impl `Deref` — it lets callers bypass them. Use explicit `as_str()` / `get()` methods.
2. **`.unwrap()` in library code.** Return `Result`; let callers decide. Use `.unwrap()` only in tests and prototypes.
3. **`panic!` for expected errors.** File not found, network timeout → `Result`. Use `panic!` only for bugs.
4. **`clone()` to satisfy the borrow checker.** Usually a sign the data should be owned elsewhere, or you need a newtype. Stop and think.
5. **`lazy_static!`** — replaced by `OnceLock` / `LazyLock` in std.
6. **`std::sync::MutexGuard` held across `.await`.** On `tokio::spawn` this is a **compile error** (`MutexGuard: !Send`), not a runtime panic. End the guard's **scope** before `.await` (a nested `{ ... }` block — `drop(guard)` is not enough for Send analysis). Short critical section with no `.await` while held → `std::sync::Mutex`. Must hold across `.await` → `tokio::sync::Mutex`.
7. **Stringly-typed APIs.** `fn connect(host: &str, port: &str)` → use validated newtypes (`Host`, `Port`).
8. **Boolean flags that should be enums.** `fn search(needle: &str, case_sensitive: bool)` → `enum CaseSensitivity { Sensitive, Insensitive }`.
9. **Long `.and_then()` chains when `?` works.** If every closure is `|x| next_step(x)`, write it as `?` with named intermediates.
10. **Wrapping `unsafe` without a `SAFETY:` comment.** Every `unsafe` block must justify why it's sound. Encapsulate behind a safe API.

## File Index

Per-chapter reference files for deeper coverage:

| File | Topic | Load when... |
|---|---|---|
| [generics.md](./generics.md) | Monomorphization, const generics, `const fn` | Choosing generic vs dyn vs enum; writing `const fn` |
| [traits.md](./traits.md) | Associated types, GATs, dyn compatibility, vtables | Designing a trait; `dyn Trait` vs `impl Trait`; `E0038` |
| [newtype-typestate.md](./newtype-typestate.md) | Newtype, type-state, builder, config trait | Distinguishing similar primitives; state machines |
| [phantomdata.md](./phantomdata.md) | Lifetime branding, variance, unit-of-measure | Raw pointers, FFI, drop-check issues, variance |
| [channels.md](./channels.md) | `mpsc`, crossbeam, `select!`, actor pattern | Multi-task message passing; serialization |
| [concurrency.md](./concurrency.md) | Threads, rayon, Mutex, RwLock, atomics | Shared state, parallelism, lock-free patterns |
| [closures.md](./closures.md) | Fn/FnMut/FnOnce, combinators, `with` pattern | Higher-order APIs; bracket-resource-access |
| [functional-style.md](./functional-style.md) | Iterator chains, combinators, combinator chains | Data pipelines; combinator vs loop decision |
| [smart-pointers.md](./smart-pointers.md) | Box/Rc/Arc/Weak/Cell/RefCell/Cow/Pin | Choosing pointer type; interior mutability; pinning |
| [error-handling.md](./error-handling.md) | thiserror, anyhow, `?`, panic vs Result | Designing error types; choosing thiserror vs anyhow |
| [serialization.md](./serialization.md) | serde, postcard, repr(C), zerocopy, bytes::Bytes | JSON/binary parsing; FFI data layouts |
| [unsafe.md](./unsafe.md) | 5 superpowers, sound abstractions, FFI, arenas | Writing `unsafe` code; calling C; allocators |
| [macros.md](./macros.md) | macro_rules!, proc macros, syn/quote | Reducing boilerplate; writing derives |
| [testing.md](./testing.md) | Unit/integration/doc, proptest, criterion | Setting up tests; property tests; benchmarks |
| [api-design.md](./api-design.md) | Module layout, ergonomic params, Parse-Don't-Validate | Public crate API; `impl Into`/`AsRef`/`Cow`; `TryFrom` |
| [async.md](./async.md) | Tokio, Future, async fn in traits, AsyncFn, Send bounds | Async I/O; concurrent tasks; runtime choice |

## Quick Reference Card

### Trait Bounds Cheat Sheet

| Bound | Meaning |
|---|---|
| `T: Clone` | Can be duplicated |
| `T: Send` | Can move to another thread |
| `T: Sync` | `&T` can be shared between threads |
| `T: 'static` | No borrows shorter than `'static` (`String`/`Vec`/`i32` are `'static`) |
| `T: Sized` | Size known at compile time (default bound) |
| `T: ?Sized` | Size may not be known (`[T]`, `dyn Trait`) |
| `F: Fn(A) -> B` | Callable repeatedly, borrows immutably |
| `F: FnMut(A) -> B` | Callable repeatedly, may mutate state |
| `F: FnOnce(A) -> B` | Callable exactly once, may consume state |
| `F: AsyncFn(A) -> B` | Async callback (`async || { ... }`, 1.85+); also `AsyncFnMut` / `AsyncFnOnce` |

### Common Derives

```rust
#[derive(
    Debug,            // {:?} formatting
    Clone,            // .clone()
    Copy,             // Implicit copy (only simple types)
    PartialEq, Eq,    // == comparison
    PartialOrd, Ord,  // < > comparison + sorting
    Hash,             // HashMap/HashSet key
    Default,          // Type::default()
)]
```

### Module Visibility

```text
pub           → visible everywhere
pub(crate)    → within this crate
pub(super)    → parent module
pub(in path)  → specific ancestor
(nothing)     → current module + children
```

### Lifetime Elision Rules (compiler does these automatically)

1. Each reference parameter gets its own lifetime
2. Exactly ONE input lifetime → used for all outputs
3. `&self` / `&mut self` → its lifetime is used for outputs

You must write explicit lifetimes when: multiple input refs + output ref, struct fields that hold refs, `'static` bounds.

### Choosing the Closure Trait Bound

- Need to call concurrently (by `&self`)? → `Fn` (also needs `Send + Sync` to cross threads)
- Default — accepts both `Fn` and `FnMut` closures → `FnMut`
- Need to consume captures? → `FnOnce`
- Async callback? → `AsyncFn` / `AsyncFnMut` / `AsyncFnOnce` (see [async.md](./async.md))

## See Also

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — checklist for polished public APIs
- [Effective Rust](https://www.lurklurk.org/effective-rust/) — 35 specific ways to improve Rust code
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) — catalog of idiomatic patterns
- [Rust Atomics and Locks](https://marabos.nl/atomics/) — Mara Bos on concurrency primitives
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — unsafe Rust and PhantomData/variance/dropck
