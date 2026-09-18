# Macros — Code That Writes Code

Load this when: reducing boilerplate that generics/traits can't handle; creating DSLs (`html!`, `sql!`, `vec!`); writing custom derive macros; deciding macro vs function vs generic.

## Declarative Macros (`macro_rules!`)

Match patterns on syntax and expand to code at compile time:

```rust
macro_rules! hashmap {
    ( $( $key:expr => $value:expr ),* $(,)? ) => {
        {
            let mut map = std::collections::HashMap::new();
            $( map.insert($key, $value); )*
            map
        }
    };
}

let scores = hashmap! {
    "Alice" => 95,
    "Bob" => 87,
};
```

### Fragment Types

| Fragment | Matches | Example |
|---|---|---|
| `$x:expr` | Any expression | `42`, `a + b`, `foo()` |
| `$x:ty` | A type | `i32`, `Vec<String>` |
| `$x:ident` | An identifier | `my_var`, `Config` |
| `$x:pat` | A pattern | `Some(x)`, `_` |
| `$x:stmt` | A statement | `let x = 5;` |
| `$x:tt` | A single token tree | Anything (most flexible) |
| `$x:literal` | A literal value | `42`, `"hello"`, `true` |

**Repetition**: `$( ... ),*` = zero or more, comma-separated. `$( ... ),+` = one or more.

### Generate Test Functions

```rust
macro_rules! test_cases {
    ( $( $name:ident: $input:expr => $expected:expr ),* $(,)? ) => {
        $(
            #[test]
            fn $name() {
                assert_eq!(process($input), $expected);
            }
        )*
    };
}

test_cases! {
    test_empty: "" => "",
    test_hello: "hello" => "HELLO",
}
```

## When (Not) to Use Macros

### Use When

- Reducing boilerplate that traits/generics can't handle (variadic args, DRY test gen)
- Creating DSLs (`html!`, `sql!`, `vec!`, custom mini-languages)
- Conditional code generation (`cfg!`, `compile_error!`)

### Don't Use When

- A function or generic would work — macros are harder to debug, autocomplete doesn't help
- You need type checking inside the macro — macros operate on tokens, not types
- The pattern is used once or twice — not worth the abstraction cost

```rust
// ❌ Unnecessary macro — a function works fine
macro_rules! double { ($x:expr) => { $x * 2 }; }

// ✅ Just use a function
fn double(x: i32) -> i32 { x * 2 }

// ✅ Good macro use — variadic, can't be a function
macro_rules! println { ($($arg:tt)*) => { /* format + args */ }; }
```

## Procedural Macros

Procedural macros are Rust functions that transform token streams. Require a separate crate with `proc-macro = true`.

### Three Types

```rust
// 1. Derive macros — #[derive(MyTrait)]
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Config { name: String, port: u16 }

// 2. Attribute macros — #[my_attribute]
#[route(GET, "/api/users")]
async fn list_users() -> Json<Vec<User>> { /* ... */ }

// 3. Function-like macros — my_macro!(...)
let query = sql!(SELECT * FROM users WHERE id = ?);
```

### Commonly Used Derives

| Derive | Crate | What it generates |
|---|---|---|
| `Debug` | std | `fmt::Debug` impl |
| `Clone`, `Copy` | std | Value duplication |
| `PartialEq`, `Eq` | std | Equality comparison |
| `Hash` | std | Hashing for HashMap keys |
| `Serialize`, `Deserialize` | serde | JSON/YAML/etc. encoding |
| `Error` | thiserror | `std::error::Error` + `Display` |
| `Parser` | clap | CLI argument parsing |
| `Builder` | `bon` (or `typed-builder` / `derive_builder`) | Builder pattern |

**Practical advice**: Use derive macros liberally — they eliminate error-prone boilerplate. Use existing crates (`serde`, `thiserror`, `clap`) before building custom ones.

## Macro Hygiene and `$crate`

**Hygiene** means identifiers inside a macro don't collide with caller's scope. `macro_rules!` is partially hygienic.

**`$crate`**: When writing macros in a library, use `$crate` to refer to your own crate — resolves correctly even if the user renamed your crate in Cargo.toml.

