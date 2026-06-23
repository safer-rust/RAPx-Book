# Chapter 7. Optimization

This module identifies performance bottlenecks and inefficiencies using static analysis methods. The implementation lives at [`rapx/src/check/opt/`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/check/opt/).

## Usage

To detect performance bugs:

```shell
cargo rapx opt
```

RAPx outputs a summary of detected inefficiencies by category, along with detailed source locations and suggested improvements.

## Architecture

The opt module operates in three steps:

1. **Dataflow Graph Construction**: Builds function-level dataflow graphs via `DataflowAnalyzer`.
2. **Pattern Matching**: Each checker walks the dataflow graph to find known inefficiency patterns.
3. **Reporting**: Detected issues are reported via `rap_warn!` with annotated source snippets, and a summary is logged via `rap_info!` with per-category counts.

All checkers run together; there is currently no way to select individual checks.

## Categories

The module covers six categories of performance checks:

| Category | Count Reported As | Focus |
|----------|-------------------|-------|
| Bounds Checking | `Bounds Checking` | Unnecessary bounds checks in loops |
| Encoding Checking | `Encoding Checking` | Inefficient string/byte encoding patterns |
| Cloning | `Cloning` | Unnecessary clone/memory duplication |
| Suboptimal | `Suboptimal` | Suboptimal collection/algorithm usage |
| Initialization | `Initialization` | Inefficient collection initialization |
| Reallocation | `Reallocation` | Missing `reserve` / unnecessary reallocation |

## Test Cases

The [`tests/opt/`](https://github.com/safer-rust/RAPx/blob/main/rapx/tests/opt/) directory contains three end-to-end test cases:

### bounds_len — Bounds Checking

```rust
fn foo(mut a: Vec<i32>) {
    for i in 0..a.len() {
        a[i] = a[i] + 1;
    }
}

fn foo1(input: &[u8]) {
    let mut index = 0;
    let len = input.len();
    while index < len {
        let b = input[index];
        index += 1;
    }
}
```

RAPx detects that the index `i` is provably within `[0, a.len())` and the `while` loop's `index < len` guard ensures `index` is always in bounds. The bounds checks on `a[i]` and `input[index]` are redundant.

Expected output: `Bounds Checking: 2`.

### encoding_check — Encoding Checking

```rust
use std::str;

fn foo_1(mut n: u128, base: usize, output: &mut String) {
    const BASE_64: &[u8; 64] = b"0123456789abcdef...ABCDEFGHIJKLMNOPQRSTUVWXYZ@$";
    let mut s = [0u8; 128];
    let mut index = s.len();
    let base = base as u128;
    loop {
        index -= 1;
        s[index] = BASE_64[(n % base) as usize];
        n /= base;
        if n == 0 {
            break;
        }
    }
    output.push_str(str::from_utf8(&s[index..]).unwrap());
}

static CHARS: &[u8; 5] = b"12345";

fn foo_2() -> String {
    let mut v = Vec::with_capacity(12);
    v.push(CHARS[2 as usize]);
    v.push(CHARS[3 as usize]);
    v.push(b' ');
    String::from_utf8_lossy(&v).to_string()
}
```

RAPx detects two encoding inefficiencies: manual base-64 encoding with `String::push_str` that could use a more efficient approach, and character-by-character `Vec::push` before UTF-8 conversion.

Expected output: `Encoding Checking: 2`.

### hash_key_cloning — Memory Cloning

```rust
use std::collections::HashSet;

fn foo(a: &Vec<String>) {
    let mut b = HashSet::new();
    for i in a {
        let c = i.clone();
        b.insert(c);
    }
}
```

RAPx detects that `i.clone()` produces a value that is only used as a `HashSet` key. The original borrowed value could be used instead, avoiding an unnecessary allocation.

Expected output: `Cloning: 1`.

## Detailed Module Reference

### Bounds Checking (`checking/bounds_checking/`)

Rust performs automatic bounds checking on indexed accesses for safety. In performance-critical loops, these checks can be eliminated when the index is provably within bounds.

Sub-modules:
- **`bounds_len.rs`**: Analyzes index-vs-length relationships to prove bounds checks are redundant.
- **`bounds_extend.rs`**: Detects cases where `Vec::extend` or `Vec::extend_from_slice` would be more efficient than element-by-element push.
- **`bounds_loop_push.rs`**: Detects patterns where `for` loop iteration combined with `push` can be replaced with bulk operations.

### Encoding Checks (`checking/encoding_checking/`)

Detects inefficient encoding patterns in string and byte manipulation.

Sub-modules:
- **`array_encoding.rs`**: Detects inefficient array-to-slice encoding patterns.
- **`string_lowercase.rs`**: Detects manual case conversion patterns that should use `to_lowercase()` or `to_ascii_lowercase()`.
- **`string_push.rs`**: Detects repeated `String::push` in loops that could use `push_str`, `write!`, or `extend`.
- **`vec_encoding.rs`**: Detects inefficient vector encoding patterns like repeated `push` when bulk operations are available.

### Memory Cloning (`memory_cloning/`)

Detects unnecessary clone operations where a cloned value is only used immutably.

Sub-modules:
- **`used_as_immutable.rs`**: Detects cloned values that are only used in immutable contexts.
- **`hash_key_cloning.rs`**: Specifically targets clones used as keys in hash-based collections where the original borrowed value could be used instead.

**Note**: Developers need to manually verify whether removing the clone is semantically safe, as clone removal may change ownership semantics.

### Data Collection (`data_collection/`)

Detects suboptimal collection choices and suggests alternatives with better algorithmic complexity.

#### Initialization (`initialization/`)
- **`local_set.rs`**: Suggests `HashSet` initialization patterns.
- **`vec_init.rs`**: Detects `Vec` initialization that can be optimized with `vec![]` or `with_capacity`.

#### Reallocation (`reallocation/`)
- **`flatten_collect.rs`**: Suggests `flatten().collect()` instead of nested iteration.
- **`unreserved_hash.rs`**: Detects `HashMap`/`HashSet` usage without `reserve`.
- **`unreserved_vec.rs`**: Detects `Vec` usage without `reserve` before known-size population.

#### Suboptimal (`suboptimal/`)
- **`participant.rs`**: Detects repeated collection traversal patterns.
- **`slice_contains.rs`**: Detects `Vec::contains` on large vectors where `HashSet` would be faster.
- **`vec_remove.rs`**: Detects `Vec::remove` in loops (O(n²) when `swap_remove` would be O(n)).

### Iterator (`iterator/`)

The iterator check (`next_iterator.rs`) detects manual `for` loops that can be expressed as iterator chains, enabling the compiler to apply loop fusion and other optimizations.

```rust
fn foo(data: &[i32]) -> Vec<i32> {
    let mut result = Vec::new();
    for &item in data {
        result.push(item * 2);
    }
    result
}
```

RAPx suggests using iterator combinators (`map`, `collect`):

```rust
fn foo(data: &[i32]) -> Vec<i32> {
    data.iter().map(|&item| item * 2).collect()
}
```
