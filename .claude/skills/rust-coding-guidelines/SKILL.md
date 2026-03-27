---
name: rust-coding-guidelines
description: Safety-critical Rust coding guidelines based on MISRA Compliance 2020. Use when creating, reviewing, or designing Rust code for safety-critical applications. Applies to safety-critical Rust, MISRA compliance, unsafe code review, numeric safety, FFI safety.
allowed-tools: Read, Grep, Glob
---

# Safety-Critical Rust Coding Guidelines

You are an expert in safety-critical Rust development. Apply these guidelines derived from the Rust Foundation's Coding Guidelines Subcommittee, which follow MISRA Compliance 2020 categorization and target compliance with IEC 61508, ISO 26262, and DO-178C.

## Compliance Categories

Each guideline has a **category** that determines enforcement:

| Category | Meaning | Deviation |
|---|---|---|
| **Mandatory** | Must be followed. No deviation permitted. | None |
| **Required** | Must be followed. Formal per-instance deviation allowed. | Per-instance with documented justification |
| **Advisory** | Should be followed as far as reasonably practical. | Project-wide deviation acceptable |

## Modes

Select your mode based on `$ARGUMENTS` or infer from context:

### `create` — Writing New Code
1. Scan the quick-reference table below to identify all applicable guidelines
2. Read the relevant reference file(s) from `${CLAUDE_SKILL_DIR}/references/`
3. Apply compliant patterns proactively as you write code
4. Add `// SAFETY:` comments for every `unsafe` block
5. Prefer checked/saturating arithmetic over raw operators

### `review` — Reviewing Existing Code
1. Scan the quick-reference table to identify all potentially applicable guidelines
2. Read the relevant reference file(s) for detailed rules and examples
3. Check each applicable guideline against the code
4. Report findings in the output format below
5. Prioritize: mandatory > required > advisory

### `design` — Architecture and Design
1. Apply structural guidelines: no recursion, bounded resource usage
2. Use strong types (newtypes) for logically distinct values
3. Minimize `unsafe` surface area; isolate FFI at module boundaries
4. Design error handling with `Result` types, not panics
5. Read relevant reference files for specific patterns

## Quick Reference — All Guidelines

### Numeric Safety → `references/numeric-safety.md`

| ID | Rule | Category | Decidable |
|---|---|---|---|
| gui_dCquvqE1csI3 | Ensure integer operations do not overflow | Required | Yes |
| gui_7y0GAMmtMhch | Do not use integer type as divisor in integer division | Advisory | Yes |
| gui_kMbiWbn8Z6g5 | Do not divide by zero | Required | No |
| gui_ADHABsmK9FXz | Do not use `as` with numeric operands | Advisory | Yes |
| gui_RHvQj8BHlz9b | Do not shift by negative or excessive bit count | Advisory | Yes |
| gui_LvmzGKdsAgI5 | Avoid out-of-range shifts | Mandatory | No |
| gui_iv9yCMHRgpE0 | Do not convert integer to invalid pointer | TBD | No |

### Type Safety → `references/type-safety.md`

| ID | Rule | Category | Decidable |
|---|---|---|---|
| gui_0cuTYG8RVYjg | Ensure union field reads produce valid values | Required | No |
| gui_xztNdXA2oFNC | Use strong types for logically distinct values | Advisory | No |
| gui_HDnAZ7EZ4z6G | Do not use `as _` in pointer casts | Required | Yes |
| gui_PM8Vpf7lZ51U | Do not convert integer to pointer | TBD | Yes |

### Macro Safety → `references/macro-safety.md`

| ID | Rule | Category | Decidable |
|---|---|---|---|
| gui_2jjWUoF1teOY | Do not use macros in place of functions | Mandatory | Yes |
| gui_SJMrWDYZ0dN4 | Use fully qualified paths in macro definitions | Required | Yes |
| gui_13XWp3mb0g2P | Do not use attribute macros | Required | Yes |
| gui_66FSqzD55VRZ | Prefer declarative over procedural macros | Advisory | Yes |
| gui_h0uG1C9ZjryA | Do not use declarative macros *(planned)* | Mandatory | Yes |
| gui_WJlWqgIxmE8P | Do not use function-like macros *(planned)* | Mandatory | Yes |
| gui_a1mHfjgKk4Xr | Do not invoke macros *(planned)* | Mandatory | Yes |
| gui_8hs33nyp0ipX | Ensure complete macro hygiene *(planned)* | Mandatory | Yes |
| gui_uuDOArzyO3Qw | Do not write code that expands macros *(planned)* | Mandatory | Yes |
| gui_FRLaMIMb4t3S | Do not hide unsafe in macro expansions *(planned)* | Required | TBD |

### Unsafe and FFI → `references/unsafe-and-ffi.md`

| ID | Rule | Category | Decidable |
|---|---|---|---|
| gui_ZDLZzjeOwLSU | Assure visibility of `unsafe` keyword | Required | Yes |

### Control Flow and Structure → `references/control-flow-and-structure.md`

| ID | Rule | Category | Decidable |
|---|---|---|---|
| gui_ot2Zt3dd6of1 | Do not use recursive functions | Required | No |

## Routing

Based on the code you're working with, read the appropriate reference file(s):

| If the code involves... | Read |
|---|---|
| Arithmetic, division, remainder, overflow, shifts, `as` casts on numbers | `references/numeric-safety.md` |
| Unions, newtypes, type aliases, pointer casts, `as _`, `transmute` | `references/type-safety.md` |
| `macro_rules!`, proc macros, derive macros, attribute macros, `#[test]` | `references/macro-safety.md` |
| `unsafe` blocks, `extern`, FFI, `#[no_mangle]`, `#[export_name]`, `// SAFETY:` | `references/unsafe-and-ffi.md` |
| Recursion, stack depth, control flow, loop bounds | `references/control-flow-and-structure.md` |

For a comprehensive review, read all reference files. For targeted work, read only the relevant ones.

## Output Format

When reporting guideline violations or applying guidelines, use this format:

```
**[gui_ID]** Category: CATEGORY | Title
- Location: file:line (or description of where)
- Issue: What violates the guideline
- Fix: How to make it compliant
```

When writing new code, note which guidelines you applied as brief inline comments only where non-obvious.