```rust
// In my_diagnostics crate:
#[macro_export]
macro_rules! diag_log {
    ($($arg:tt)*) => {
        // ✅ $crate always resolves to my_diagnostics
        $crate::log_result(&format!($($arg)*))
    };
}
```

**Rule**: Always use `$crate::` in `#[macro_export]` macros. Never use your crate's name directly.

`#[macro_export]` also places the macro at the **crate root**, ignoring the module path — callers write `crate::diag_log!`, not `crate::macros::diag_log!`.

## Recursive Macros and `tt` Munching

Recursive macros process input one token at a time:

```rust
// Count expressions passed to the macro
macro_rules! count {
    () => { 0usize };
    ($head:expr $(, $tail:expr)* $(,)?) => {
        1usize + count!($($tail),*)
    };
}

const N: usize = count!(1, 2, 3);
assert_eq!(N, 3);

// Build a heterogeneous tuple from a list of expressions:
macro_rules! tuple_from {
    ($single:expr $(,)?) => { ($single,) };
    ($head:expr, $($tail:expr),+ $(,)?) => {
        ($head, tuple_from!($($tail),+))
    };
}

let t = tuple_from!(1, "hello", 3.14, true);
// Expands to: (1, ("hello", (3.14, (true,))))
```

### Fragment Subtleties

| Fragment | Gotcha |
|---|---|
| `$x:expr` | Greedy — `1 + 2` is ONE expression |
| `$x:ty` | Greedy — `Vec<String>` is one type; can't be followed by `+` or `<` |
| `$x:tt` | Matches exactly ONE token tree — most flexible, least checked |
| `$x:ident` | Only plain identifiers — not paths like `std::io` |
| `$x:pat` | In Rust 2021, matches `A \| B` patterns; use `$x:pat_param` for single patterns |

**Use `tt` when** you need to forward tokens to another macro without the parser constraining them. `$($args:tt)*` is the "accept everything" pattern (used by `println!`, `format!`, `vec!`).

## Writing a Derive Macro

Derive macros live in a separate crate:

```toml
# my_derive/Cargo.toml
[lib]
proc-macro = true

[dependencies]
syn = { version = "3", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

```rust
// my_derive/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    let name_str = name.to_string();

    let fields = match &input.data {
        syn::Data::Struct(data) => data.fields.iter()
            .filter_map(|f| f.ident.as_ref())
            .map(|id| id.to_string())
            .collect::<Vec<_>>(),
        _ => vec![],
    };
    let field_list = fields.join(", ");

    let expanded = quote! {
        impl #name {
            pub fn describe() -> String {
                format!("{} {{ {} }}", #name_str, #field_list)
            }
        }
    };
    TokenStream::from(expanded)
}
```

**Workflow**: `TokenStream` → `syn::parse` (AST) → inspect/transform → `quote!` (generate tokens) → `TokenStream`.

| Crate | Role | Key types |
|---|---|---|
| `proc-macro` | Compiler interface | `TokenStream` |
| `syn` | Parse Rust source into AST | `DeriveInput`, `ItemFn`, `Type` |
| `quote` | Generate Rust tokens from templates | `quote!{}`, `#variable` |
| `proc-macro2` | Bridge between syn/quote and proc-macro | `TokenStream`, `Span` |

**Practical tip**: Study the source of `thiserror` or `derive_more` before writing your own. Use `cargo expand` (via `cargo-expand`) to see what any macro expands to — invaluable for debugging.

## Anti-Patterns

- **Macros when a function works.** Functions are type-checked, debuggable, and IDE-friendly.
- **Macros when a generic would work.** Generics preserve type information; macros erase it.
- **`#[macro_export]` without `$crate`.** Breaks for users who rename your crate.
- **Deeply recursive macros without a base case.** Will recurse forever if no base case matches.
- **Hygiene-leaking identifiers.** Use `$crate` and rely on Rust's partial hygiene.
- **Generating `unsafe` from a derive without proper SAFETY justification.** Every `unsafe` site needs human review.
- **Building a custom proc macro when an existing crate solves it.** Thousands of derives exist — search first.

## See Also

- [traits.md](./traits.md) — when traits/generics beat macros
- [testing.md](./testing.md) — testing macro-generated code
- [api-design.md](./api-design.md) — when macros are appropriate in public APIs