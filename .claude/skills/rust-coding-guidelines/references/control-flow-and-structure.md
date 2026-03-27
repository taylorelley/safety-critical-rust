# Control Flow and Structure Guidelines

## gui_ot2Zt3dd6of1 — Do not use recursive functions

- **Category:** Required | **Decidability:** Undecidable | **Scope:** System
- **Tags:** stack-overflow

### Rule

Any function shall not call itself directly or indirectly (no direct or mutual recursion).

### Rationale

Recursive functions can easily cause stack overflows, which may result in exceptions or undefined behavior (especially on embedded systems). Although the Rust compiler supports tail call optimization, it is **not guaranteed** and depends on the specific implementation and function structure. Until tail call optimization is guaranteed and stabilized, avoid recursion to prevent stack overflows.

### Non-Compliant Example

Recursive function on a nested data structure:

```rust
enum MyEnum {
    Str(String),
    List(Vec<MyEnum>),
}

fn concat_strings(input: &[MyEnum]) -> String {
    let mut result = String::new();
    for item in input {
        match item {
            MyEnum::Str(s) => result.push_str(s),
            MyEnum::List(list) => result.push_str(&concat_strings(list)), // recursive call
        }
    }
    result
}
```

### Compliant Example

Iterative implementation with explicit stack and bounded depth:

```rust
enum MyEnum {
    Str(String),
    List(Vec<MyEnum>),
}

fn concat_strings_non_recursive(input: &[MyEnum]) -> Result<String, &'static str> {
    const MAX_STACK_SIZE: usize = 1000;
    let mut result = String::new();
    let mut stack = Vec::new();

    stack.extend(input.iter());

    while let Some(item) = stack.pop() {
        match item {
            MyEnum::Str(s) => result.insert_str(0, s),
            MyEnum::List(list) => {
                for sub_item in list.iter() {
                    stack.push(sub_item);
                    if stack.len() > MAX_STACK_SIZE {
                        return Err("Too big structure");
                    }
                }
            }
        }
    }
    Ok(result)
}
```

---

## General Control Flow Principles for Safety-Critical Rust

These principles apply broadly when designing and reviewing safety-critical Rust code:

### Bounded Resource Usage

- All loops must have a provable termination condition
- Prefer iterators with known bounds over open-ended `loop` constructs
- When converting recursive algorithms to iterative ones, add explicit depth/size limits
- Use `const` bounds where possible (e.g., `const MAX_DEPTH: usize = 100`)

### Error Handling

- Use `Result<T, E>` for all fallible operations — do not panic
- Avoid `unwrap()` and `expect()` in production code paths
- Propagate errors up to callers who can make informed decisions
- Define domain-specific error types that carry actionable context

### Predictable Execution

- Avoid dynamic dispatch (`dyn Trait`) where static dispatch suffices
- Prefer stack allocation over heap allocation when sizes are known
- Avoid `String` and `Vec` growth in tight loops — preallocate capacity
- Use `#[must_use]` on functions whose return values should not be silently ignored

### No Panicking in Safety-Critical Paths

In safety-critical code, panicking is an abnormal termination that must be avoided:

- Replace `panic!()`, `todo!()`, `unimplemented!()`, `unreachable!()` with proper error handling
- Replace indexing (`array[i]`) with `array.get(i)` for bounds-checked access returning `Option`
- Use `checked_*`, `saturating_*`, or `wrapping_*` arithmetic instead of bare operators
- Test with `#[cfg(debug_assertions)]` to ensure code behaves consistently in debug and release
