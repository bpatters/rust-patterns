# Crate Architecture and API Design

Load this when: designing a public crate API; choosing parameter types (`Into`/`AsRef`/`Cow`); applying parse-don't-validate with `TryFrom`; setting up feature flags; organizing a workspace; sealing traits.

## Module Layout Conventions

```
my_crate/
├── Cargo.toml
├── src/
│   ├── lib.rs          # Crate root — re-exports and public API
│   ├── config.rs       # Feature module
│   ├── parser/         # Complex module with sub-modules
│   │   ├── mod.rs      # or parser.rs at parent level (Rust 2018+)
│   │   ├── lexer.rs
│   │   └── ast.rs
│   ├── error.rs        # Error types
│   └── utils.rs        # Internal helpers (pub(crate))
├── tests/              # Integration tests
├── benches/            # Benchmarks
└── examples/           # cargo run --example basic
```

```rust
// lib.rs — curate your public API with re-exports
mod config;
mod error;
mod parser;
mod utils;

pub use config::Config;
pub use error::Error;
pub use parser::Parser;

// Users write:
// use my_crate::Config;
// NOT: use my_crate::config::Config;
```

### Visibility Modifiers

| Modifier | Visible to |
|---|---|
| `pub` | Everyone |
| `pub(crate)` | This crate only |
| `pub(super)` | Parent module |
| `pub(in path)` | Specific ancestor |
| (none) | Current module + children |

## Public API Design Checklist

1. **Accept references, return owned** — `fn process(input: &str) -> String`
2. **Use `impl Trait` for parameters** — `fn read(r: impl Read)` over `fn read<R: Read>(r: R)` for cleaner signatures
3. **Return `Result`, not `panic!`** — let callers decide how to handle errors
4. **Implement standard traits** — `Debug`, `Display`, `Clone`, `Default`, `From`/`Into`, `Eq`/`Hash` as appropriate
5. **Make invalid states unrepresentable** — use type-state and newtypes
6. **Follow the builder pattern for complex configuration** — with type-state if fields are required
7. **Seal traits you don't want users to implement** — `pub trait Sealed: private::Sealed {}`
8. **Mark types and functions `#[must_use]`** — prevents silent discard of important values, `Result`s, guards
9. **Mark public enums `#[non_exhaustive]`** — adding variants is not a breaking change
10. **Implement `FromStr` for types parsed from text** — enables `.parse()`, integrates with `clap`
11. **Public `-> impl Trait` in edition 2024 captures all in-scope lifetimes.** If that is too tight, add `+ use<...>` (see [traits.md](./traits.md)). Public `async fn` in traits should promise `Send` (or ship a `trait-variant` pair) if callers `tokio::spawn`

### Sealed Trait Pattern

```rust
mod private { pub trait Sealed {} }

pub trait DatabaseDriver: private::Sealed {
    fn connect(&self, url: &str) -> Connection;
}

pub struct PostgresDriver;
impl private::Sealed for PostgresDriver {}
impl DatabaseDriver for PostgresDriver { /* ... */ }
// Only types in THIS crate can implement DatabaseDriver.
```

### `#[non_exhaustive]` for Public Enums

```rust
#[non_exhaustive]
pub enum DiagError {
    Timeout,
    HardwareFault,
    // Adding a new variant in v0.2 is NOT a semver break — downstream
    // match statements must have a wildcard arm.
}
```

## Ergonomic Parameter Patterns — `impl Into`, `AsRef`, `Cow`

Be liberal in what you accept so callers don't need `.to_string()`, `&*s`, or `.as_ref()` at every call site.

### `impl Into<T>` — Accept Anything Convertible

```rust
// ❌ Friction — callers must convert
fn connect(host: String, port: u16) -> Connection { /* ... */ }
connect("localhost".to_string(), 5432);   // annoying .to_string()
connect(hostname.clone(), 5432);          // unnecessary clone

// ✅ Ergonomic — accept anything that converts to String
fn connect(host: impl Into<String>, port: u16) -> Connection {
    let host = host.into();
    // ...
}
connect("localhost", 5432);    // &str — zero friction
connect(hostname, 5432);       // String — moved, no clone
```

### `AsRef<T>` — Borrow as a Reference

When you only need to read:

```rust
fn file_exists(path: impl AsRef<Path>) -> bool {
    path.as_ref().exists()
}
file_exists("/tmp/test.txt");                     // &str ✅
file_exists(String::from("/tmp/test.txt"));       // String ✅
file_exists(Path::new("/tmp/test.txt"));          // &Path ✅
file_exists(PathBuf::from("/tmp/test.txt"));      // PathBuf ✅
```

### `Cow<T>` — Clone on Write

Delays allocation until mutation is needed:

```rust
fn normalize_message(msg: &str) -> Cow<'_, str> {
    if msg.contains('\t') || msg.contains('\r') {
        Cow::Owned(msg.replace('\t', "    ").replace('\r', ""))
    } else {
        Cow::Borrowed(msg)  // no allocation
    }
}
```

### Decision

```text
Need ownership inside the function?
├── YES → impl Into<T>
└── NO  → Only read it?
     ├── YES → impl AsRef<T> or &T
     └── MAYBE (might modify sometimes) → Cow<'_, T>
```

| Pattern | Ownership | Allocation | When |
|---|---|---|---|
| `&str` | Borrowed | Never | Simple string params |
| `impl AsRef<str>` | Borrowed | Never | Accept String/&str — read only |
| `impl Into<String>` | Owned | On conversion | Accept &str/String — will store/own |
| `Cow<'_, str>` | Either | Only if modified | Processing that usually doesn't modify |

**`Borrow<T>` vs `AsRef<T>`**: both provide `&T`, but `Borrow<T>` guarantees `Eq`/`Ord`/`Hash` are consistent between original and borrowed. Use `Borrow` for lookup keys (HashMap), `AsRef` for general "give me a reference".

