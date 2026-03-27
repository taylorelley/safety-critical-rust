# Numeric Safety Guidelines

## gui_dCquvqE1csI3 — Ensure integer operations do not overflow

- **Category:** Required | **Decidability:** Decidable | **Scope:** System
- **Tags:** security, performance, numerics

### Rule

Eliminate arithmetic overflow of both signed and unsigned integer types. Any wraparound behavior must be explicitly specified to ensure the same behavior in both debug and release modes.

Applies to all primitive integer types: `i8`, `i16`, `i32`, `i64`, `i128`, `u8`, `u16`, `u32`, `u64`, `u128`, `usize`, `isize`.

### Rationale

Arithmetic overflow panics in debug mode but wraps in release mode, resulting in inconsistent behavior. Use explicit `Wrapping` or `Saturating` semantics where these behaviors are intentional. Range checking can also eliminate overflow.

### Non-Compliant Example

Direct arithmetic without overflow protection:

```rust
fn add(si_a: i32, si_b: i32) {
    let _sum: i32 = si_a + si_b; // May overflow
}
```

Division that checks for zero but not overflow:

```rust
fn div(s_a: i64, s_b: i64) -> Result<i64, DivError> {
    if s_b == 0 {
        Err(DivError::DivisionByZero)
    } else {
        Ok(s_a / s_b) // i64::MIN / -1 overflows
    }
}
```

### Compliant Examples

**Checked arithmetic** — returns `None` on overflow:

```rust
fn add(si_a: i32, si_b: i32) -> Result<i32, ArithmeticError> {
    si_a.checked_add(si_b).ok_or(ArithmeticError::Overflow)
}

fn sub(a: i32, b: i32) -> Result<i32, ArithmeticError> {
    a.checked_sub(b).ok_or(ArithmeticError::Overflow)
}

fn mul(a: i32, b: i32) -> Result<i32, ArithmeticError> {
    a.checked_mul(b).ok_or(ArithmeticError::Overflow)
}
```

**Wrapping arithmetic** — when wraparound is intentional (e.g., cryptography, hashing):

```rust
fn add(a: i32, b: i32) -> i32 {
    a.wrapping_add(b)
}
```

Or use the `Wrapping` type for consistent wrapping behavior:

```rust
use std::num::Wrapping;

fn add(si_a: Wrapping<i32>, si_b: Wrapping<i32>) -> Wrapping<i32> {
    si_a + si_b
}
```

**Saturating arithmetic** — clamps to min/max:

```rust
fn add(a: i32, b: i32) -> i32 {
    a.saturating_add(b)
}
```

Or use the `Saturating` type:

```rust
use std::num::Saturating;

fn add(si_a: Saturating<i32>, si_b: Saturating<i32>) -> Saturating<i32> {
    si_a + si_b
}
```

**Division with full overflow protection:**

```rust
fn div(s_a: i64, s_b: i64) -> Result<i64, DivError> {
    if s_b == 0 {
        Err(DivError::DivisionByZero)
    } else if s_a == i64::MIN && s_b == -1 {
        Err(DivError::Overflow)
    } else {
        Ok(s_a / s_b)
    }
}
```

---

## gui_7y0GAMmtMhch — Do not use integer type as divisor in integer division

- **Category:** Advisory | **Decidability:** Decidable | **Scope:** Module
- **Tags:** numerics, subset

### Rule

Do not provide a right operand of integer type during a division or remainder expression when the left operand also has integer type. This is a strict subsetting rule — it eliminates all integer division/remainder at the language level.

### Rationale

Integer division and remainder both panic when the right operand is zero. Division by zero is undefined in mathematics.

### Non-Compliant Example

```rust
fn main() {
    let x = 0;
    let _y = 5 / x; // Panics if x == 0
    let _z = 5 % x; // Also panics
}
```

### Compliant Examples

**Checked division:**

```rust
let _y = match 5i32.checked_div(0) {
    None => 0,
    Some(r) => r,
};

let _z = match 5i32.checked_rem(0) {
    None => 0,
    Some(r) => r,
};
```

**NonZero types** — guarantee the divisor is never zero:

```rust
use std::num::NonZero;

let x = 0u32;
if let Some(divisor) = NonZero::<u32>::new(x) {
    let _result = 5u32 / divisor;
}
```

---

## gui_kMbiWbn8Z6g5 — Do not divide by zero

- **Category:** Required | **Decidability:** Undecidable | **Scope:** System
- **Tags:** numerics, defect

### Rule

Integer division by zero results in a panic. This includes both division and remainder expressions. This is a less strict version of gui_7y0GAMmtMhch — it allows integer division but requires proving the divisor is non-zero.

### Rationale

Integer division by zero results in a panic — an abnormal program state that may terminate the process.

### Non-Compliant Example

```rust
fn main() {
    let x = 0;
    let _y = 5 / x; // Panics
    let _z = 5 % x; // Panics
}
```

### Compliant Example

Manual zero check (less preferred as complexity increases):

```rust
let x = 0u32;
let _y = if x != 0u32 {
    5u32 / x
} else {
    0u32
};
```

Also compliant: all solutions from gui_7y0GAMmtMhch (checked_div, NonZero).

---

## gui_ADHABsmK9FXz — Do not use `as` with numeric operands

- **Category:** Advisory | **Decidability:** Decidable | **Scope:** Module
- **Tags:** subset, reduce-human-error

### Rule

