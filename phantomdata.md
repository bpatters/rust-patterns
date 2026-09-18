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

Prevent mixing values from different "sessions" or "contexts":

```rust
struct ArenaHandle<'arena> {
    index: usize,
    _brand: PhantomData<*mut &'arena ()>,
}

struct Arena<'arena> {
    data: RefCell<Vec<String>>,
    _phantom: PhantomData<&'arena ()>,
}

impl<'arena> Arena<'arena> {
    fn alloc(&self, value: String) -> ArenaHandle<'arena> { /* ... */ }
    fn get(&self, handle: &ArenaHandle<'arena>) -> String { /* ... */ }
}

// Can't use handle from arena1 with arena2 — compile-time error.
```

**Use**: arena allocators, generational references, session tokens, separating handles from different resources.

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

| PhantomData type | Variance over `T` | Variance over `'a` | Use when |
|---|---|---|---|
| `PhantomData<T>` | Covariant | — | Logically own a `T` |
| `PhantomData<&'a T>` | Covariant | Covariant | Borrow a `T` with lifetime `'a` |
| `PhantomData<&'a mut T>` | **Invariant** | Covariant | Mutably borrow `T` |
| `PhantomData<*const T>` | Covariant | — | Non-owning pointer |
| `PhantomData<*mut T>` | **Invariant** | — | Non-owning mutable pointer |
| `PhantomData<fn(T)>` | **Contravariant** | — | `T` appears in argument position |
| `PhantomData<fn() -> T>` | Covariant | — | `T` appears in return position |
| `PhantomData<fn(T) -> T>` | **Invariant** | — | `T` in both positions |

### Decision Rule

- Start with `PhantomData<&'a T>` (covariant). It's the most permissive — callers can shorten lifetimes.
- Switch to `PhantomData<&'a mut T>` (invariant) only if your abstraction hands out mutable access.
- Use `PhantomData<fn(T)>` (contravariant) almost never — only for callback-storage scenarios.

## Anti-Patterns

- **`PhantomData<T>` on a view/reference type.** Should be `PhantomData<&'a T>` or `PhantomData<*const T>`. Wrong choice makes the borrow checker too strict or too lax.
- **Missing `PhantomData` on a struct holding raw pointers.** Causes confusing borrow-check errors and soundness issues.
- **Wrong variance for a callback holder.** Use `PhantomData<fn(T)>` (contravariant) for callback slots that take `T`, not `PhantomData<T>`.

## See Also

- [newtype-typestate.md](./newtype-typestate.md) — typestate builds on PhantomData
- [unsafe.md](./unsafe.md) — raw pointers + drop check
- [traits.md](./traits.md) — `Send`/`Sync` auto-trait inference is affected by PhantomData