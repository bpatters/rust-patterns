# Closures and Higher-Order Functions

Load this when: designing a higher-order API; choosing between `Fn`/`FnMut`/`FnOnce`; deciding closure vs trait-object parameter; building iterator chains; needing bracketed resource access.

## Fn, FnMut, FnOnce

Every closure implements one or more traits based on capture mode:

```rust
// FnOnce — consumes captured values (one call)
let name = String::from("Alice");
let greet = move || { drop(name); };
greet();
// greet(); // ❌ name consumed

// FnMut — mutably borrows captures (many calls, may mutate)
let mut count = 0;
let mut inc = || { count += 1; };
inc(); inc();

// Fn — immutably borrows captures (many calls, concurrent-safe)
let prefix = "Result";
let display = |x: i32| println!("{prefix}: {x}");
display(1); display(2);
```

**Hierarchy**: `Fn` ⊂ `FnMut` ⊂ `FnOnce`. Every `Fn` is also `FnMut` and `FnOnce`.

## Passing Closures

```rust
// Parameter — static dispatch (monomorphized, fastest)
fn apply_twice<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 { f(f(x)) }
// Or:
fn apply_twice_v2(f: impl Fn(i32) -> i32, x: i32) -> i32 { f(f(x)) }

// Parameter — dynamic dispatch
fn apply_dyn(f: &dyn Fn(i32) -> i32, x: i32) -> i32 { f(x) }

// Return — must box (closures have anonymous types)
fn make_adder(n: i32) -> Box<dyn Fn(i32) -> i32> { Box::new(move |x| x + n) }
// Or:
fn make_adder_v2(n: i32) -> impl Fn(i32) -> i32 { move |x| x + n }
```

### Choosing the Right Trait Bound

- Need to call concurrently? → `Fn`
- Default — most flexible, accepts both `Fn` and `FnMut` → **`FnMut`**
- Need to consume captures? → `FnOnce`

## Higher-Order API Design

```rust
/// Retry an operation with caller-controlled strategy
fn retry<T, E, F, S>(
    mut operation: F,
    mut should_retry: S,
    max_attempts: usize,
) -> Result<T, E>
where
    F: FnMut() -> Result<T, E>,
    S: FnMut(&E, usize) -> bool,
{
    for attempt in 1..=max_attempts {
        match operation() {
            Ok(val) => return Ok(val),
            Err(e) if attempt < max_attempts && should_retry(&e, attempt) => continue,
            Err(e) => return Err(e),
        }
    }
    unreachable!()
}

retry(|| connect_db(), |err, n| { eprintln!("attempt {n}: {err}"); true }, 3);
retry(|| http_get(url), |err, _| err.is_transient(), 5);
```

The caller controls the retry logic — no specialized variants needed.

## Combinator Chains — Idiomatic Rust

```rust
// C-style loop (imperative):
let mut result = Vec::new();
for x in &data {
    if x % 2 == 0 { result.push(x * x); }
}

// Idiomatic (functional chain):
let result: Vec<i32> = data.iter()
    .filter(|&&x| x % 2 == 0)
    .map(|&x| x * x)
    .collect();
// Same performance — iterators are lazy and inlined by LLVM.
```

### Common Combinators

| Combinator | What it does |
|---|---|
| `.map(f)` | Transform each element |
| `.filter(p)` | Keep elements where predicate true |
| `.filter_map(f)` | Map + filter (returns `Option`) |
| `.flat_map(f)` | Map then flatten |
| `.fold(init, f)` | Reduce to single value |
| `.any(p)` / `.all(p)` | Short-circuit boolean |
| `.find(p)` | First match |
| `.position(p)` | Index of first match |
| `.enumerate()` | Add index |
| `.zip(other)` | Pair with another iterator |
| `.take(n)` / `.skip(n)` | First/skip N |
| `.chain(other)` | Concatenate |
| `.peekable()` | Look ahead |
| `.collect()` | Gather into collection |

**Break chains at ~4 adapters** — use named intermediates for readability.

## The `with` Pattern — Bracketed Resource Access

Guarantee setup/teardown around an operation, regardless of how the caller's code exits:

```rust
pub fn with_pin_input<R>(&self, pin: u8, mut f: impl FnMut(&GpioPin<'_>) -> R) -> R {
    let prev = self.current_direction.get();
    self.set_direction(pin, Direction::In);
    let handle = GpioPin { pin_number: pin, _controller: self };
    let result = f(&handle);
    if let Some(dir) = prev { self.set_direction(pin, dir); }
    result
}

let level = gpio.with_pin_input(4, |pin| pin.read());
```

**Guarantees**:
- Direction always set before the caller's code runs
- Always restored after, even on early return / `?` / panic
- The `GpioPin` handle cannot escape — borrow checker enforces via lifetime
- Callers never see `Direction`, never call `set_direction` — impossible to misuse

### Where This Pattern Appears

| API | Setup | Callback | Teardown |
|---|---|---|---|
| `std::thread::scope` | Create scope | spawn threads | Join |
| `Mutex::lock` | Acquire | Use guard | Release on drop |
| `tempfile::tempdir` | Create tempdir | Use path | Delete on drop |
| `BufWriter::new` | Buffer writes | Write ops | Flush on drop |
| `with_pin_*` | Set direction | Use pin | Restore direction |

**`with` vs RAII**: Both guarantee cleanup. RAII when caller needs to hold resource across multiple statements. `with` when the operation is bracketed — one setup, one block, one teardown — and you don't want the caller to break the bracket.

## Anti-Patterns

- **Long `.and_then()` chains when `?` works.** If every closure is `|x| next_step(x)`, write it as `?` with named intermediates.
- **5+ deep iterator chains.** Use named intermediates or extract helpers.
- **Under-functionalizing — C-style loop with a stdlib equivalent.**

  ```rust
  // ❌ Loop
  let mut found = false;
  for item in &list { if item.is_expired() { found = true; break; } }

  // ✅ Combinator
  let found = list.iter().any(|item| item.is_expired());
  ```

- **Over-using `move` closures when a borrow suffices.** `move` is needed only when the closure outlives the captured scope.
- **Returning closures by value without `Box` or `impl Trait`.** Closures have anonymous types — you need one of the wrapper types.
- **Accepting `Fn` when `FnMut` works.** `Fn` is more restrictive; `FnMut` accepts more closures including all `Fn` ones.

## See Also

- [functional-style.md](./functional-style.md) — when to use combinators vs loops; combinator families
- [traits.md](./traits.md) — extension trait pattern (e.g., `Itertools`)
- [api-design.md](./api-design.md) — designing ergonomic higher-order APIs