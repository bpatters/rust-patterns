# Unsafe Rust — Controlled Danger

Load this when: writing or reviewing `unsafe` code; calling C from Rust; building primitives (Vec, HashMap internals); FFI boundaries; custom allocators; deciding whether `unsafe` is justified.

## The Five Unsafe Superpowers

`unsafe` unlocks exactly five operations. It does NOT turn off the borrow checker or type system — all other Rust rules still apply.

1. **Dereference a raw pointer** (`*const T`, `*mut T`)
2. **Call an unsafe function or method**
3. **Access or modify a `static mut` variable**
4. **Implement an unsafe trait** (`unsafe impl Send for MyType`)
5. **Access fields of a `union`**

## The Three Rules of Sound Unsafe Code

1. **Document invariants** — every `unsafe` block needs a `// SAFETY:` comment explaining why it's sound
2. **Encapsulate** — the unsafe stays inside a safe API; users can't trigger UB
3. **Minimize** — only the smallest possible block is `unsafe`

```rust
/// A fixed-capacity stack-allocated buffer. All public methods are safe.
pub struct StackBuf<T, const N: usize> {
    data: [std::mem::MaybeUninit<T>; N],
    len: usize,
}

impl<T, const N: usize> StackBuf<T, N> {
    pub fn new() -> Self {
        StackBuf {
            data: [const { std::mem::MaybeUninit::uninit() }; N],  // Rust 1.79+
            len: 0,
        }
    }

    pub fn push(&mut self, value: T) -> Result<(), T> {
        if self.len >= N { return Err(value); }
        // SAFETY: len < N, so data[len] is within bounds.
        self.data[self.len] = std::mem::MaybeUninit::new(value);
        self.len += 1;
        Ok(())
    }

    pub fn get(&self, index: usize) -> Option<&T> {
        if index < self.len {
            // SAFETY: index < len, and data[0..len] are all initialized.
            Some(unsafe { self.data[index].assume_init_ref() })
        } else { None }
    }
}

impl<T, const N: usize> Drop for StackBuf<T, N> {
    fn drop(&mut self) {
        // SAFETY: data[0..len] are initialized — drop them properly.
        for i in 0..self.len {
            unsafe { self.data[i].assume_init_drop() };
        }
    }
}
```

## FFI Patterns — Calling C from Rust

```rust
extern "C" {
    fn strlen(s: *const std::ffi::c_char) -> usize;
    fn printf(format: *const std::ffi::c_char, ...) -> std::ffi::c_int;
}

fn safe_strlen(s: &str) -> usize {
    let c_string = std::ffi::CString::new(s).expect("string contains null byte");
    // SAFETY: c_string is a valid null-terminated string, alive for the call.
    unsafe { strlen(c_string.as_ptr()) }
}

// Calling Rust from C:
#[no_mangle]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}
```

### Common FFI Types

| Rust | C | Notes |
|---|---|---|
| `i32` / `u32` | `int32_t` / `uint32_t` | Fixed-width, safe |
| `*const T` / `*mut T` | `const T*` / `T*` | Raw pointers |
| `std::ffi::CStr` | `const char*` (borrowed) | Null-terminated, borrowed |
| `std::ffi::CString` | `char*` (owned) | Null-terminated, owned |
| `std::ffi::c_void` | `void` | Opaque pointer target |
| `Option<fn(...)>` | Nullable function pointer | `None` = NULL |

## Common UB Pitfalls

| Pitfall | Why it's UB |
|---|---|
| Null dereference: `*std::ptr::null::<i32>()` | Dereferencing null is always UB |
| Dangling pointer | Dereference after `drop()` — memory may be reused |
| Data race | Two threads write to `static mut` without sync |
| Wrong `assume_init` | `MaybeUninit::<String>::uninit().assume_init()` reads uninit memory |
| Aliasing violation | Creating two `&mut` to same data |
| Invalid enum value | `transmute::<u8, bool>(2)` — `bool` can only be 0 or 1 |
| Unaligned reference from packed | `&field` on `#[repr(C, packed)]` may be unaligned |

**Note**: `[const { MaybeUninit::uninit() }; N]` (Rust 1.79+) is the safe way to create an array of `MaybeUninit` — no `unsafe` or `assume_init` needed.

## When to Use `unsafe` in Production

