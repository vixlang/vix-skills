---
name: vix-compiler
description: >
  Work with the Vix compiler codebase — a systems language with
  Rust-like ownership/borrowing, Hindley-Milner type inference,
  algebraic data types, generics, and LLVM codegen.
---

# Vix Compiler

## Overview

Vix is a systems programming language with automatic memory management via
ownership/borrowing (inspired by Rust), Hindley-Milner type inference,
algebraic data types (`Option`/`Result`), generics, `match` expressions,
and LLVM-based code generation.

The compiler has **two implementations**:
- **C/C++ (`src/`)**: Production compiler — parsing, type checking,
  ownership checking, LLVM codegen, linking.
- **Vix bootstrap (`bootstrap/src/`)**: Self-hosting compiler written in Vix
  itself. Has its own AST, parser, and MIR/asm backend.
  **Does NOT** have ownership checking (yet).

## Architecture

```
src/
├── main.c                   → Entry point, CLI, pipeline orchestration
├── parser/
│   ├── parser.y             → Bison grammar (LALR, 209 S/R conflicts)
│   └── lexer.l              → Flex lexer
├── ast/
│   └── ast.c                → AST node construction
├── semantic/
│   └── semantic.c           → Symbol table, name resolution, scope checks
├── Typeck/                  → Type checker (C++)
│   ├── Typeck.cpp           → Hindley-Milner type inference
│   ├── TypeckInfer.cpp      → Type inference helpers
│   └── LayOut.cpp           → Struct memory layout computation
├── Ownership/
│   └── Ownership.cpp        → Transaction-based ownership & borrow checker
├── compiler/                → LLVM codegen (C++)
│   ├── Codegen.cpp          → Main codegen driver
│   ├── Exprs.cpp            → Expression codegen
│   ├── Stmts.cpp            → Statement codegen
│   ├── Funcs.cpp            → Function codegen
│   ├── Structs.cpp          → Struct codegen
│   ├── Arrays.cpp           → Array codegen
│   ├── Types.cpp            → LLVM type mapping
│   ├── Passes.cpp           → LLVM optimization pipeline
│   ├── Attrs.cpp            → Source attribute parsing (`#[no_main]`, etc.)
│   ├── Llc/Llc.cpp          → Assembly/object file output
│   └── Linker/Linker.cpp    → lld-based linking
├── utils/
│   ├── error.c              → Error reporting with source context & ^ underlines
│   └── compat.h             → Platform compatibility macros
include/
├── ast.h, compiler.h, codegen.h, parser.h, type.h, typeck.h, ...
└── ownership.h              → C API for ownership checker
```

## Build

```bash
# CMake-based build (recommended, requires LLVM 18):
cmake -B build -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_PREFIX_PATH=/usr/lib/llvm-18
cmake --build build -j$(nproc)

# Manual compile (for quick iteration on a single file, e.g. Ownership):
g++ -std=c++20 -I/usr/lib/llvm-18/include -Iinclude -Isrc/compiler \
  -Ibuild/parser \
  -D_GNU_SOURCE -D__STDC_CONSTANT_MACROS -D__STDC_FORMAT_MACROS \
  -D__STDC_LIMIT_MACROS \
  -fstack-protector-strong -fexceptions \
  -c src/Ownership/Ownership.cpp -o Ownership.o
# Then relink with all the other .o files
```

## LLVM Version Handling

Supports **LLVM 18** (Linux/macOS) and **LLVM 22** (Windows/MSYS2).
API differences handled via `LLVM_VERSION_MAJOR`:

```cpp
#include <llvm/Config/llvm-config.h>
#if LLVM_VERSION_MAJOR >= 22
    module->setTargetTriple(targetTriple);    // takes Triple
#else
    module->setTargetTriple(targetTriple.str());  // takes StringRef
