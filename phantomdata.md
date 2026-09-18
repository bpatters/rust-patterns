# PhantomData — Types That Carry No Data

Load this when: working with raw pointers and the compiler complains about ownership/lifetimes; designing a unit-of-measure or arena; seeing variance errors; debugging drop-check issues.

`PhantomData<T>` is a zero-sized type that tells the compiler "this struct is logically associated with `T`" — even though it doesn't contain a `T`. Affects variance, drop checking, and auto-trait inference, with no runtime cost.

## The Three Jobs of PhantomData

| Job | Example | What it does |
|---|---|---|
| Lifetime binding | `PhantomData<&'a T>` | Struct treated as borrowing `'a` |
| Ownership simulation | `PhantomData<T>` | Drop check assumes struct owns a `T` |
| Variance control | `PhantomData<fn(T)>` | Makes struct contravariant over `T` |

```rust
// Without PhantomData — compiler doesn't know this borrows or owns
struct Slice<'a, T> {
    ptr: *const T,
    len: usize,
}

// With PhantomData — compiler knows everything
struct Slice<'a, T> {
    ptr: *const T,
    len: usize,
    _marker: PhantomData<&'a T>,
}
```

## Lifetime Branding

Prevent mixing values from different "sessions" or "contexts". Branding needs a **fresh invariant lifetime per instance**, usually via an HRTB — `PhantomData` alone does not separate two `Arena<'a>` values in the same scope.

```rust
use std::cell::RefCell;
use std::marker::PhantomData;

struct ArenaHandle<'arena> {
    index: usize,
    // Invariant over 'arena, and Send + Sync (unlike *mut, which is !Send + !Sync)
    _brand: PhantomData<fn(&'arena ()) -> &'arena ()>,
}

struct Arena<'arena> {
    data: RefCell<Vec<String>>,
    _phantom: PhantomData<&'arena ()>,
}

/// Each call gets a unique lifetime that cannot be forged or mixed.
fn with_arena<R>(f: impl for<'arena> FnOnce(&Arena<'arena>) -> R) -> R {
    let arena = Arena { data: RefCell::new(Vec::new()), _phantom: PhantomData };
    f(&arena)
}

impl<'arena> Arena<'arena> {
    fn alloc(&self, value: String) -> ArenaHandle<'arena> { /* ... */ }
    fn get(&self, handle: &ArenaHandle<'arena>) -> String { /* ... */ }
}

with_arena(|arena1| {
    let h = arena1.alloc("hello".into());
    arena1.get(&h); // ✅
    // with_arena(|arena2| arena2.get(&h)); // ❌ handle branded to arena1
});
```

**Use**: arena allocators, generational references, session tokens, separating handles from different resources.

Do not use `PhantomData<*mut &'a ()>` for branding unless you *want* `!Send + !Sync`.

## Unit-of-Measure Pattern

Prevent mixing incompatible units at compile time, zero runtime cost:

```rust
struct Meters; struct Seconds; struct MetersPerSecond;

#[derive(Debug, Clone, Copy)]
struct Quantity<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl<U> Add for Quantity<U> {
    type Output = Quantity<U>;
    fn add(self, rhs: Self) -> Self::Output {
        Quantity::new(self.value + rhs.value)
    }
}

impl Div<Quantity<Seconds>> for Quantity<Meters> {
    type Output = Quantity<MetersPerSecond>;
    fn div(self, rhs: Quantity<Seconds>) -> Quantity<MetersPerSecond> {
        Quantity::new(self.value / rhs.value)
    }
}

let dist = Quantity::<Meters>::new(100.0);
let time = Quantity::<Seconds>::new(9.58);
let speed = dist / time;  // Quantity<MetersPerSecond>
// dist + time;         // ❌ Compile error: can't add Meters + Seconds
```

Pure type-system magic — `Quantity<Meters>` has the same layout as `f64`. Full unit safety, no runtime cost.

## PhantomData and Drop Check

The compiler uses `PhantomData` to decide whether a struct's destructor might access expired data:

```rust
// PhantomData<T> — compiler assumes we MIGHT drop a T → T must outlive us
struct OwningSemantic<T> {
    ptr: *const T,
    _marker: PhantomData<T>,
}

// PhantomData<*const T> — compiler assumes we DON'T own T → more permissive
struct NonOwningSemantic<T> {
    ptr: *const T,
    _marker: PhantomData<*const T>,
}
```

