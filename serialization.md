# Serialization, Zero-Copy, and Binary Data

Load this when: designing serde-based structs; choosing JSON/TOML/binary format; parsing binary protocols; zero-copy deserialization; hardware register layouts; FFI data exchange.

## serde Fundamentals

`serde` separates **data model** (your structs) from **format** (JSON, TOML, binary).

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct ServerConfig {
    name: String,
    port: u16,
    #[serde(default)]
    max_connections: usize,
    #[serde(skip_serializing_if = "Option::is_none")]
    tls_cert_path: Option<String>,
}

// JSON
let json_input = r#"{"name": "hw-diag", "port": 8080}"#;
let config: ServerConfig = serde_json::from_str(json_input)?;

// Same struct, TOML — no code changes
let config: ServerConfig = toml::from_str(toml_input)?;
```

Derive `Serialize`/`Deserialize` once. Works with every serde-compatible format.

## Common serde Attributes

| Attribute | Level | Effect |
|---|---|---|
| `rename_all = "camelCase"` | Container | Rename all fields (snake_case, SCREAMING_SNAKE_CASE, kebab-case, etc.) |
| `deny_unknown_fields` | Container | Reject extra keys (strict mode) |
| `default` | Field | Use `Default::default()` when missing |
| `rename = "..."` | Field | Custom serialized name |
| `skip` | Field | Exclude entirely |
| `skip_serializing_if = "fn"` | Field | Conditionally exclude (e.g., `Option::is_none`) |
| `flatten` | Field | Inline nested struct's fields |
| `with = "module"` | Field | Custom ser/de functions |
| `alias = "..."` | Field | Accept alternative names during deserialize |
| `deserialize_with = "fn"` | Field | Custom deserialize only |

## Enum Representations

```rust
// 1. Externally tagged (DEFAULT)
enum Command {
    Reboot,
    RunDiag { test_name: String, timeout_secs: u64 },
}
// "Reboot"
// {"RunDiag": {"test_name": "gpu", "timeout_secs": 60}}

// 2. Internally tagged — #[serde(tag = "type")]
#[serde(tag = "type")]
enum Event {
    Start { timestamp: u64 },
    Error { code: i32, message: String },
}
// {"type": "Start", "timestamp": 1706000000}

// 3. Adjacently tagged — #[serde(tag = "t", content = "c")]
#[serde(tag = "t", content = "c")]
enum Payload { Text(String), Binary(Vec<u8>) }
// {"t": "Text", "c": "hello"}

// 4. Untagged — #[serde(untagged)] — first matching variant wins
#[serde(untagged)]
enum StringOrNumber { Str(String), Num(f64) }
```

**Pick**: Internally tagged (`tag = "type"`) for most JSON APIs. Untagged for union types where the shape alone disambiguates.

## Zero-Copy Deserialization

Borrow from the input buffer instead of allocating:

```rust
#[derive(Deserialize)]
struct OwnedRecord { name: String, value: String }  // 2 allocations

#[derive(Deserialize)]
struct BorrowedRecord<'a> { name: &'a str, value: &'a str }  // 0 allocations

let input = r#"{"name": "cpu_temp", "value": "72.5"}"#;
let borrowed: BorrowedRecord = serde_json::from_str(input)?;
// borrowed.name and borrowed.value point INTO input
```

### Understanding the Lifetimes

```rust
// Deserialize<'de>: struct can borrow from data with lifetime 'de
// DeserializeOwned: struct owns all its data, no borrowing
//   trait DeserializeOwned: for<'de> Deserialize<'de> {}

fn parse_owned<T: DeserializeOwned>(input: &str) -> T {
    serde_json::from_str(input).unwrap()
}

fn parse_borrowed<'a, T: Deserialize<'a>>(input: &'a str) -> T {
    serde_json::from_str(input).unwrap()
}
```

**Use zero-copy when**: large files, read-heavy pipelines, input buffer lives long enough (memory-mapped file, network packet with owned buffer).

**Avoid when**: input is ephemeral (network read buffer that's reused), result must outlive input, fields need transformation.

**Practical tip**: `Cow<'a, str>` gives the best of both — borrow when possible, allocate when necessary (e.g., JSON escape sequences). serde supports `Cow` natively.

