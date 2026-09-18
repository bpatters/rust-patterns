# Functional vs Imperative Style

Load this when: deciding iterator chain vs loop; writing one-liners from `if let` boilerplate; choosing between `.map()`, `.and_then()`, `.unwrap_or_else()`; folding vs accumulating.

**Core principle**: Functional style shines when transforming data through a pipeline. Imperative style shines when managing state transitions with side effects. Most code has both — the skill is knowing where the boundary falls.

## Option Combinators

`Option<T>` is a one-element-or-empty collection. Combinators are collection operations.

```rust
// Common rewrites:
opt.unwrap_or(default)         // if let Some(x) = opt { x } else { default }
opt.unwrap_or_else(|| lazy())  // same, but default is lazy
opt.map(f)                     // match opt { Some(x) => Some(f(x)), None => None }
opt.and_then(f)                // match opt { Some(x) => f(x), None => None } (flatmap)
opt.filter(|x| pred(x))        // match with predicate guard
opt.zip(other)                 // both-or-neither pairing
opt.or(fallback)               // first available
opt.map_or(default, f)         // transform or default — one-liner
opt.map_or_else(def_fn, f)     // both sides are closures
opt?                           // propagate absence upward
```

### When `if let` IS Better

Combinators lose when:
- Multiple statements in the `Some` branch
- The control flow IS the point (`if let Some(conn) = pool.try_get() { /* use it */ } else { /* log, retry, alert */ }` — branches are genuinely different)
- Side effects dominate (different I/O with different error handling)

**Rule**: If both branches produce the same type and bodies are short expressions, use a combinator. If branches do fundamentally different things, use `if let` or `match`.

**Let chains (edition 2024, Rust 1.88+)**: flatten nested `if let` with `&&`. Bindings from earlier `let`s are in scope for later conditions. Requires `edition = "2024"` — it will not compile on 2021.

```rust
if let Some(user) = session.user()
    && let Some(email) = user.email.as_ref()
    && email.ends_with("@example.com")
{
    send(email);
}
```

## Result Combinators

```rust
res.map(f)                      // transform success
res.map_err(f)                  // transform error
res.and_then(f)                 // chain fallible ops
res.unwrap_or_else(|e| def())   // recover from error
res.ok()                        // discard error → Option<T>
res?                            // propagate errors upward
```

## Bool Combinators

```rust
let label = is_admin.then_some("ADMIN");
let perms = is_admin.then(|| compute_admin_permissions());

let tags: Vec<&str> = [
    user.is_admin.then_some("admin"),
    user.is_verified.then_some("verified"),
    (user.score > 100).then_some("power-user"),
].into_iter().flatten().collect();
```

## Iterator Chains vs Loops

### When Iterators Win

**Data pipelines** — transforming a collection through stages:

```rust
let results: Vec<_> = inventory.iter()
    .filter(|item| item.category == Category::Server)
    .filter_map(|item| item.last_temperature().map(|t| (item.id, t)))
    .filter(|(_, temp)| *temp > 80.0)
    .collect();
```

The functional version wins because:
- Each filter is independently readable
- No `mut` — data flows in one direction
- Add/remove/reorder stages without restructuring
- LLVM inlines iterator adapters to the same machine code as loops

**Aggregation** — computing a single value:

```rust
let total: f64 = fleet.iter().map(|s| s.power_draw()).sum();
let avg_temp = fleet.iter().map(|s| s.max_temperature()).fold(f64::NEG_INFINITY, f64::max);
```

### When Loops Win

**Early exit with complex state** — though often `find`/`any` work too.

**Building multiple outputs simultaneously**:

```rust
let mut warnings = Vec::new();
let mut errors = Vec::new();
let mut stats = Stats::default();

for event in log_stream {
    match event.severity {
        Severity::Warn => { warnings.push(event.clone()); stats.warn_count += 1; }
        Severity::Error => {
            errors.push(event.clone()); stats.error_count += 1;
            if event.is_critical() { alert_oncall(&event); }
        }
        _ => stats.other_count += 1,
    }
}

// Functional version — awkward, mutates anyway
let (warnings, errors, stats) = log_stream.iter().fold(
    (Vec::new(), Vec::new(), Stats::default()),
    |(mut w, mut e, mut s), event| {
        match event.severity {
            Severity::Warn => { w.push(event.clone()); s.warn_count += 1; }
            Severity::Error => {
                e.push(event.clone()); s.error_count += 1;
                if event.is_critical() { alert_oncall(event); }
            }
            _ => s.other_count += 1,
        }
        (w, e, s)
    },
);
```

**State machines with I/O** — the loop IS the algorithm.

### Decision Flowchart

