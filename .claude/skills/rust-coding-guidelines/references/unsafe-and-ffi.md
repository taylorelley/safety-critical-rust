# Unsafe and FFI Guidelines

## gui_ZDLZzjeOwLSU — Assure visibility of `unsafe` keyword in unsafe code

- **Category:** Required | **Decidability:** Decidable | **Scope:** Crate
- **Tags:** readability, reduce-human-error

### Rule

Mark all code that may violate safety guarantees with a visible `unsafe` keyword. Specifically:

| Construct | Required Form |
|---|---|
| `extern` blocks | `unsafe extern` |
| `#[no_mangle]` | `#[unsafe(no_mangle)]` |
| `#[export_name]` | `#[unsafe(export_name = "...")]` |
| `#[link_section]` | `#[unsafe(link_section = "...")]` |

Starting with Rust Edition 2024, missing `unsafe` in these contexts is a compilation error.

### Rationale

- **Auditability:** `unsafe` blocks create clear audit boundaries. Safety standards (ISO 26262, DO-178C) require traceability of hazardous operations.
- **Explicit responsibility:** The `unsafe` keyword signals the programmer is taking responsibility for upholding invariants the compiler cannot verify.
- **Tooling:** `cargo-geiger` and similar tools can locate and count unsafe blocks for safety assessments.
- **Certification:** Safety-critical certifications require demonstrating that hazardous operations are identified and controlled.

### Non-Compliant Examples

Missing `unsafe` on `#[no_mangle]`:

```rust
// Non-compliant (fails to compile in Edition 2024)
#[no_mangle]
fn convert() {}
```

Missing `unsafe` on `extern` block:

```rust
// Non-compliant (fails to compile in Edition 2024)
use std::ffi;

extern "C" {
    fn malloc(size: f32) -> *mut ffi::c_void; // also: wrong type for size
}
```

Missing `unsafe` on `#[export_name]` and `#[link_section]`:

```rust
// Non-compliant
#[export_name = "printf"]     // could collide with C library
#[link_section = ".init_array"] // corrupts initialization table
static DATA: u32 = 42;
```

### Compliant Examples

`unsafe` on `#[no_mangle]`:

```rust
#[unsafe(no_mangle)]
fn convert() {}
```

`unsafe extern` block with correct types:

```rust
use std::ffi;

unsafe extern "C" {
    fn malloc(size: usize) -> *mut ffi::c_void;
    fn free(ptr: *mut ffi::c_void);
}

fn main() {
    unsafe {
        let ptr = malloc(1024);
        if !ptr.is_null() {
            free(ptr);
        }
    }
}
```

`unsafe` wrappers on `#[export_name]` and `#[link_section]`:

```rust
// SAFETY: 'custom_symbol' does not conflict with any other symbol
#[unsafe(export_name = "custom_symbol")]
pub fn my_function() {}

// SAFETY: Placing data in a specific section for embedded systems
#[unsafe(link_section = ".noinit")]
static mut PERSISTENT_DATA: [u8; 256] = [0; 256];
```

### Enforcement

- **Rust Edition 2024:** Violations are compilation errors
- **Compiler Lints:**
  - `#![deny(unsafe_code)]` — deny all unsafe (use `#[allow(unsafe_code)]` for justified exceptions)
  - `#![deny(unsafe_op_in_unsafe_fn)]` — require explicit unsafe blocks within unsafe functions
  - `#![warn(unsafe_attr_outside_unsafe)]` — warn about unsafe attributes without `unsafe()` wrapper (pre-2024)
- **Static Analysis:** `cargo-geiger` for unsafe code metrics

---

## General Unsafe and FFI Principles

These principles are synthesized from the guidelines corpus and apply broadly to safety-critical Rust:

### Minimize Unsafe Surface Area

- Use `#![forbid(unsafe_code)]` at the crate level where possible
- Isolate `unsafe` code into small, well-documented modules
- Create safe abstractions that encapsulate unsafe operations
- Every `unsafe` block must have a `// SAFETY:` comment explaining why it is sound

### SAFETY Comment Pattern

```rust
// SAFETY: `ptr` is guaranteed non-null and properly aligned because
// it was obtained from `Box::into_raw` in the constructor, and we
// have exclusive access via `&mut self`.
unsafe { *ptr = value; }
```

### FFI Boundary Design

- Use `unsafe extern` blocks (Edition 2024+) for all FFI declarations
- Validate all data crossing the FFI boundary at the Rust side
- Create safe Rust wrapper types around FFI types
- Convert C errors to `Result` types at the boundary
- Use `repr(C)` for types shared with C code
- Prefer `CStr`/`CString` over raw `*const c_char`

### Unsafe Code Review Checklist

When reviewing `unsafe` code in safety-critical contexts, verify:

1. Every `unsafe` block has a `// SAFETY:` comment
2. The safety justification is correct and complete
3. All pointer dereferences are to valid, aligned, non-null pointers
4. No aliasing violations (`&` and `&mut` to same data)
5. All type invariants are maintained (see gui_0cuTYG8RVYjg for union fields)
6. No data races in concurrent code
7. FFI type declarations match the actual C signatures
