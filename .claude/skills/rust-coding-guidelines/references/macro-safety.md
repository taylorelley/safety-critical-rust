# Macro Safety Guidelines

## gui_2jjWUoF1teOY — Do not use macros in place of functions

- **Category:** Mandatory | **Decidability:** Decidable | **Scope:** System
- **Tags:** reduce-human-error

### Rule

Functions should always be preferred over macros, except when macros provide essential functionality that functions cannot, such as variadic interfaces, compile-time code generation, or syntax extensions.

### Rationale

- **Debugging:** Errors in macro expansions are harder to trace than function errors
- **Optimization:** Macros inhibit compiler optimizations; they act like `#[inline(always)]` causing code bloat
- **Clarity:** Functions have clear type signatures, predictable behavior, and proper stack traces

### Non-Compliant Example

Using a macro where a function suffices — hidden mutation:

```rust
macro_rules! increment_and_double {
    ($x:expr) => {
        {
            $x += 1; // mutation is implicit
            $x * 2
        }
    };
}

fn main() {
    let mut num = 5;
    let result = increment_and_double!(num);
    println!("Result: {}, Num: {}", result, num);
    // Result: 12, Num: 6 — mutation was hidden
}
```

### Compliant Example

Function with explicit borrowing:

```rust
fn increment_and_double(x: &mut i32) -> i32 {
    *x += 1; // mutation is explicit
    *x * 2
}

fn main() {
    let mut num = 5;
    let result = increment_and_double(&mut num);
    println!("Result: {}, Num: {}", result, num);
}
```

---

## gui_SJMrWDYZ0dN4 — Use fully qualified paths in macro definitions

- **Category:** Required | **Decidability:** Decidable | **Scope:** Module
- **Tags:** reduce-human-error

### Rule

Each name inside a macro definition shall either use a global path (e.g., `::std::vec::Vec`) or a path prefixed with `$crate`.

### Rationale

Relative paths inside macros are subject to path resolution that may change depending on where the macro is invoked. The intended entity can be shadowed, leading to unexpected behavior or developer confusion.

### Non-Compliant Example

```rust
#[macro_export]
macro_rules! my_vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new(); // non-global path — could be shadowed
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

### Compliant Example

```rust
#[macro_export]
macro_rules! my_vec_global {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = ::std::vec::Vec::new(); // global path
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

---

## gui_13XWp3mb0g2P — Do not use attribute macros

- **Category:** Required | **Decidability:** Decidable | **Scope:** System
- **Tags:** reduce-human-error

### Rule

Attribute macros shall neither be declared nor invoked. Prefer less powerful macros that only extend source code.

### Rationale

Attribute macros can rewrite items entirely or in unexpected ways, causing confusion and introducing errors.

### Non-Compliant Example

```rust
#[test] // non-compliant: attribute macro rewrites the item
fn example_test() {
    assert!(true);
}
```

### Compliant Example

Avoid attribute macros; use plain functions or declarative macros where possible.

---

## gui_66FSqzD55VRZ — Prefer declarative over procedural macros

- **Category:** Advisory | **Decidability:** Decidable | **Scope:** Crate
- **Tags:** readability, reduce-human-error

### Rule

Macros should be expressed using declarative syntax (`macro_rules!`) in preference to procedural syntax.

### Rationale

Procedural macros are not restricted to pure transcription and can contain arbitrary Rust code. They can have arbitrary side effects, exhaust compiler resources, or expose vulnerabilities.

---

## Planned Guidelines (Not Yet Finalized)

The following guidelines have been proposed but do not yet have actionable content. Be aware of their intent but do not fabricate compliance rules for them:

| ID | Title | Category |
|---|---|---|
| gui_h0uG1C9ZjryA | Shall not use declarative macros | Mandatory |
| gui_WJlWqgIxmE8P | Shall not use function-like macros | Mandatory |
| gui_a1mHfjgKk4Xr | Shall not invoke macros | Mandatory |
| gui_8hs33nyp0ipX | Shall ensure complete hygiene of macros | Mandatory |
| gui_uuDOArzyO3Qw | Shall not write code that expands macros | Mandatory |
| gui_FRLaMIMb4t3S | Do not hide unsafe blocks within macro expansions | Required |

These planned mandatory guidelines indicate a strong direction toward macro-free safety-critical Rust. When writing new safety-critical code, minimize macro usage in anticipation of these stricter rules.
