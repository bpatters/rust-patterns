# Newtype and Type-State Patterns

Load this when: distinguishing similar primitives (e.g., UserId vs AccountId); designing compile-time state machines; building a builder that enforces required fields; taming generic parameter explosion; deciding whether to impl `Deref`.

## Newtype — Zero-Cost Type Safety

Wrap an existing type in a single-field tuple struct. Distinct type at zero runtime cost.

```rust
struct UserName(String);
struct Email(String);
struct Age(u32);
struct EmployeeId(u32);

fn create_user(name: UserName, email: Email, age: Age, id: EmployeeId) { }

// create_user(name, email, EmployeeId(42), Age(30));  // ❌ Compile error
```

Use whenever you'd otherwise reach for named primitives that could be confused.

### When NOT to impl Deref

`Deref` punches a hole through your abstraction — every method on the inner type becomes callable.

```rust
struct Email(String);
impl Deref for Email {
    type Target = str;
    fn deref(&self) -> &str { &self.0 }
}
// Now `.split_at()`, `.trim()`, `.replace()` all work on Email —
// and the "must contain @" invariant is bypassed the moment someone
// stores the resulting &str.
```

| Scenario | Impl Deref? |
|---|---|
| Smart-pointer wrappers (`Box<T>`, `Arc<T>`) | ✅ whole purpose |
| Transparent wrappers (`String → str`, `PathBuf → Path`) | ✅ IS-A relationship |
| Domain types with invariants | ❌ invariant leaks |
| Types where you want restricted API | ❌ leaks too much |
| Fake inheritance (`Manager` derefs to `Widget`) | ❌ explicitly discouraged |

**Prefer explicit delegation** when you want only some of the inner type's methods:

```rust
impl Email {
    pub fn as_str(&self) -> &str { &self.0 }
    pub fn domain(&self) -> &str { self.0.split('@').nth(1).unwrap_or("") }
    // .split_at, .trim, .replace NOT exposed
}
```

`DerefMut` doubles the risk — callers can mutate the inner value directly, bypassing constructor validation. Only when the inner type has no invariants.

## Type-State — Compile-Time Protocol Enforcement

Use the type system to enforce that operations happen in the correct order. **Invalid states become unrepresentable.**

```rust
struct Disconnected;
struct Connected;
struct Authenticated;

struct Connection<State> {
    address: String,
    _state: PhantomData<State>,
}

impl Connection<Disconnected> {
    fn new(address: &str) -> Self { /* ... */ }
    fn connect(self) -> Connection<Connected> { /* consumes self */ }
}

impl Connection<Connected> {
    fn authenticate(self, _token: &str) -> Connection<Authenticated> { /* ... */ }
}

impl Connection<Authenticated> {
    fn request(&self, path: &str) -> String { /* ... */ }
}

let conn = Connection::new("api.example.com");
conn.request("/data");                    // ❌ no method on Disconnected
let conn = conn.connect();
conn.request("/data");                    // ❌ no method on Connected
let conn = conn.authenticate("token");
conn.request("/data");                    // ✅ only works after authenticate
```

Each transition **consumes** `self` and returns a new type. You can't use the old state after transitioning. Zero runtime cost — `PhantomData` is zero-sized, states are erased.

**Case study — connection pool**: `pool.acquire()` returns `PooledConnection<Idle>`. Only `Idle` connections can `release()`. Forgetting to commit/rollback is a compile error.

## Builder with Type States

A builder that enforces required fields:

```rust
struct NeedsName;
struct NeedsPort;
struct Ready;

struct ServerConfig<State> {
    name: Option<String>,
    port: Option<u16>,
    max_connections: usize,
    _state: PhantomData<State>,
}

impl ServerConfig<NeedsName> {
    fn new() -> Self { /* ... */ }
    fn name(self, name: &str) -> ServerConfig<NeedsPort> { /* ... */ }
}

impl ServerConfig<NeedsPort> {
    fn port(self, port: u16) -> ServerConfig<Ready> { /* ... */ }
}

impl ServerConfig<Ready> {
    fn max_connections(mut self, n: usize) -> Self { self.max_connections = n; self }
    fn build(self) -> Server { /* ... */ }
}

let server = ServerConfig::new()
    .name("my-server")
    .port(8080)
    .max_connections(500)
    .build();

// ServerConfig::new().port(8080);       // ❌ no port() on NeedsName
// ServerConfig::new().name("x").build(); // ❌ no build() on NeedsPort
```