```text
What are you doing?
├── Transforming a collection into another → iterator chain
├── Computing a single value from a collection
│    ├── Sum, count, min, max → .sum(), .count(), .min(), .max()
│    └── Custom accumulation → .fold() (no mutation), or loop (mutation)
├── Multiple outputs from one pass → for loop
├── State machine with I/O or side effects → for loop
└── One Option/Result transform + default → combinators
```

## The `?` Operator

`?` is `.and_then()` + early return. Use when you can early-return.

```rust
fn load_config() -> Result<Config, Error> {
    let contents = read_file("config.toml")?;
    let table = parse_toml(&contents)?;
    let valid = validate_config(table)?;
    Config::from_validated(valid)
}
```

**Anti-pattern**: long `.and_then()` chains when `?` is available. If every closure is `|x| next_step(x)`, you've reinvented `?`.

**When `.and_then()` IS better**:

```rust
// Building an Option, not propagating from a function
let port: Option<u16> = config.get("port")
    .and_then(|v| v.parse::<u16>().ok())
    .filter(|&p| p > 0 && p < 65535);
```

## Collection Building

### Collecting into a Result

```rust
let numbers: Vec<i64> = input_strings.iter()
    .map(|s| s.parse::<i64>().map_err(|_| Error::BadInput(s.clone())))
    .collect::<Result<_, _>>()?;  // short-circuits on first Err
```

### Collecting into Other Types

```rust
let index: HashMap<_, _> = fleet.into_iter()
    .map(|s| (s.id.clone(), s)).collect();

let csv = fields.join(",");
// Or: let csv: String = fields.iter().map(|f| format!("\"{f}\"")).collect::<Vec<_>>().join(",");
```

### When the Loop Wins — In-Place Update

```rust
for server in &mut fleet {
    if server.needs_refresh() { server.refresh_telemetry()?; }
}
// Functional: .iter_mut().for_each(|s| { ... }) is just a loop with extra syntax.
```

## Pattern Matching as Function Dispatch

```rust
// if/else chain — silent fallthrough on missing case
fn status_message(code: StatusCode) -> &'static str {
    if code == StatusCode::OK { "Success" }
    else if code == StatusCode::NOT_FOUND { "Not found" }
    // ...
}

// match — exhaustive; adding a variant becomes a compile error elsewhere
fn status_message(code: StatusCode) -> &'static str {
    match code {
        StatusCode::OK => "Success",
        StatusCode::NOT_FOUND => "Not found",
        StatusCode::INTERNAL => "Server error",
        _ => "Unknown",
    }
}
```

Each match arm is an expression returning the same type — the arms are essentially a function table indexed by the enum variant.

## Scoped Mutability — Imperative Inside, Functional Outside

Confine mutation to a block, bind result immutably:

```rust
let samples = {
    let mut buf = Vec::with_capacity(10);
    while buf.len() < 10 {
        let reading: f64 = random();
        buf.push(reading);
        if random::<u8>() % 3 == 0 { break; }  // random early exit
    }
    buf
};
// samples is immutable; compiler rejects later samples.push(...)
```

Use when:
- Sort-then-freeze (both return `()`, no chainable output)
- Stateful termination (stop on condition unrelated to data)
- Multi-step struct population from different sources

## Performance: They Compile to the Same Code

```rust
// These produce identical assembly on release builds:
let sum: i64 = (0..1000).filter(|n| n % 2 == 0).map(|n| n * n).sum();

let mut sum: i64 = 0;
for n in 0..1000 { if n % 2 == 0 { sum += n * n; } }
```

**Exception**: `.collect()` allocates. If you're chaining `.map().collect().iter().map().collect()`, you're paying for allocations the loop avoids.

## Anti-Patterns

- **Over-functionalizing — 5+ deep chains.** Use named intermediates or extract helpers.
- **Under-functionalizing — C-style loop with a stdlib equivalent** (`any`/`all`/`find`/`sum`).
- **Long `.and_then()` chains when `?` works.**
- **Using `.collect::<Vec<_>>()` for one element.** Use `.next()`.
- **Mixing pure transforms and I/O in one chain** — readers can't tell which calls fail or have side effects. Separate the pure pipeline from the I/O bookends.
- **Trying to force-fold multiple outputs** — a `fold` with three `mut` accumulators in a tuple is just a loop rewritten with worse syntax.

## See Also

- [closures.md](./closures.md) — combinators, `Fn`/`FnMut`/`FnOnce`, `with` pattern
- [error-handling.md](./error-handling.md) — `?` operator, Result combinators
- [smart-pointers.md](./smart-pointers.md) — `Cow` for clone-on-write pipelines