- FFI boundaries (calling C/C++ code)
- Performance-critical inner loops (avoid bounds checks)
- Building primitives (`Vec`, `HashMap` — they use `unsafe` internally)
- **Never** in application logic if you can avoid it

If you're reaching for `unsafe` in regular code, ask: is there a safe abstraction I'm missing? (e.g., `bytemuck`/`zerocopy` instead of `transmute`, `parking_lot::Mutex` instead of hand-rolled spin lock)

## Custom Allocators — Arena and Slab

In systems programming you often need bulk-allocation patterns. Rust provides safe wrappers.

### Arena — Bulk Alloc, Bulk Free (bumpalo)

```rust
use bumpalo::Bump;

fn process_sensor_frame(raw_data: &[u8]) {
    let arena = Bump::new();
    let header = arena.alloc(parse_header(raw_data));
    let readings: &mut [f32] = arena.alloc_slice_fill_default(header.sensor_count);

    for (i, chunk) in raw_data[header.payload_offset..].chunks(4).enumerate() {
        if i < readings.len() {
            readings[i] = f32::from_le_bytes(chunk.try_into().unwrap());
        }
    }
    // arena drops here — ALL allocations freed at once in O(1)
}
```

| | `Vec`/`Box` | Arena (`Bump`) |
|---|---|---|
| Alloc speed | ~25ns (malloc) | ~2ns (pointer bump) |
| Free speed | Per-object destructor | O(1) bulk free |
| Fragmentation | Yes (long-lived processes) | None within arena |
| Use case | General purpose | Request/frame/batch |

### typed-arena — Type-Safe Arena

```rust
use typed_arena::Arena;

struct AstNode<'a> { value: i32, children: Vec<&'a AstNode<'a>> }

fn build_tree() {
    let arena: Arena<AstNode<'_>> = Arena::new();
    let root = arena.alloc(AstNode { value: 1, children: vec![] });
    let left = arena.alloc(AstNode { value: 2, children: vec![] });
    // All references valid as long as `arena` lives — compile-time scoped
}
```

### Slab — Fixed-Size Object Pool

```rust
use slab::Slab;

struct Connection { id: u64, buffer: [u8; 1024], active: bool }

fn connection_pool() {
    let mut connections: Slab<Connection> = Slab::with_capacity(256);

    let key1 = connections.insert(Connection { id: 1001, buffer: [0; 1024], active: true });
    if let Some(conn) = connections.get_mut(key1) {
        conn.buffer[0..5].copy_from_slice(b"hello");
    }

    let removed = connections.remove(key2);  // O(1), slot reused
    let key3 = connections.insert(Connection { /* ... */ });
    assert_eq!(key3, key2);  // Same slot reused
}
```

### Choosing an Allocator

```text
All same type?
├── Yes → Need individual free?
│    ├── Yes → Slab
│    └── No  → typed-arena
└── No  → Need individual free?
     ├── Yes → Standard allocator (Box, Vec)
     └── No  → Bump arena

Environment?
├── no_std  → Custom FixedArena or embedded-alloc
└── std     → bumpalo / typed-arena / slab
```

## Anti-Patterns

- **`unsafe` without `// SAFETY:` comments.** Always justify the soundness.
- **`unsafe` that escapes its block's scope.** Each `unsafe` block should be the smallest possible scope.
- **Using `transmute` when `bytemuck`/`zerocopy` works.** The safe alternatives verify at compile time.
- **Hand-rolled spinlocks, lock-free queues, or atomics patterns.** Reach for `parking_lot`, `crossbeam`, `arc-swap`, `dashmap` — proven implementations.
- **`static mut` for shared state.** `static` + `AtomicX` or `Mutex`/`RwLock` instead.
- **Forgetting `#[repr(C)]` on FFI structs.** Layout is undefined otherwise.
- **Exposing `unsafe` in public APIs without safe wrappers.** Library users should never need `unsafe` for normal use.
- **Allocating in arena types with non-trivial `Drop` impls.** Arena doesn't call destructors — file handles, sockets leak. Only allocate types without meaningful `Drop`, or manually drop them before the arena drops.

## See Also

- [smart-pointers.md](./smart-pointers.md) — `Pin`, `MaybeUninit`, `ManuallyDrop`, drop order
- [phantomdata.md](./phantomdata.md) — variance + drop-check interactions
- [serialization.md](./serialization.md) — `repr(C)` + `zerocopy`/`bytemuck`
- [api-design.md](./api-design.md) — sealing traits so users can't implement them wrong