## Format Ecosystem

| Format | Crate | Human-Readable | Size | Speed | Use case |
|---|---|---|---|---|---|
| JSON | `serde_json` | ✅ | Large | Good | Config, REST APIs, logs |
| TOML | `toml` | ✅ | Medium | Good | Config (Cargo.toml style) |
| YAML | `serde-saphyr` | ✅ | Medium | Good | Nested config (MSRV 1.89). Old `serde_yaml` API → `serde_norway`. Not `serde_yaml` (archived) or `serde_yml` (RUSTSEC-2025-0068) |
| postcard | `postcard` | ❌ | Tiny | Very fast | New Rust-to-Rust IPC / `no_std` |
| rkyv | `rkyv` | ❌ | Tiny | Zero-copy | Large buffers, mmap, IPC where you read without allocating |
| MessagePack | `rmp-serde` | ❌ | Small | Fast | Cross-language binary |
| CBOR | `ciborium` | ❌ | Small | Fast | IoT, constrained |

**Choose**:
- Config humans edit → TOML or JSON. YAML → `serde-saphyr` (`serde_saphyr::from_str`). Drop-in for old `serde_yaml` → `serde_norway`. Do not use `serde_yml`
- New Rust-to-Rust IPC/cache → **postcard**. `bincode` is unmaintained (`cargo add bincode` resolves to a 3.0 stub that is a compiler error). Existing bincode wire format → `wincode`
- Zero-copy of large in-memory graphs → `rkyv`
- Cross-language binary → MessagePack or CBOR
- Embedded / `no_std` → postcard

## Binary Data and `repr(C)`

Predictable memory layout for hardware registers and protocol headers:

```rust
#[repr(C)]
#[derive(Debug, Clone, Copy)]
struct IpmiHeader {
    rs_addr: u8,
    net_fn_lun: u8,
    checksum: u8,
    rq_addr: u8,
    rq_seq_lun: u8,
    cmd: u8,
}

impl IpmiHeader {
    fn from_bytes(data: &[u8]) -> Option<Self> {
        if data.len() < std::mem::size_of::<Self>() { return None; }
        Some(Self {
            rs_addr: data[0], net_fn_lun: data[1], checksum: data[2],
            rq_addr: data[3], rq_seq_lun: data[4], cmd: data[5],
        })
    }
}

// Endianness-aware parsing
fn read_u16_le(data: &[u8], offset: usize) -> u16 {
    u16::from_le_bytes([data[offset], data[offset + 1]])
}
fn read_u32_be(data: &[u8], offset: usize) -> u32 {
    u32::from_be_bytes([data[offset], data[offset+1], data[offset+2], data[offset+3]])
}
```

### Packed Structs

```rust
#[repr(C, packed)]
#[derive(Debug, Clone, Copy)]
struct PcieCapabilityHeader {
    cap_id: u8,
    next_cap: u8,
    cap_reg: u16,
}
// ⚠️ Taking &field creates an unaligned reference — UB.
// Always copy fields out: let id = header.cap_id;   // OK (Copy)
// Never: let r = &header.cap_reg;                  // UB if unaligned
```

## zerocopy and bytemuck — Safe Transmutation

Replace `unsafe { transmute() }` with compile-time-checked alternatives:

