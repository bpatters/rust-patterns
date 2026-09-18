# Testing and Benchmarking Patterns

Load this when: writing unit/integration/doc tests; setting up property-based tests; benchmarking performance-critical code; mocking dependencies; organizing test fixtures.

## Three Test Tiers

```rust
// Unit tests — in same file as code
pub fn factorial(n: u64) -> u64 { (1..=n).product() }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_factorial_zero() { assert_eq!(factorial(0), 1); }

    #[test]
    #[cfg(debug_assertions)] // overflow checks are debug-only unless overflow-checks = true
    #[should_panic(expected = "overflow")]
    fn test_factorial_overflow() {
        // Release wraps silently — this test is debug-only. Prefer checked_mul
        // and assert None if the test must pass under cargo test --release.
        factorial(100);
    }

    #[test]
    fn test_with_result() -> Result<(), Box<dyn std::error::Error>> {
        let v: u64 = "42".parse()?;
        assert_eq!(v, 42);
        Ok(())  // ? works inside test functions returning Result
    }
}

// Integration tests — tests/ directory, public API only
// tests/integration_test.rs
use my_crate::factorial;
#[test]
fn from_outside() { assert_eq!(factorial(10), 3_628_800); }

// Doc tests — compiled and run by cargo test
/// Computes factorial of `n`.
///
/// # Examples
/// ```
/// use my_crate::factorial;
/// assert_eq!(factorial(5), 120);
/// ```
///
/// Overflow panics only in debug (or with `overflow-checks = true`).
/// Do not use a `should_panic` doc-test for debug-only panics — it fails
/// under `cargo test --release`.
pub fn factorial(n: u64) -> u64 { (1..=n).product() }
```

Run: `cargo test` (all), `cargo test --doc`, `cargo test --test integration_test`.

## Test Fixtures and Setup

```rust
#[cfg(test)]
mod tests {
    fn setup_database() -> TestDb { /* ... */ }

    #[test]
    fn test_user_creation() {
        let db = setup_database();
        // ...
    }

    // Cargo.toml: tempfile = "3"
    #[test]
    fn test_file_ops() {
        let dir = tempfile::TempDir::new().unwrap();
        std::fs::write(dir.path().join("test.txt"), "hello").unwrap();
        // dir dropped here → temp directory cleaned up
    }
}
```

## Property-Based Testing (proptest)

Test **properties** that should always hold, not specific values:

```rust
use proptest::prelude::*;
// Cargo.toml: proptest = "1"

fn reverse(v: &[i32]) -> Vec<i32> { v.iter().rev().cloned().collect() }

proptest! {
    #[test]
    fn reverse_twice_is_identity(v in prop::collection::vec(any::<i32>(), 0..100)) {
        assert_eq!(reverse(&reverse(&v)), v);
    }

    #[test]
    fn reverse_preserves_length(v in prop::collection::vec(any::<i32>(), 0..100)) {
        assert_eq!(reverse(&v).len(), v.len());
    }

    #[test]
    fn parse_roundtrip(x in any::<f64>().prop_filter("finite", |x| x.is_finite())) {
        let s = format!("{x}");
        let parsed: f64 = s.parse().unwrap();
        prop_assert_eq!(x, parsed); // Display of finite f64 is designed to round-trip
    }
}
```

proptest generates hundreds of random inputs and **shrinks** failures to the minimal reproducing case.

**When to use**: large input space, want confidence for edge cases you didn't think of, parsing/serialization invariants, commutation laws.

## Benchmarking with Criterion

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.8", features = ["html_reports"] }

[[bench]]
name = "my_benchmarks"
harness = false   # required — otherwise libtest is the entry point
```

```rust
// benches/my_benchmarks.rs
use criterion::{criterion_group, criterion_main, Criterion, black_box};

fn fibonacci(n: u64) -> u64 {
    match n { 0 | 1 => n, _ => fibonacci(n - 1) + fibonacci(n - 2) }
}

fn bench_fibonacci(c: &mut Criterion) {
    c.bench_function("fibonacci 20", |b| b.iter(|| fibonacci(black_box(20))));

    let mut group = c.benchmark_group("fibonacci_compare");
    for size in [10, 15, 20, 25] {
        group.bench_with_input(
            criterion::BenchmarkId::from_parameter(size),
            &size,
            |b, &size| b.iter(|| fibonacci(black_box(size))),
        );
    }
    group.finish();
}

criterion_group!(benches, bench_fibonacci);
criterion_main!(benches);
```

```bash
cargo bench              # all
cargo bench -- fib_20    # filtered
# HTML reports in target/criterion/
```

`black_box` prevents the compiler from optimizing away the inputs.

## Mocking Without Frameworks

Rust's trait system provides natural dependency injection — no mocking framework required:

```rust
trait Clock { fn now(&self) -> std::time::Instant; }
trait HttpClient { fn get(&self, url: &str) -> Result<String, String>; }

struct RealClock;
impl Clock for RealClock { fn now(&self) -> std::time::Instant { std::time::Instant::now() } }

struct CacheService<C: Clock, H: HttpClient> {
    clock: C, client: H, ttl: std::time::Duration,
}

impl<C: Clock, H: HttpClient> CacheService<C, H> {
    fn fetch(&self, url: &str) -> Result<String, String> { self.client.get(url) }
}

// Test with mocks — no framework needed
#[cfg(test)]
mod tests {
    use super::*;

    struct MockClock { fixed_time: std::time::Instant }
    impl Clock for MockClock { fn now(&self) -> std::time::Instant { self.fixed_time } }

    struct MockHttpClient { response: String }
    impl HttpClient for MockHttpClient {
        fn get(&self, _url: &str) -> Result<String, String> { Ok(self.response.clone()) }
    }

    #[test]
    fn test_cache_service() {
        let service = CacheService {
            clock: MockClock { fixed_time: std::time::Instant::now() },
            client: MockHttpClient { response: "cached data".into() },
            ttl: std::time::Duration::from_secs(300),
        };
        assert_eq!(service.fetch("http://example.com").unwrap(), "cached data");
    }
}
```

**Test philosophy**: Prefer real dependencies in integration tests, trait-based mocks in unit tests. Avoid mocking frameworks unless the dependency graph is genuinely complex.

## Anti-Patterns

- **Tests that depend on each other.** Each test must set up its own state — no shared mutable test fixtures.
- **Tests that depend on real network, real filesystem at hardcoded paths.** Use mocks or `tempfile::TempDir`.
- **`#[should_panic]` without `expected = "..."`** — makes the test pass for any panic. Be specific.
- **Tests that pass in debug but fail in release (or vice versa).** Watch out for overflow checks enabled only in debug.
- **Benchmarking without `black_box`.** The compiler optimizes away the work you're trying to measure.
- **Reaching for `mockall`/`mockito` when a small handwritten trait mock works.** Mocking frameworks add complexity.
- **Tests that don't run any assertions.** `#[test] fn test_x() { /* ... */ }` is not a test.
- **Testing private internals from `tests/`.** Integration tests see only the public API. Unit tests in `#[cfg(test)] mod tests` in the same module **are** the place to test private helpers.
- **`unwrap()` in tests is fine** — but consider `assert_eq!` for richer failure messages.

## See Also

- [macros.md](./macros.md) — testing macro-generated code
- [api-design.md](./api-design.md) — module layout affects test organization
- [error-handling.md](./error-handling.md) — testing error paths
- [newtype-typestate.md](./newtype-typestate.md) — type-state enables test fixtures with different capabilities