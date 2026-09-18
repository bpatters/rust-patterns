# Traits In Depth

Load this when: designing a new trait; choosing associated type vs generic parameter; deciding `impl Trait` vs `dyn Trait`; figuring out dyn-compatibility / object-safety errors (`E0038`); writing `async fn` in traits; upcasting `dyn Sub` to `dyn Super`; writing extension traits; using GATs.

## Associated Types vs Generic Parameters

```rust
// Associated type: ONE natural output per implementing type
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// Generic parameter: MANY possible impls per type
trait From<T> {
    fn from(value: T) -> Self;
}
```

| Use | When |
|---|---|
| **Associated type** | Exactly ONE natural output. `Iterator::Item`, `Deref::Target`, `Add::Output` |
| **Generic parameter** | Type can meaningfully implement for MANY types. `From<T>`, `AsRef<T>`, `PartialEq<Rhs>` |

**Intuition**: If it makes sense to ask "what is the Item of this iterator?", use associated type. If it makes sense to ask "can this convert to `f64`? to `String`?", use a generic parameter.

## Generic Associated Types (GATs)

Enables **lending iterators** that return references tied to the borrow of `&self`.

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>>;
}
```

**When you need GATs**: lending iterators, streaming parsers, or any trait where the associated type's lifetime depends on `&self`. For most code, plain associated types are sufficient.

## Supertraits

```rust
trait Error: fmt::Display + fmt::Debug {
    fn source(&self) -> Option<&(dyn Error + 'static)> { None }
}
// Implementing Error requires Display + Debug.
```

Build hierarchies like `Entity: Identifiable + Timestamped` to compose required capabilities.

**Trait upcasting (1.86+)**: `&dyn Sub` coerces to `&dyn Super` when `trait Sub: Super`. Same for `&mut`, `Box`, `Rc`, `Arc`. The old `as_any` / `Deref`-to-supertrait hacks are unnecessary. Inherent methods on `dyn Super` (e.g. `Any::downcast_ref`) are **not** in the sub object's method set — upcast first:

```rust
trait Store: std::any::Any {
    fn name(&self) -> &str;
}
struct Postgres;
impl Store for Postgres {
    fn name(&self) -> &str { "pg" }
}
fn downcast(s: &dyn Store) -> Option<&Postgres> {
    (s as &dyn std::any::Any).downcast_ref()
}
```

## Blanket Implementations

Implement for all types satisfying a bound. Use sparingly — they're powerful but irreversible (orphan rules + coherence).

```rust
impl<T: fmt::Display> ToString for T {
    fn to_string(&self) -> String { format!("{self}") }
}
// Now every Display type gets to_string() for free.
```

**Caution**: You can't add a more specific impl for a type covered by a blanket impl. Design carefully.

## Marker Traits

No methods — they mark a property.

```rust
/// Marker: this sensor has been factory-calibrated
trait Calibrated {}

struct CalibratedSensor { /* ... */ }
impl Calibrated for CalibratedSensor {}

fn record_measurement<S: Calibrated>(sensor: &S) {
    // Only calibrated sensors accepted — compile-time enforcement.
}
```

Standard library markers: `Send`, `Sync`, `Unpin`, `Sized`, `Copy`.

Connects to **type-state pattern** (see [newtype-typestate.md](./newtype-typestate.md)).

## Dyn Compatibility (formerly Object Safety)

A trait is **dyn-compatible** (usable as `dyn Trait`) only if it can have a vtable. The compiler error is still `E0038`; older docs say "object-safe."

1. `Sized` is not a supertrait (`trait Foo: Sized` is fatal)
2. No generic *type* parameters on dispatchable methods (generic *lifetimes* are OK)
3. No `Self` except as the receiver — not in args (`&Self`), not in returns (`-> Self`). **`Box<Self>` does not help** (it still names `Self`). Return `Box<dyn Trait>` instead
4. Dispatchable receivers: `&self`, `&mut self`, `self: Box<Self>` / `Rc<Self>` / `Arc<Self>` / `Pin<P>`. Bare `self` by value is *not* dispatchable (implicit `where Self: Sized`) — the trait can still be `dyn`, but that method cannot be called on it
5. Associated functions without a receiver must opt out: `fn create() -> Self where Self: Sized`
6. All supertraits must themselves be dyn-compatible
7. No associated constants; no GATs. Plain associated types are OK but must be specified at the use site: `dyn Iterator<Item = u32>`, never a bare `dyn Iterator`
8. No `async fn` or `-> impl Trait` on dispatchable methods (opaque types). Workaround: `Pin<Box<dyn Future<Output = T> + '_>>` or `Box<dyn Iterator<Item = U> + '_>`

```rust
// ✅ Dyn compatible
trait Drawable {
    fn draw(&self);
}

// ❌ Self in return
trait Cloneable {
    fn clone_self(&self) -> Self;
}

// ❌ Self in argument — why PartialEq isn't dyn compatible
trait Comparable {
    fn equals(&self, other: &Self) -> bool;
}

// ❌ Box<Self> still names Self
trait Spawner {
    fn spawn(&self) -> Box<Self>;
}

// ✅ Clone through a trait object
trait CloneableDyn {
    fn clone_box(&self) -> Box<dyn CloneableDyn>;
}

// ❌ Generic method (vtable can't hold infinite monomorphizations)
trait Converter {
    fn convert<T>(&self) -> T;
}

// ✅ Associated function opted out of the vtable
trait Factory {
    fn describe(&self) -> String;
    fn create() -> Self where Self: Sized;
}
```

**Workaround**: add `where Self: Sized` to exclude a method from the vtable.

**Rule**: If you plan to use `dyn Trait`, keep methods simple. When in doubt, try `let _: Box<dyn YourTrait>;` and let the compiler tell you.

## Trait Objects Under the Hood

`&dyn Trait` / `Box<dyn Trait>` is a **fat pointer**: `(data_ptr, vtable_ptr)` — 16 bytes on 64-bit.

| Aspect | Static (`impl Trait`) | Dynamic (`dyn Trait`) |
|---|---|---|
| Call overhead | Zero — inlined | One indirect call |
| Inlining | ✅ | ❌ (opaque fn pointer) |
| Binary size | One copy per type | Shared |
| Pointer size | 1 word | 2 words |
| Heterogeneous collection | ❌ | ✅ `Vec<Box<dyn T>>` |

**When vtable cost matters**: In tight loops calling a trait method millions of times, indirection + no inlining can be 2-10× slower. For cold paths, config, plugin architectures — flexibility wins.

## Higher-Ranked Trait Bounds (HRTBs)

`for<'a>` says "works for ALL lifetimes". You'll rarely write this yourself — it appears in error messages.

```rust
fn apply<F>(f: F, data: &str) -> &str
where
    F: for<'a> Fn(&'a str) -> &'a str,
{ f(data) }
```

`serde::DeserializeOwned` is defined as `trait DeserializeOwned: for<'de> Deserialize<'de> {}` — can be deserialized from data with any lifetime (the result doesn't borrow from input).

## `impl Trait` — Argument vs Return Position

```rust
// Argument position: caller picks the type
fn print_all(items: impl Iterator<Item = i32>) { /* ... */ }

// Return position: function picks one concrete type
fn evens(limit: i32) -> impl Iterator<Item = i32> {
    (0..limit).filter(|x| x % 2 == 0)
}
```

| Position | Who picks type | Equivalent to |
|---|---|---|
| APIT `fn(x: impl T)` | Caller | `fn<X: T>(x: X)` |
| RPIT `fn() -> impl T` | Callee | Existential type |

**RPITIT / `async fn` in traits (1.75+)**: `fn items(&self) -> impl Iterator<Item = &str>` and `async fn fetch(&self)` are native. **Not dyn-compatible.** Need `dyn Container`? Keep `Box<dyn Iterator<Item = &str> + '_>` or `Pin<Box<dyn Future<...> + Send + '_>>` (or split the trait). See [async.md](./async.md) for `Send` bounds on public async traits (`trait-variant`).

**Precise capturing (`use<>`, 1.82; default in edition 2024)**: RPIT captures all in-scope lifetimes in 2024. To *not* capture one, write `-> impl Trait + use<T>` (capture only `T`). The old `Captures<'a>` trick is obsolete.

## `impl Trait` vs `dyn Trait` Decision

```text
Do you know the concrete type at compile time?
├── YES → impl Trait or generics (zero cost, inlinable)
└── NO  → Need heterogeneous collection?
     ├── YES → Vec<Box<dyn T>>
     └── NO  → Same trait across an API boundary?
          ├── YES → dyn Trait
          └── NO  → generics / impl Trait
```

## Type Erasure with `Any` and `TypeId`

For storing truly unknown types and downcasting later.

```rust
use std::any::Any;

struct AnyMap(HashMap<TypeId, Box<dyn Any + Send>>);

impl AnyMap {
    fn insert<T: Any + Send + 'static>(&mut self, value: T) {
        self.0.insert(TypeId::of::<T>(), Box::new(value));
    }
    fn get<T: Any + Send + 'static>(&self) -> Option<&T> {
        self.0.get(&TypeId::of::<T>())?.downcast_ref()
    }
}
```

**Use**: plugin systems, event buses, error downcasting. **Avoid** when the set of types is known — prefer generics or trait objects.

## Extension Traits — Adding Methods to Foreign Types

Rust's orphan rule prevents `impl ForeignTrait for ForeignType`. Workaround: define a new trait, blanket-impl it for relevant types, callers import the trait.

```rust
pub trait IteratorExt: Iterator {
    fn mean(self) -> Option<f64> where Self: Sized, Self::Item: Into<f64>;
}

impl<I: Iterator> IteratorExt for I {
    fn mean(self) -> Option<f64> { /* ... */ }
}

// Usage:
use crate::IteratorExt;
readings.iter().copied().mean();
```

**Naming convention**: `<Trait>Ext` (e.g., `Itertools`, `StreamExt`, `FutureExt`, `AsyncReadExt`).

### When to Use

| Situation | Extension trait? |
|---|---|
| Add convenience methods to foreign types | ✅ |
| Group domain logic on generic collections | ✅ |
| Method needs access to private fields | ❌ use newtype |
| Method logically belongs on your type | ❌ inherent methods |
| Want method without any import | ❌ inherent methods only |

## Enum Dispatch — Static Polymorphism Without dyn

For a closed set of types, replace `dyn Trait` with an enum whose variants hold the concrete types.

```rust
trait Sensor {
    fn read(&self) -> f64;
}

enum AnySensor {
    Gps(Gps),
    Thermometer(Thermometer),
}

impl Sensor for AnySensor {
    fn read(&self) -> f64 {
        match self {
            AnySensor::Gps(s) => s.read(),
            AnySensor::Thermometer(s) => s.read(),
        }
    }
}
```

| | `dyn Trait` | Enum dispatch |
|---|---|---|
| Dispatch cost | Vtable indirection (~2ns) | Branch prediction (~0.3ns) |
| Heap allocation | Usually (`Box`) | None (inline) |
| Open to new types | ✅ | ❌ (closed set) |
| Code size | Shared | One copy per variant |
| Trait must be dyn-compatible | Yes | No |

**When**: closed set, hot path, < ~20 variants (manual enum). For 10+ variants with many methods, use the `enum_dispatch` crate to automate.

## Capability Mixins — Zero-Cost Composition

Ingredient traits (associated types) + mixin traits (default methods) + blanket impls = Rust's compile-time mixin pattern.

```rust
// Ingredient: one bus capability
pub trait HasI2c {
    type I2c: I2cBus;
    fn i2c(&self) -> &Self::I2c;
}

// Mixin: fan diagnostics (needs I2C + GPIO)
pub trait FanDiagMixin: HasI2c + HasGpio {
    fn read_fan_rpm(&self, fan_id: u8) -> io::Result<u32> {
        // default body using self.i2c().i2c_read(...)
        todo!()
    }
}

// The magic line — any type with the ingredients gets the methods for free:
impl<T: HasI2c + HasGpio> FanDiagMixin for T {}
```

Conditional methods via per-method `where` bounds: `fn bulk_read(...) where Self::Spi: DmaCapable`. The method only exists when the bound is satisfied — compile-time "respond_to?".

**Use when**: multiple operations share bus dependencies; tests need different ingredient subsets; you want zero-cost composition without inheritance.

## Typed Commands — GADT-Style Return Safety

Bind each command to its return type via associated types. Mixing up units (RPM as Celsius) becomes a compile error.

```rust
trait IpmiCmd {
    type Response;
    fn net_fn(&self) -> u8;
    fn cmd_byte(&self) -> u8;
    fn payload(&self) -> Vec<u8>;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}

struct ReadTemp { sensor_id: u8 }
impl IpmiCmd for ReadTemp {
    type Response = Celsius;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Celsius> {
        Ok(Celsius(raw[0] as i8 as f64))
    }
}
```

Replaces `Vec<u8>` swampland with types like `Celsius`, `Rpm`, `Volts` that cannot be confused.

## Anti-Patterns

- **Returning `Self` (or `Box<Self>`, or taking `&Self`) from a trait you want as `dyn`.** `Box<Self>` does not help. Return `Box<dyn Trait>`, or add `where Self: Sized` to opt the method out of the vtable.
- **Blanket impls that conflict with future specific impls.** Orphan rules + coherence make this irreversible.
- **`dyn Trait` for performance-critical hot paths.** Use generics or enum dispatch.
- **Trait with too many responsibilities.** Split into focused traits (ISP). Use supertraits to require combinations.
- **Sealing a trait you later wish was implementable.** Use `pub trait Foo: Sealed` only when you genuinely need to prevent downstream impls.

## See Also

- [generics.md](./generics.md) — static vs dynamic dispatch decision
- [newtype-typestate.md](./newtype-typestate.md) — type-state with marker traits, conditional impls
- [phantomdata.md](./phantomdata.md) — variance for trait method signatures
- [smart-pointers.md](./smart-pointers.md) — `Pin<&mut Self>` for futures