```rust
// zerocopy 0.8 — Cargo.toml: zerocopy = { version = "0.8", features = ["derive"] }
use zerocopy::{FromBytes, IntoBytes, KnownLayout, Immutable};

#[derive(FromBytes, IntoBytes, KnownLayout, Immutable)]
#[repr(C)]
struct SensorReading {
    sensor_id: u16, flags: u8, _reserved: u8, value: u32,
}

fn parse_sensor(raw: &[u8]) -> Option<&SensorReading> {
    // Derive proves the *type* is transmutable. Size/alignment of *this slice*
    // is checked at runtime — handle Err. Native-endian u32 is wrong for
    // on-wire layouts; use from_le_bytes or zerocopy's endian wrappers.
    SensorReading::ref_from_bytes(raw).ok()
}

// bytemuck — Cargo.toml: bytemuck = { version = "1", features = ["derive"] }
use bytemuck::{Pod, Zeroable};
#[derive(Pod, Zeroable, Clone, Copy)]
#[repr(C)]
struct GpuRegister { address: u32, value: u32 }

fn cast_registers(data: &[u8]) -> Option<&[GpuRegister]> {
    // cast_slice panics on bad size/alignment; try_cast_slice is fallible
    bytemuck::try_cast_slice(data).ok()
}
```

| Approach | Safety | Overhead | Use when |
|---|---|---|---|
| Manual field-by-field | ✅ Safe | Copy fields | Small structs, complex layouts |
| `zerocopy` | ✅ Safe | Zero-copy | Large buffers; type layout compile-time, slice size/align runtime |
| `bytemuck` | ✅ Safe | Zero-copy | Simple `Pod` types; `try_cast_slice` (infallible `cast_slice` panics) |
| `unsafe { transmute() }` | ❌ Unsafe | Zero-copy | Last resort — avoid |

## bytes::Bytes — Reference-Counted Buffers

`Bytes` is to `Vec<u8>` what `Arc<[u8]>` is to owned slices. Zero-copy sub-slicing.

```rust
use bytes::{Bytes, BytesMut, Buf, BufMut};

let mut buf = BytesMut::with_capacity(1024);
buf.put_u8(0x01);
buf.put_u16(0x1234);            // big-endian; use put_u16_le for little-endian
buf.put_slice(b"hello");
let data: Bytes = buf.freeze();

let data2 = data.clone();           // O(1) refcount, NOT deep copy
let slice = data.slice(3..8);        // zero-copy sub-slice, shares buffer

// Split without copying:
let mut original = Bytes::from_static(b"HEADER\x00PAYLOAD");
let header = original.split_to(6);  // header = "HEADER", original = "\x00PAYLOAD"
```

| | `Vec<u8>` | `Bytes` |
|---|---|---|
| Clone cost | O(n) deep copy | O(1) refcount |
| Sub-slicing | Borrows with lifetime | Owned, refcount-tracked |
| Sharing | `Send + Sync`; clone is O(n) (or wrap in `Arc`) | `Send + Sync`; clone is O(1) |
| Used by | std | tokio, hyper, tonic, axum |

**Use for**: network protocols, packet parsing, splitting buffers across components/threads.

## Anti-Patterns

- **Using `unsafe { transmute() }` when `bytemuck`/`zerocopy` works.** The crates verify safety at compile time.
- **Zero-copy when input is ephemeral.** Result outlives input → use `DeserializeOwned` instead.
- **Always using `#[serde(flatten)]`.** Breaks `deny_unknown_fields` and can cause unexpected matches.
- **Custom `Deserialize` for what serde attributes handle.** Reach for attributes first; custom only when needed.
- **Untagged enums with overlapping variants.** First-match-wins means reordering variants breaks callers.
- **Reading unaligned `&field` from `#[repr(C, packed)]`.** Undefined behavior — copy out with `let id = field.id`.
- **Forgetting endianness.** Always use `from_le_bytes`/`from_be_bytes`/`to_le_bytes`/`to_be_bytes`, never assume host endianness.
- **Using `Vec<u8>` for network buffers.** `Bytes` gives you cheap cloning and zero-copy slicing.
- **`cargo add bincode` on a new project.** Latest `bincode` 3.0 is an unmaintained stub that fails to compile. Use `postcard` (or `wincode` for the old wire format).

## See Also

- [error-handling.md](./error-handling.md) — combining serde errors with `thiserror`/`anyhow`
- [unsafe.md](./unsafe.md) — `repr(C)` and FFI data layouts
- [smart-pointers.md](./smart-pointers.md) — `Cow` for clone-on-write binary pipelines