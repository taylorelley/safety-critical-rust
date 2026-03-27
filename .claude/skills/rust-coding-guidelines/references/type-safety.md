# Type Safety Guidelines

## gui_0cuTYG8RVYjg — Ensure union field reads produce valid values

- **Category:** Required | **Decidability:** Undecidable | **Scope:** System
- **Tags:** defect, safety, undefined-behavior

### Rule

Ensure that the underlying bytes constitute a valid value for that field's type when reading from a union field. Reading a union field whose bytes do not represent a valid value is **undefined behavior**.

Before accessing a union field, verify that the union was either:
- Last written through that field, or
- Written through a field whose bytes are valid when reinterpreted as the target field's type

### Validity Invariants by Type

- **bool**: Must be `0` (false) or `1` (true). Any other value is invalid.
- **char**: Must be a valid Unicode scalar value (`0x0`–`0xD7FF` or `0xE000`–`0x10FFFF`).
- **References**: Must be non-null and properly aligned.
- **Enums**: Must hold a valid discriminant value.
- **Floating point**: All bit patterns are valid.
- **Integers**: All bit patterns are valid.

### Non-Compliant Examples

Reading an invalid bool:

```rust
union IntOrBool { i: u8, b: bool }

fn main() {
    let u = IntOrBool { i: 3 };
    unsafe { u.b }; // UB: 3 is not a valid bool
}
```

Reading an invalid char:

```rust
union IntOrChar { i: u32, c: char }

fn main() {
    let u = IntOrChar { i: 0xD800 }; // surrogate, not valid Unicode
    unsafe { u.c }; // UB
}
```

Reading an invalid enum discriminant:

```rust
#[repr(u8)]
enum Color { Red = 0, Green = 1, Blue = 2 }

union IntOrColor { i: u8, c: Color }

fn main() {
    let u = IntOrColor { i: 42 };
    unsafe { u.c }; // UB: 42 is not a valid Color
}
```

Reading a null reference:

```rust
union PtrOrRef { p: *const i32, r: &'static i32 }

fn main() {
    let u = PtrOrRef { p: std::ptr::null() };
    unsafe { u.r }; // UB: null is not a valid reference
}
```

### Compliant Examples

**Track active field at runtime:**

```rust
#[repr(C)]
union IntOrBoolData { i: u8, b: bool }

enum ActiveField { Int, Bool }

pub struct IntOrBool {
    data: IntOrBoolData,
    active: ActiveField,
}

impl IntOrBool {
    pub fn as_int(&self) -> Option<u8> {
        match self.active {
            // SAFETY: We only read `i` when it was last written as `i`
            ActiveField::Int => Some(unsafe { self.data.i }),
            ActiveField::Bool => None,
        }
    }

    pub fn as_bool(&self) -> Option<bool> {
        match self.active {
            // SAFETY: We only read `b` when it was last written as `b`
            ActiveField::Bool => Some(unsafe { self.data.b }),
            ActiveField::Int => None,
        }
    }
}
```

**Read as an always-valid type, then validate:**

```rust
union IntOrBool { i: u8, b: bool }

fn try_read_bool(u: &IntOrBool) -> Option<bool> {
    // SAFETY: Reading as u8 is always valid (all bit patterns valid)
    let raw = unsafe { u.i };
    match raw {
        0 => Some(false),
        1 => Some(true),
        _ => None,
    }
}
```

**Type-state pattern with PhantomData (zero-cost):**

```rust
use std::marker::PhantomData;

pub struct AsInt;
pub struct AsBool;

#[repr(C)]
pub union IntOrBoolData { pub i: u8, pub b: bool }

pub struct IntOrBool<T> {
    data: IntOrBoolData,
    _marker: PhantomData<T>,
}

impl IntOrBool<AsInt> {
    pub fn get(&self) -> u8 {
        // SAFETY: Type parameter `AsInt` guarantees integer field is active
        unsafe { self.data.i }
    }
}

impl IntOrBool<AsBool> {
    pub fn get(&self) -> bool {
        // SAFETY: Type parameter `AsBool` guarantees boolean field is active
        unsafe { self.data.b }
    }
}
```

---

## gui_xztNdXA2oFNC — Use strong types for logically distinct values

- **Category:** Advisory | **Decidability:** Undecidable | **Scope:** Module
- **Tags:** types, safety, understandability

### Rule