## Parse Don't Validate — `TryFrom` and Validated Types

Don't check data and pass around the raw unchecked form. Parse it into a type that can only exist if the data is valid.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Port(u16);

#[derive(Debug)]
pub enum PortError { Zero, InvalidFormat }

impl std::fmt::Display for PortError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            PortError::Zero => write!(f, "port must be non-zero"),
            PortError::InvalidFormat => write!(f, "invalid port format"),
        }
    }
}
impl std::error::Error for PortError {}

impl TryFrom<u16> for Port {
    type Error = PortError;
    fn try_from(value: u16) -> Result<Self, Self::Error> {
        // Destination port: 0 is invalid. Bind-to-ephemeral uses port 0
        // (`TcpListener::bind(("127.0.0.1", 0))`) — that is a different type
        // (`BindPort`), not this one. Keep the field private so Port(0) cannot be forged.
        if value == 0 { Err(PortError::Zero) } else { Ok(Port(value)) }
    }
}

impl Port {
    pub fn get(&self) -> u16 { self.0 }
}

fn start_server(port: Port) {  // No validation needed — Port is valid by construction.
    println!("Listening on port {}", port.get());
}

let port = Port::try_from(8080)?;  // Validate once at the boundary
start_server(port);                  // No re-validation downstream
```

### Implementing `FromStr` for Text Input

```rust
use std::str::FromStr;
impl FromStr for Port {
    type Err = PortError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let n: u16 = s.parse().map_err(|_| PortError::InvalidFormat)?;
        Port::try_from(n)
    }
}

// Now works with .parse() and clap:
let port: Port = "8080".parse()?;
// #[derive(Parser)] struct Args { #[arg(short, long)] port: Port }
```

### Validate vs Parse

| Approach | Data checked? | Compiler enforces validity? | Re-validation needed? |
|---|---|---|---|
| Runtime checks (`if`/`assert`) | ✅ | ❌ | Every function boundary |
| Validated newtype + `TryFrom` | ✅ | ✅ | Never — type is proof |

**Rule**: parse at the boundary, use validated types everywhere inside.

## Feature Flags and Conditional Compilation

```toml
# Cargo.toml
[features]
default = ["json"]
json = ["dep:serde_json"]      # dep: avoids an implicit feature named serde_json
xml = ["dep:quick-xml"]
full = ["json", "xml"]

[dependencies]
serde = "1"
serde_json = { version = "1", optional = true }
quick-xml = { version = "0.42", optional = true, features = ["serialize"] }
```

```rust
#[cfg(feature = "json")]
pub fn to_json<T: serde::Serialize>(value: &T) -> String {
    serde_json::to_string(value).unwrap()
}

#[cfg(feature = "xml")]
pub fn to_xml<T: serde::Serialize>(value: &T) -> String {
    quick_xml::se::to_string(value).unwrap()
}

#[cfg(not(any(feature = "json", feature = "xml")))]
compile_error!("At least one format feature must be enabled");
```

**Conditional attribute** — apply attribute only when condition is true:

```rust
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
#[derive(Debug, Clone)]
pub struct DiagResult { /* ... */ }

#[cfg_attr(test, derive(PartialEq))]
pub struct LargeStruct { /* ... */ }
```

**Best practices**:
- Keep `default` features minimal
- Use `dep:` syntax for optional dependencies
- Document features in README

## Workspace Organization

```toml
# Root Cargo.toml
[workspace]
members = ["core", "parser", "server", "client", "cli"]
resolver = "3"            # edition-2024 default; set explicitly in mixed workspaces

[workspace.package]
edition = "2024"
rust-version = "1.85"     # first release that can compile edition 2024

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["rt", "macros", "net", "time", "sync"] }  # not "full" here — leaks into every member and downstream
thiserror = "2"
tracing = "0.1"

# In each member Cargo.toml:
# [package]
# edition.workspace = true
# [dependencies]
# serde = { workspace = true }
```

**Benefits**: single `Cargo.lock`, shared build cache, `cargo test --workspace`, clean dependency boundaries.

## `.cargo/config.toml`

Project-level Cargo configuration:

```toml
# Do NOT set [build] target here unless every developer cross-compiles —
# a default target of musl breaks local macOS/Windows `cargo test`.

[alias]
xt = "test --workspace --release"
ci = "clippy --workspace -- -D warnings"
```

Cross-compile in CI or with `--target`, not as a repo-wide default.

## Anti-Patterns

- **Stringly-typed APIs.** `fn connect(host: &str, port: &str)` — use validated newtypes.
- **Boolean flags that should be enums.** `search(needle: &str, case_sensitive: bool)` → `enum CaseSensitivity`.
- **Wide module trees exposed publicly.** Re-export at crate root so users have one import path.
- **Default features that pull in heavy deps.** Keep `default` minimal.
- **`tokio` `features = ["full"]` in `[workspace.dependencies]`.** Members inherit it and force it on dependents. Keep workspace tokio minimal; enable `full` only in binaries (see [async.md](./async.md)).
- **Public types without `Debug`.** Add `#[derive(Debug)]` to anything users might want to inspect.
- **Returning `Result<T, String>` in public APIs.** Use a real error type.
- **Not implementing `From`/`Into`/`AsRef`/`Borrow`** on obvious types — saves users from manual conversions.
- **Implementing `Deref` to make a newtype "feel like" the inner type** when invariants are involved. Use explicit methods.

## See Also

- [newtype-typestate.md](./newtype-typestate.md) — type-state for builder APIs
- [error-handling.md](./error-handling.md) — designing error types for public APIs
- [traits.md](./traits.md) — sealed traits, extension traits, dyn compatibility