The `as` operator should not be used with numeric types (`i8`–`i128`, `u8`–`u128`, `isize`, `usize`, `f32`, `f64`), `bool`, or `char` as either operand.

**Exception:** `as` may be used with `usize` as the right operand and a raw pointer as the left operand (pointer-to-address cast).

### Rationale

`as` coerces values to fit, which may silently truncate, round, or lose precision. Use `Into`/`From` for lossless conversions and `TryInto`/`TryFrom` for fallible conversions to communicate intent explicitly.

### Non-Compliant Example

```rust
fn f1(x: u16, y: i32, _z: u64, w: u8) {
    let _a = w as char;           // non-compliant
    let _b = y as u32;            // changes value range
    let _c = x as i64;            // could use .into()
    let d = y as f32;             // lossy
    let e = d as f64;             // could use .into()
    let _f = e as f32;            // lossy

    let b: u32 = 0;
    let p1: *const u32 = &b;
    let _a1 = p1 as usize;       // compliant by exception
    let _a2 = p1 as u16;         // may lose address range
}
```

### Compliant Example

```rust
use std::convert::TryInto;

fn f2(x: u16, y: i32, _z: u64, w: u8) {
    let _a: char            = w.into();
    let _b: Result<u32, _>  = y.try_into(); // error on range clip
    let _c: i64             = x.into();
    let d = f32::from(x);  // u16 fits in f32
    let _e = f64::from(d);

    let h: u32 = 0;
    let p1: *const u32 = &h;
    let _a1 = p1 as usize;  // compliant exception

    unsafe {
        let _a2: usize = std::mem::transmute(p1);
    }
}
```

---

## gui_RHvQj8BHlz9b — Do not shift by negative or excessive bit count

- **Category:** Advisory | **Decidability:** Decidable | **Scope:** Module
- **Tags:** numerics, reduce-human-error, maintainability, surprising-behavior, subset

### Rule

Do not shift an expression by a negative number of bits or by a value greater than or equal to the bitwidth of the left operand. This is the strict subsetting version — use `checked_shl`/`checked_shr` instead.

Note: `wrapping_shl`, `unbounded_shr`, and `overflowing_shl` are all **non-compliant** because they mask or silently handle out-of-range shifts rather than detecting them.

### Rationale

Out-of-range shifts are nonsensical expressions that typically indicate a logic error. Based on CERT C INT34-C.

### Non-Compliant Examples

Direct shift that panics:

```rust
let bits: u32 = 61;
let shifts = vec![-1, 4, 40];
for sh in shifts {
    println!("{bits} << {sh} = {:?}", bits << sh); // panics on -1 and 40
}
```

Using `wrapping_shl` (masks the problem):

```rust
println!("{}", bits.wrapping_shl(40)); // non-compliant: silently masks
```

### Compliant Example

**Checked shifts** — return `None` on invalid shift:

```rust
let bits: u32 = 61;
let shifts = vec![4, 40];
for sh in shifts {
    println!("{bits} << {sh} = {:?}", bits.checked_shl(sh));
}
```

---

## gui_LvmzGKdsAgI5 — Avoid out-of-range shifts

- **Category:** Mandatory | **Decidability:** Undecidable | **Scope:** Module
- **Tags:** numerics, surprising-behavior, defect

### Rule

Avoid shifting by negative values or by values greater than or equal to the bit width of the left operand. This is the less strict but undecidable version of gui_RHvQj8BHlz9b — it allows the `<<`/`>>` operators but requires proving the shift amount is in range.

### Rationale

Out-of-range shifts indicate a logic error. Based on CERT C INT34-C.

### Non-Compliant Example

```rust
let bits: u32 = 61;
let shifts = vec![-1, 4, 40];
for sh in shifts {
    println!("{bits} << {sh} = {:?}", bits << sh); // panics on -1 and 40
}
```

### Compliant Examples

**Range check before shift:**

```rust
let bits: u32 = 61;
let shifts = vec![-1, 0, 4, 40];
for sh in shifts {
    if sh >= 0 && sh < 32 {
        println!("{bits} << {sh} = {}", bits << sh);
    }
}
```

**Using `overflowing_shl` with overflow check:**

```rust
fn safe_shl(bits: u32, shift: u32) -> u32 {
    let (result, overflowed) = bits.overflowing_shl(shift);
    if overflowed {
        0
    } else {
        result
    }
}
```

---

## gui_iv9yCMHRgpE0 — Do not convert integer to invalid pointer

- **Category:** TBD | **Decidability:** Undecidable | **Scope:** System
- **Tags:** defect, undefined-behavior

### Rule

An expression of numeric type shall not be converted to a pointer if the resulting pointer is incorrectly aligned, does not point to an entity of the referenced type, or is an invalid representation.

### Rationale

The mapping between pointers and integers must be consistent with the addressing structure of the execution environment. Manipulating pointer values as integers may discard address space information.

### Non-Compliant Example

```rust
fn f1(flag: u32, ptr: *const u32) {
    let mut rep = ptr as usize;
    rep = (rep & 0x7fffff) | ((flag as usize) << 23);
    let _p2 = rep as *const u32; // invalid pointer manipulation
}
```

### Compliant Example

Use a struct to store pointers alongside metadata:

```rust
struct PtrFlag {
    pointer: *const u32,
    flag: u32,
}

fn f2(flag: u32, ptr: *const u32) {
    let _ptrflag = PtrFlag {
        pointer: ptr,
        flag: flag,
    };
}
```