**Use when**: construction has required fields, ordering of operations matters, or you want compile-time enforcement that all required fields are set.

## Config Trait — Taming Generic Parameter Explosion

A struct with 3+ trait-constrained generics gets unwieldy. Bundle associated types into a single trait:

```rust
trait BoardConfig {
    type Spi: SpiBus;
    type Com: ComPort;
    type I3c: I3cBus;
}

struct DiagController<Cfg: BoardConfig> {
    spi: Cfg::Spi,
    com: Cfg::Com,
    i3c: Cfg::I3c,
}
// Adding a 4th bus: one new associated type + one new field. No downstream signature changes.
```

```rust
struct ProductionBoard;
impl BoardConfig for ProductionBoard {
    type Spi = PlatformSpi;
    type Com = UartCom;
    type I3c = LinuxI3c;
}

let ctrl = DiagController::<ProductionBoard>::new(/* ... */);
```

**Tests** swap the config: `struct TestBoard; impl BoardConfig for TestBoard { type Spi = MockSpi; /* ... */ }` — entire hardware layer mocked by changing one type parameter.

**Use when**:
- 3+ trait-constrained generics on a struct
- Need to swap entire platform/hardware layer (prod vs test)
- Component traits form a natural group (board, platform, runtime)

**Don't use when**:
- 1-2 generics (overkill, direct generics are clearer)
- Need runtime polymorphism (use `dyn Trait`)
- Open-ended plugin system (use `Any` + type maps)

**Battle-tested**: Substrate/Polkadot's frame system manages 20+ associated types through a single `Config` trait.

## Dual-Axis Typestate (Vendor × State)

When two independent axes vary — "who provides it" and "what state is it in":

```rust
struct Locked; struct Unlocked; struct ExtendedUnlocked;

trait HasRegAccess {}
impl HasRegAccess for Unlocked {}
impl HasRegAccess for ExtendedUnlocked {}

trait JtagVendor { /* raw ops */ }
trait JtagMemoryVendor: JtagVendor { /* extended ops */ }

struct Jtag<V, S = Locked> { vendor: V, _state: PhantomData<S> }

// Always starts Locked
impl<V: JtagVendor> Jtag<V, Locked> {
    fn unlock(mut self) -> Jtag<V, Unlocked> { /* ... */ }
}

// Register I/O — any vendor with HasRegAccess
impl<V: JtagVendor, S: HasRegAccess> Jtag<V, S> {
    fn read_reg(&self, addr: u32) -> u32 { /* ... */ }
}

// Memory I/O — only memory-capable vendors with HasMemAccess
impl<V: JtagMemoryVendor, S: HasMemAccess> Jtag<V, S> {
    fn read_memory(&self, addr: u64, buf: &mut [u8]) { /* ... */ }
}
```

**Why marker traits, not just concrete states**: `impl<V, S: HasRegAccess>` works for *any* state with reg access. Add `DebugHalted` tomorrow with one line: `impl HasRegAccess for DebugHalted {}`. Every register function works with it automatically.

**When**: two independent axes, some providers have strictly more capabilities (super-trait pattern), misusing state or capability is a safety bug.

**Three or more axes**: collapse the vendor axis into a Config trait: `Handle<Cfg, S>` instead of `Handle<V, S, D, T>`.

## Anti-Patterns

- **`Deref` for newtypes with invariants.** Leaks the API. Use explicit methods or `AsRef`.
- **Reaching for typestate when a simple enum is enough.** If you don't need compile-time enforcement of state transitions and just want runtime check + panic, an enum is simpler.
- **Mixing typestate with runtime polymorphism.** `dyn Connection<State>` doesn't work well — `State` is a type parameter. Use enum dispatch instead.
- **Too many state markers in one type.** If `State` has 5 variants and 20 transitions, the type signatures become unreadable. Consider restructuring.

## See Also

- [phantomdata.md](./phantomdata.md) — zero-sized markers that make typestate possible
- [traits.md](./traits.md) — marker traits, supertraits, blanket impls
- [api-design.md](./api-design.md) — `TryFrom` for parse-don't-validate (a related way to encode invariants)
- [error-handling.md](./error-handling.md) — newtype-wrapping error types