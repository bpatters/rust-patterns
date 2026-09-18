# Rust Patterns — Chapter Index

Distilled from [Microsoft's Rust Patterns & Engineering How-Tos](https://microsoft.github.io/RustTraining/rust-patterns-book/).

Start with [SKILL.md](./SKILL.md) for orientation, decision trees, and the anti-pattern cheatsheet. Then drill into specific chapter files when relevant.

## Part I: Type-Level Patterns

- [**generics.md**](./generics.md) — Monomorphization, const generics, `const fn`
- [**traits.md**](./traits.md) — Associated types, GATs, blanket impls, vtables, object safety
- [**newtype-typestate.md**](./newtype-typestate.md) — Newtype, type-state, builder with states, config trait
- [**phantomdata.md**](./phantomdata.md) — Lifetime branding, variance, unit-of-measure, drop check

## Part II: Concurrency & Runtime

- [**channels.md**](./channels.md) — `mpsc`, crossbeam, `select!`, actor pattern
- [**concurrency.md**](./concurrency.md) — Threads, scoped threads, rayon, Mutex/RwLock/atomics, lock-free
- [**closures.md**](./closures.md) — Fn/FnMut/FnOnce, combinators, `with` pattern
- [**functional-style.md**](./functional-style.md) — Iterator chains, Option/Result combinators, when loops win
- [**smart-pointers.md**](./smart-pointers.md) — Box/Rc/Arc/Weak/Cell/RefCell/Cow/Pin/ManuallyDrop

## Part III: Systems & Production

- [**error-handling.md**](./error-handling.md) — thiserror vs anyhow, `?`, panic, `catch_unwind`
- [**serialization.md**](./serialization.md) — serde fundamentals, repr(C), zerocopy, bytes::Bytes
- [**unsafe.md**](./unsafe.md) — 5 superpowers, sound abstractions, FFI, arenas
- [**macros.md**](./macros.md) — `macro_rules!`, proc macros, syn/quote
- [**testing.md**](./testing.md) — Unit/integration/doc tests, proptest, criterion
- [**api-design.md**](./api-design.md) — Module layout, ergonomic params, Parse-Don't-Validate, feature flags
- [**async.md**](./async.md) — Tokio, Future, Send bounds, common pitfalls

## Quick Lookup

| Need | File |
|---|---|
| Choose static vs dynamic dispatch | [traits.md](./traits.md) |
| Build a compile-time state machine | [newtype-typestate.md](./newtype-typestate.md) |
| Decide Mutex vs RwLock vs atomics vs channels | [concurrency.md](./concurrency.md), [channels.md](./channels.md) |
| Pick thiserror vs anyhow | [error-handling.md](./error-handling.md) |
| Design an ergonomic public API | [api-design.md](./api-design.md) |
| Use newtype with invariants safely | [newtype-typestate.md](./newtype-typestate.md) |
| Write Send-safe async code | [async.md](./async.md) |
| Reduce boilerplate | [macros.md](./macros.md) |
| Wrap a C library | [unsafe.md](./unsafe.md) |
| Validate input at the boundary | [api-design.md](./api-design.md), [error-handling.md](./error-handling.md) |
| Parse binary data without copying | [serialization.md](./serialization.md) |
| Set up property tests | [testing.md](./testing.md) |
| Decide iterator chain vs loop | [functional-style.md](./functional-style.md) |
| Choose smart pointer | [smart-pointers.md](./smart-pointers.md) |
| Use the right closure trait bound | [closures.md](./closures.md) |