Parameters and variables with logically distinct types must be statically distinguishable by the type system. Use a newtype (e.g., `struct Meters(u32)`) when:

- Two or more quantities share the same primitive but are logically distinct
- Confusing them would be a semantic error
- You need type-safe encapsulation or domain-specific trait implementations

### Rationale

Primitive types like `u32` can represent lengths, counters, timestamps, IDs, etc. The compiler cannot distinguish between these semantically different uses. Newtypes make the distinction a compile-time guarantee.

### Non-Compliant Examples

Bare primitives — nothing prevents swapping distance and time:

```rust
fn travel(distance: u32, time: u32) -> u32 {
    distance / time
}

fn main() {
    let d = 100;
    let t = 10;
    let _result = travel(t, d); // Compiles but semantically wrong
}
```

Type aliases — still not distinct types:

```rust
type Meters = u32;
type Seconds = u32;

fn travel(distance: Meters, time: Seconds) -> u32 {
    distance / time
}

fn main() {
    let d: Meters = 100;
    let t: Seconds = 10;
    let _result = travel(t, d); // Still compiles! Aliases are not distinct types
}
```

### Compliant Example

Newtypes with domain-specific trait implementations:

```rust
use std::ops::Div;

#[derive(Debug, Clone, Copy)]
struct Meters(u32);

#[derive(Debug, Clone, Copy)]
struct Seconds(u32);

#[derive(Debug, Clone, Copy)]
struct MetersPerSecond(u32);

impl Div<Seconds> for Meters {
    type Output = MetersPerSecond;
    fn div(self, rhs: Seconds) -> Self::Output {
        MetersPerSecond(self.0 / rhs.0)
    }
}

fn main() {
    let d = Meters(100);
    let t = Seconds(10);
    let result = d / t; // Type-safe
    println!("Speed: {} m/s", result.0);
}
```

---

## gui_HDnAZ7EZ4z6G — Do not use `as _` in pointer casts

- **Category:** Required | **Decidability:** Decidable | **Scope:** Module
- **Tags:** readability, reduce-human-error

### Rule

Code must not rely on type inference when doing explicit pointer casts via `as` or `transmute`. Always specify the complete target type.

### Rationale

Not specifying the concrete target pointer type allows the compiler to infer it from surrounding context. If the surrounding types change, the cast may silently change semantics, resulting in invalid pointer casts.

### Non-Compliant Example

```rust
#[repr(C)]
struct Base { position: (u32, u32) }

#[repr(C)]
struct Extended { base: Base, scale: f32 }

fn non_compliant_example(extended: &Extended) {
    let extended = extended as *const _; // type inferred
    with_base(unsafe { &*(extended as *const _) }) // type inferred
}

fn with_base(_: &Base) {}
```

### Compliant Example

```rust
#[repr(C)]
struct Base { position: (u32, u32) }

#[repr(C)]
struct Extended { base: Base, scale: f32 }

fn compliant_example(extended: &Extended) {
    let extended = extended as *const Extended; // explicit
    with_base(unsafe { &*(extended as *const Base) }) // explicit
}

fn with_base(_: &Base) {}
```

---

## gui_PM8Vpf7lZ51U — Do not convert integer to pointer

- **Category:** TBD | **Decidability:** Decidable | **Scope:** Module
- **Tags:** subset, undefined-behavior

### Rule

The `as` operator shall not be used with a numeric expression as the left operand and any pointer type as the right operand. `transmute` shall not be used with any numeric type as source and any pointer type as destination.

### Rationale

A pointer created from an arbitrary arithmetic expression may designate an invalid address, point to an object of the wrong type, or be improperly aligned. The `as` operator also does not check that the source size matches pointer size.

### Non-Compliant Example

```rust
fn f1(x: u16, y: i32, z: u64, w: usize) {
    let _p1 = x as *const u32;  // non-compliant
    let _p2 = y as *const u32;  // non-compliant
    let _p3 = z as *const u32;  // non-compliant
    let _p4 = w as *const u32;  // non-compliant despite right size

    unsafe {
        let _p7: *const u32 = std::mem::transmute(z); // non-compliant
        let _p8: *const u32 = std::mem::transmute(w); // non-compliant
    }
}
```

### Compliant Example

There is no compliant way to convert an integer to a pointer. Use `core::ptr::null()` / `core::ptr::null_mut()` for null pointers. For other needs, restructure code to avoid integer-to-pointer conversion.