**Rule**: When wrapping raw pointers, choose carefully:
- Container that **owns** its data → `PhantomData<T>`
- View / reference type → `PhantomData<&'a T>` or `PhantomData<*const T>`

## Variance — Why the Parameter Type Matters

Variance determines whether a generic type can be substituted with a sub- or super-type. Getting it wrong causes either rejected-good-code or unsound-accepted-code.

| Variance | Meaning | Rust example |
|---|---|---|
| **Covariant** | Subtype flows through | `&'a T`, `Vec<T>`, `Box<T>` |
| **Contravariant** | Subtype flows against | `fn(T)` in parameter position |
| **Invariant** | No substitution | `&mut T`, `Cell<T>`, `UnsafeCell<T>` |

### Why `&'a T` is Covariant Over `'a`

```rust
fn print_str(s: &str) { println!("{s}"); }
let owned = String::from("hello");  // lives long
print_str(&owned);                   // works — covariance: long → short is safe
```

A longer-lived reference can always be used where a shorter one is needed.

### Why `&mut T` is Invariant Over `T`

```rust
fn evil(s: &mut &'static str) {
    let local = String::from("temporary");
    // *s = &local;  // would create a dangling &'static str
}
// Invariance prevents this: &'static str ≠ &'a str when mutating.
```

### How PhantomData Controls Variance

`PhantomData<X>` gives your struct the same variance as `X`:

```rust
struct Ref<'a, T> {
    ptr: *const T,
    _marker: PhantomData<&'a T>,           // Covariant over 'a and T
}
struct MutRef<'a, T> {
    ptr: *mut T,
    _marker: PhantomData<&'a mut T>,       // Covariant over 'a, INVARIANT over T
}
struct CallbackSlot<T> {
    _marker: PhantomData<fn(T)>,           // Contravariant over T
}
```

### PhantomData Variance Cheat Sheet

| PhantomData type | Variance over `T` | Send/Sync | Use when |
|---|---|---|---|
| `PhantomData<T>` | Covariant | Follows `T` | Logically own a `T` (dropck owns `T`) |
| `PhantomData<&'a T>` | Covariant | `Send`/`Sync` iff `T: Sync` | Borrow a `T` with lifetime `'a` |
| `PhantomData<&'a mut T>` | **Invariant** | `Send` iff `T: Send`, `Sync` iff `T: Sync` | Mutably borrow `T` |
| `PhantomData<*const T>` | Covariant | **Always `!Send + !Sync`** | Non-owning pointer that must not cross threads |
| `PhantomData<*mut T>` | **Invariant** | **Always `!Send + !Sync`** | Non-owning mutable pointer, same |
| `PhantomData<fn(T)>` | **Contravariant** | Always `Send + Sync` | `T` in argument position |
| `PhantomData<fn() -> T>` | Covariant | Always `Send + Sync` | `T` in return position |
| `PhantomData<fn(T) -> T>` | **Invariant** | Always `Send + Sync` | Invariance without losing `Send` |

### Decision Rule

- Borrowed view: start with `PhantomData<&'a T>` (covariant). Switch to `PhantomData<&'a mut T>` only if the abstraction hands out mutable access.
- Owning pointer (Vec-like): `*const T` **plus** `PhantomData<T>` so `Send`/`Sync` follow `T` and dropck treats it as owned. `*const T` alone makes the wrapper `!Send`.
- Need invariance + `Send`: `PhantomData<fn(T) -> T>`, not `*mut T`.
- `PhantomData<fn(T)>` (contravariant) almost never — only for callback-storage.

## Anti-Patterns

- **`PhantomData<T>` on a view/reference type.** Should be `PhantomData<&'a T>` or `PhantomData<*const T>`. Wrong choice makes the borrow checker too strict or too lax.
- **Missing `PhantomData` on a struct holding raw pointers.** Causes confusing borrow-check errors and soundness issues.
- **Wrong variance for a callback holder.** Use `PhantomData<fn(T)>` (contravariant) for callback slots that take `T`, not `PhantomData<T>`.
- **`PhantomData<*const T>` on a type you then `unsafe impl Send`.** Prefer a marker that already has the auto-traits you want (`PhantomData<T>` or `fn(...)`) so you don't need the unsafe impl.

## See Also

- [newtype-typestate.md](./newtype-typestate.md) — typestate builds on PhantomData
- [unsafe.md](./unsafe.md) — raw pointers + drop check
- [traits.md](./traits.md) — `Send`/`Sync` auto-trait inference is affected by PhantomData