# Generics — The Full Picture

Load this when: choosing between generics, enums, and trait objects; designing const generics; deciding whether a function should be `const fn`.

## Monomorphization: Zero-Cost Generics

Generics compile to a specialized copy of the function for each concrete type. No runtime dispatch, no vtable — but no erasure either.

```rust
fn max_of<T: PartialOrd>(a: T, b: T) -> T {
    if a >= b { a } else { b }
}

// Compiler generates max_of_i32, max_of_f64, max_of_str — three real functions.
```

Rust **monomorphizes** (one specialized copy per type), unlike Java erasure. The difference from **C++ templates** is that trait bounds are checked at the **definition site** (`T: PartialOrd` is required when you write `fn max_of`) — you can only call methods that appear in the bounds. Call-site errors are "T doesn't impl Trait", not deep instantiation failures.

**Cost**: Binary size. Each unique type used generates a copy. If `serialize<T: Serialize>` is called with 50 types, the binary has 50 copies.

### Mitigation: Extract the Non-Generic Core ("outline")

```rust
fn serialize<T: serde::Serialize>(value: &T) -> Result<Vec<u8>, serde_json::Error> {
    let json_value = serde_json::to_value(value)?;  // generic part
    serialize_value(json_value)                      // monomorphic part
}

fn serialize_value(value: serde_json::Value) -> Result<Vec<u8>, serde_json::Error> {
    serde_json::to_vec(&value)  // exists ONCE in the binary
}
```

### Mitigation: Trait Objects for Cold Paths

```rust
fn log_item(item: &dyn std::fmt::Display) {
    println!("[LOG] {item}");
}
// One copy, vtable dispatch — fine for logging/error paths.
```

**Rule of thumb**: Use generics on hot paths (inlined, optimized per-type). Use `dyn Trait` on cold paths (logging, config, error handling) where a few nanoseconds of vtable indirection is invisible.

## Generics vs Enum vs dyn Trait — Decision Guide

```text
Known set of types?
├── Closed set (variants never added by users) → enum (exhaustive match, zero cost)
├── Open set, hot path (millions of calls)     → generics/<T: Trait> (inlined)
├── Open set, cold path (logging, errors, cfg) → dyn Trait (one vtable indirection)
└── Need heterogeneous collection              → Vec<Box<dyn Trait>> or enum dispatch
```

| Approach | Dispatch | Extensible? | Overhead |
|---|---|---|---|
| Generics | Static | ✅ open set | Zero — inlined |
| Enum | Match | ❌ closed set | Zero — no vtable |
| `dyn Trait` | Dynamic | ✅ open set | Vtable ptr + indirect call |

### Closed Set → Enum

```rust
enum Shape {
    Circle(f64),
    Rect(f64, f64),
    Triangle(f64, f64, f64),
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle(r) => std::f64::consts::PI * r * r,
            Shape::Rect(w, h) => w * h,
            Shape::Triangle(a, b, c) => {
                let s = (a + b + c) / 2.0;
                (s * (s - a) * (s - b) * (s - c)).sqrt()
            }
        }
    }
}
// Adding a new variant forces updating ALL match arms — compile-time enforcement.
```

### Open Set, Hot Path → Generics

```rust
fn process<H: Handler>(handler: H, request: Request) -> Response {
    handler.handle(request)  // monomorphized, inlined per H
}
```

### Open Set, Heterogeneous Collection → dyn Trait

```rust
fn log_all(items: &[Box<dyn std::fmt::Display>]) {
    for item in items {
        println!("{item}");  // vtable dispatch per item
    }
}
```

## Const Generics

Parameterize over **constant values**, not just types.

```rust
struct Matrix<const ROWS: usize, const COLS: usize> {
    data: [[f64; COLS]; ROWS],
}

impl<const ROWS: usize, const COLS: usize> Matrix<ROWS, COLS> {
    fn transpose(&self) -> Matrix<COLS, ROWS> {
        // The compiler enforces dimensional correctness:
        // multiply takes Matrix<M, N> and Matrix<N, P>, returns Matrix<M, P>
        todo!()
    }
}

let a = Matrix::<2, 3>::new();
let b = Matrix::<3, 4>::new();
let c = multiply(&a, &b);  // 2x4 ✅
// multiply(&a, &Matrix::<5, 5>::new());  // ❌ Compile error: dimensions don't match
```

Use when: array sizes, buffer capacities, fixed matrix dimensions, compile-time dimensional analysis.

## const fn — Compile-Time Evaluation

`const fn` marks a function as evaluable at compile time. Results can be used in `const` and `static` contexts.

```rust
const fn celsius_to_fahrenheit(c: f64) -> f64 {
    c * 9.0 / 5.0 + 32.0
}

const BOILING_F: f64 = celsius_to_fahrenheit(100.0);  // Computed at compile time
```

**Use cases**: Lookup tables, register masks, threshold arrays, simple arithmetic constants. Eliminates the need for `lazy_static!` / `OnceLock` when the value is purely compile-time computable.

**Allowed in `const fn`**: arithmetic, bit ops, control flow (`if`/`match`/`loop`/`while`), references, calling other `const fn`s, `panic!` (compile error if reached at const time), basic float ops, many inherent std methods.

**NOT allowed** (stable): heap allocation (`Box`/`Vec`/`String`), I/O, generic trait method calls (const traits are not stable). The body must be const-evaluable — a `const` context requires compile-time evaluation or it is a hard error.

**Idiomatic advice**: Make constructors and simple utility functions `const fn` whenever possible — costs nothing, enables callers to use them in const contexts.

## Anti-Patterns

- **Monomorphization bloat from a single generic over too many types.** Extract the type-specific core, leave the rest monomorphic (see "outline" pattern).
- **Using `dyn Trait` for performance-critical inner loops.** Each call is an indirect branch; the compiler can't inline.
- **Using generics when the set is closed and small.** An enum with exhaustive matching gives you compile-time enforcement that no caller missed a variant.

## See Also

- [traits.md](./traits.md) — `impl Trait` vs `dyn Trait`, associated types, dyn compatibility, `use<>` capturing
- [smart-pointers.md](./smart-pointers.md) — `Box<dyn Trait>` for the heterogeneous-collection case
- [api-design.md](./api-design.md) — when to seal a trait vs leave it open for downstream impls