#endif
```

LLVM 18: `setTargetTriple(StringRef)`, `createTargetMachine(StringRef, ...)`
LLVM 22+: `setTargetTriple(Triple)`, `createTargetMachine(Triple, ...)`

LLVM headers included as SYSTEM to suppress internal warnings:
```cmake
include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})
```

## Language Features

### Ownership & Borrowing (`src/Ownership/`)

Uses a **transaction-based** system for branch analysis:

1. Before entering a branch (if/else), call `begin_transaction()`
2. Evaluate the branch code
3. Capture outer variable states with `capture_outer_branch_state()`
4. Rollback with `rollback_transaction()`
5. Merge states from both branches (4 dimensions: moved, borrowed_shared,
   borrowed_mut, borrow_sources)

Key rules:
- Non-copy types (`string`, `[i32]`, structs with non-copy fields) are **moved** on assignment/argument passing
- `ref x` creates a shared (immutable) borrow — multiple allowed
- `mut ref x` creates an exclusive (mutable) borrow — only one at a time
- After borrow ends, the original variable is usable again
- `let mut ptr = mut ref x` for writing through a pointer (`@ptr = val`)

### Type System (`src/Typeck/`)

Hindley-Milner type inference with unification:
- Primitives: `i8`, `i32`, `i64`, `f32`, `f64`, `bool`, `string`, `ptr`
- Structs: `type Point = struct { x: i32, y: i32 }`
- Arrays: `[i32]` (dynamic), `[i32; 5]` (fixed-size, via `AST_TYPE_FIXED_SIZE_LIST`)
- Option/Result: `?string` (via `Some`/`None`, `Ok`/`Err`)
- Generics: `fn id:[T](value: T): T { return value }`
- Match: `match x { Some(v) -> ..., None -> ... }`
- Traits/impl: `impl Point { fn area(self): i32 { ... } }` (bootstrap)

## Testing

```bash
# Run error/diagnostic tests (no linking needed):
cd tests && python3 -m pytest errors.py -v

# Run a single test file:
cd tests && python3 -m pytest errors.py::TestErrorCodes -v

# Run all non-integration tests:
cd tests && python3 -m pytest -v -x -m "not slow and not fuzz and not stress and not cli and not integration and not feature"
```

Test infrastructure:
- `tests/helpers.py` — `compile_vix()`, `compile_and_run()`, `compile_source()`
- `tests/errors.py` — error detection tests (16+ tests)
- `tests/test_types.py` — type system tests (108 tests)
- `tests/regression/` — 300+ regression `.vix` files

## Error Reporting

Errors use a Rust-like format with error codes (E001–E008):

```
error [SemanticError E006]: use of moved value 'buf'
  --> mirror.vix:51:12
   |
50 |     snprintf(buf, 64, "%d", n)
51 |     return buf
   |            ^^^
52 | }
   |
   = note: 'buf' was moved here at mirror.vix:50
```

- Column auto-corrected by `adjust_column_to_identifier()` (searches source line)
- Length set to variable name length for precise `^^^` highlighting
- Error codes shipped via `error_type_code()` in `error.c`
- Notes from `\n`-separated lines in error messages

## CI

GitHub Actions: Linux (LLVM 18) + macOS (LLVM 18) + Windows (LLVM 22).

CI pipeline:
1. Build with CMake
2. Smoke test: `examples/hello.vix --check`
3. Ownership check: `pointer.vix`, `fib.vix`, `struct2.vix` + `owner.vix` expects failure
4. Pytest on `tests/errors.py`
5. Package as `tar.gz` / `zip`

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| `const char[]` + `const char*` | `invalid operands to binary '+'` | Wrap first string in `std::string()` |
| `llvm::Triple` passed to `StringRef` API | `cannot convert Triple to StringRef` | Add `.str()` for LLVM < 22 |
| `getTargetTriple()` returns string (LLVM 18) | `has no member named 'str'` | Don't call `.str()`, it is already a string |
| `rfind('\'')` for variable name | `^^^` covers entire rest of line | Use `find('\'', start+1)` instead |
| Missing `UPDATE_COLUMN()` in lexer IDENTIFIER rule | `^` points at keyword, not variable | Add `UPDATE_COLUMN()` before `GET_FIRST_COLUMN()` |
| `GET_FIRST_COLUMN()` formula wrong | All columns shifted | Should be `yycolumn - yyleng`, not `- yyleng + 1` |
| CI: `ld.lld` can't find `libstdc++` | `unable to find library -lstdc++` | Install `libstdc++-12-dev` or skip `-o` tests |
| CI: APT LLVM version mismatch | Random CI failures | Pin to LLVM 18 via `llvm.sh 18` |
| CMake `if(WIN32)` for `YY_NO_UNISTD_H` | MSYS2 can't find `isatty` | Use `if(MSVC)` instead (MSYS2 has `<unistd.h>`) |
| CMake `if(NOT WIN32)` for `_POSIX_C_SOURCE` | MSYS2 can't find POSIX functions | Use `if(NOT MSVC)` instead |

## Coding Conventions

- C files: `snake_case` functions, C++ files: `camelCase`
- Ownership checker: methods defined out-of-line (`OwnershipChecker::method`)
- Error messages: lowercase, no trailing punctuation
- Type names in error messages: single-quoted (`'buf'`)
- LLVM conditional code: `#if LLVM_VERSION_MAJOR >= 22` guards
- No `-Werror` in CMakeLists (warnings stay warnings)
- License header: Apache 2.0 on all source files
