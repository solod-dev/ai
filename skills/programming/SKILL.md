---
name: programming-solod
description: Writes programs in the Solod (So) language - a strict subset of Go that transpiles to C. Use when the user asks to write So code, work with So packages, create So programs, or asks about So language features. Covers the type system, memory model, concurrency, C interop, standard library, and key differences from Go.
---

# Programming in Solod (So)

So is a strict subset of Go that transpiles to C11. All So code is syntactically valid Go. If a feature isn't listed here, it's not supported. This skill covers So 0.3.

## The `so` command

The `so` CLI tool transpiles and compiles So programs. Make sure it is installed before use: `go install solod.dev/cmd/so@latest`.

Commands:

- `so run <package> [args]` - transpile, compile, and run a package. Example: `so run .` or `so run ./myapp`.
- `so build <package>` - compile a package to an executable. Use `-o <file>` to set output name (default: package directory basename).
- `so test <package>` - run the tests in the package's `test` subdirectory.
- `so bench <package>` - run the benchmarks in the package's `bench` subdirectory.
- `so translate <package>` - transpile a package to C (generates `.h` and `.c` files). Always use `-o generated` to set output directory to `generated`.
- `so version` - print compiler version.

Build flags, shared by `build`, `run`, `test`, and `bench`:

- `-panic=trace|exit|abort` - how a panic terminates the program. `trace` (default) prints a symbolized backtrace then exits 1. Use `exit` on musl and `abort` when debugging or running under a sanitizer.
- `-sanitize` - enable C sanitizers (`address,undefined` by default; `-sanitize=address` picks a set). Adds `-g` and frame pointers. Pair with `-panic=abort`.
- `-track-source` - report So source locations in panic messages instead of C ones.

The C compiler comes from `CC` (default `cc`), with `CFLAGS` and `LDFLAGS` passed through.

A So project is a standard Go module. Create one with `go mod init <name>`, then add the So standard library dependency: `go get solod.dev@latest`. Third-party So packages (via `go install` or vendoring) and multi-module projects are supported.

Targets: native 64-bit and 32-bit, WebAssembly (WASI), and freestanding mode (no libc dependency).

## Key restrictions vs Go

Not supported: goroutines and channels (use `so/conc` instead), function literals and closures, iterators, non-extern generic types, generic methods (except `//so:inline` ones), `recover`, `fallthrough`, type switches, interface-to-interface assertions, `delete` on maps, dynamic errors, named return values, labeled `continue`, method values, struct comparison with `==`, and slices of arrays.

Multiple assignment can't have the same variable on both sides (`a[0], a[2] = a[2], a[0]`); use a temporary. A local variable can't shadow itself (`x := x`).

IMPORTANT: So is a manually managed language. Make sure you avoid typical memory bugs (use-after-free, double-free, memory leaks, etc.) when writing So programs.

## Type mapping

| Go type   | C type              | Notes                                            |
| --------- | ------------------- | ------------------------------------------------ |
| `int`     | `so_int` (int64_t)  |                                                  |
| `byte`    | `so_byte` (uint8_t) |                                                  |
| `rune`    | `so_rune` (int32_t) |                                                  |
| `float64` | `double`            |                                                  |
| `float32` | `float`             |                                                  |
| `bool`    | `bool`              |                                                  |
| `string`  | `so_String`         | struct: `{ const char* ptr; so_int len; }`       |
| `[]T`     | `so_Slice`          | struct: `{ void* ptr; so_int len; so_int cap; }` |
| `map[K]V` | `so_Map*`           | pointer to stack-allocated hash table            |
| `error`   | `so_Error`          | regular interface, as in Go                      |
| `any`     | `void*`             | not an interface - just a pointer                |
| `nil`     | `NULL`              |                                                  |

All variables are zero-initialized as in Go.

Go's `unsafe` package is available for `Sizeof`, `Alignof`, `Add`, `Pointer`, `String`, `StringData`, `Slice`, and `SliceData` (no `Offsetof`). The `min`, `max`, `len`, `cap`, `copy`, `clear`, `new`, `make`, `append`, `panic`, `print`, and `println` builtins all work.

### Reserved C names

Local variables and parameters whose names collide with C keywords or macros (`long`, `bool`, `register`, ...) are mangled automatically with a trailing underscore. The same names as struct fields or package-level declarations are rejected at compile time, so rename them.

## Strings

`so_String` with `ptr` and `len`. Literals wrapped in `so_str()`.

- Indexing returns a byte. Range iteration decodes UTF-8 runes.
- Slicing is zero-copy.
- `[]byte(string)` and `string([]byte)` are zero-copy (alias original data).
- `[]rune(string)` and `string([]rune)` allocate (stack).
- `string(byte)` and `string(rune)` allocate (stack).
- `+` and `+=` concatenation works but allocates on stack. For large/heap strings, use `so/strings.Builder`.

## Arrays

Plain C arrays. Value types on struct assignment (memcpy).

- Decay to pointers when passed to functions, so a callee mutates the caller's array (no value semantics on calls).
- Comparable with `==` and `!=` (unlike structs).
- `len()` and `cap()` are compile-time constants.
- Slicing an array produces a `so_Slice`.
- A returned array becomes a pointer to that array in C, so never return a local one - return the parameter or a heap slice instead. Multi-dimensional arrays can't be returned at all.
- Slices of arrays (`[][3]int`) are not supported. Use a slice of slices or wrap the array in a struct.

## Slices

`so_Slice` with `ptr`, `len`, `cap`. Stack-allocated by default.

- `make([]T, len)` and `make([]T, len, cap)` allocate on the stack.
- `append()` panics if capacity exceeded - no automatic reallocation.
- For heap allocation and dynamic growth, use `so/slices` package.
- Nil slice and empty slice are the same: `(so_Slice){0}`.
- `clear` zeros elements (length/capacity unchanged). Not supported for maps.
- Full slice expressions `s[low:high:max]` supported.

## Maps

Fixed-size, stack-allocated, pointer-based (`so_Map*`). MSI (mask-step-index) hash tables.

- Capacity argument required: `make(map[K]V, cap)`.
- Setting a key when full panics. No `delete`. No resize.
- Compound assignment `m[k] += 1` not supported.
- Cannot return maps from functions (stack-allocated data becomes dangling).
- For dynamic/heap maps, use `so/maps` package.
- Supported key types: integers, `bool`, `float32`, `float64`, `string`, pointers.

## Functions

```go
func sum(a, b int) int { return a + b }
```

- No closures or function literals. Use named functions and function types instead.
- Function values are supported: pass a named function where a `func(...)` type is expected.
- Anonymous function types work as parameter and variable types (`func apply(n int, f func(int) int) int`), but not as return types - use a named type like `type CalcFunc func(int) int` there.
- Exported (capitalized) -> `package_Func` in C. Unexported -> `static`.
- Exported functions must only use exported types in signatures.

### Multiple return values

Only two-value returns in the following patterns:

**`(T, error)` pattern:**

```go
func divide(a, b int) (int, error) {
    if b == 0 { return 0, ErrDivByZero }
    return a / b, nil
}
```

Supported T types: `bool`, `float32`, `float64`, `int`, `int32`, `int64`, `uint`, `uint16`, `uint32`, `uint64`, `rune`, `byte`, `string`, `[]T`, `*T`.

**Custom struct results** for `(StructType, error)`:

```go
func open(name string) (File, error) { ... }
```

The compiler auto-generates the `{T}Result` struct (here `FileResult`). Auto-generation works only if at least one function in `T`'s package returns `(T, error)`. Otherwise, define the result struct `{T}Result` manually.

**`(T1, T2)` pattern** for two values of the supported types:

```go
func divmod(a, b int) (int, int) { return a / b, a % b }
```

Supported T1/T2 types: `bool`, `float64`, `int`, `int64`, `uint`, `uint32`, `uint64`, `byte`, `rune`, `string`.

Arrays can't appear in a multi-return.

### Variadic functions

Standard `...T` syntax. Can spread slices with `s...` (except when calling extern C functions).

## Structs

Standard Go struct syntax. Supports anonymous structs and inner struct fields.

`new()` works with types and values:

```go
p := new(point)         // *point, zero-initialized
p2 := new(point{1, 2})  // *point with values
n := new(42)            // *int with value 42
```

Anonymous structs are only allowed as local variables and as inner struct fields. Elsewhere (slice elements, params, returns) use a named type. Struct comparison with `==` is not supported.

## Methods

- Pointer receivers: `void* self` in C, cast internally.
- Value receivers: struct passed by value.
- Method expressions (`T.method`) supported; method values (`v.method`) not supported.

## Interfaces

Struct with `void* self` + function pointers. No runtime type info.

- Methods must use pointer receivers.
- Convert to interface via pointer: `var s Shape = &rect`.
- Interfaces can be returned from functions and stored in struct fields.
- Type assertions: `_, ok := s.(*Rect)` or `r := s.(*Rect)` (not both).
- Empty interface (`any`/`interface{}`) = `void*`. Only the direct form `v := a.(T)` works on an `any`; the comma-ok form doesn't.
- No interface-to-interface assertions. No type switches.

Because `any` carries no type information, an assertion on it is unchecked. If an `any` holds an interface value, assert it back to the interface type, never to the concrete pointer type inside it.

## Enums

Typed constant groups. Integer and string enums. `iota` supported for integers.

```go
type Day int
const (
    Sunday Day = iota
    Monday
    Tuesday
)
```

## Errors

The `error` type is a regular interface with an `Error() string` method. In practice you use sentinel errors only, defined at package level:

```go
var ErrNotFound = errors.New("not found")
```

- Compared by pointer equality (`==`). O(1) operation.
- No `fmt.Errorf`, no `errors.New` inside functions, no wrapping.

## Panic and defer

`panic()` accepts string literal, string variable, or error. No `recover`.

`defer` works in function scope only, as in Go. If you access a variable in a deferred function, it must be declared at the top level of the function (not inside a block). Otherwise, the variable will be out of scope when the deferred function executes (unlike Go, So can't capture variables in closures).

## Runtime safety

So catches a limited set of errors instead of letting them become undefined behavior:

- **Escape analysis** rejects, at compile time, a function that returns a pointer to a stack-allocated value (`return &Point{x: x}`). It covers common cases only. Allocate through `so/mem` or `so/slices` and return that instead.
- **Assertions** check preconditions the caller must satisfy: slice and string bounds, index out of range, slice-to-array length, zero map capacity, and integer division or modulo by zero. They panic on failure and report a source location. `c.Assert(cond, msg)` adds your own. Defining `NDEBUG` removes all of them, so assertion conditions must be side-effect free.
- **Other runtime checks** (`append` beyond capacity, setting a key in a full map) always panic and ignore `NDEBUG`.
- **Nil pointer dereference** is caught at runtime in POSIX hosted builds and reported as a panic with a backtrace but no source line. Freestanding and Windows builds fault like C.

## Concurrency

There are no goroutines or channels in the language. Use the standard library instead:

- `so/conc` - OS threads, generic channels (buffered or rendezvous), and worker pools.
- `so/sync` - mutexes, condition variables, once-only execution.
- `so/sync/atomic` - lock-free atomic values.

`sync` and `conc` types must be initialized before use, must not be copied, and must be freed. Importing either package links `-lpthread` automatically.

## Packages

Each package -> one `.h` + `.c` pair. Multiple `.go` files merged.

- Exported symbols prefixed with package name; unexported are `static`. `//so:promote` puts an unexported symbol in the header with the package prefix without exporting it in Go.
- Import -> `#include`.
- **Constants and variables are emitted in source order**, so they can't refer to ones declared later. Types are emitted in dependency order, so a type may refer to a type declared below it.
- A recursive type only works if the cycle passes through a struct (`type Node struct { next *Node }` is fine, `type StateFn func() StateFn` is not).
- One `init()` per package allowed (emitted as `__attribute__((constructor))`). With several packages, init order is unspecified.

## Memory model

- **Stack by default:** `make()`, `append()`, `new()`, map literals are all stack-allocated.
- **Heap via stdlib:** `slices.Make`, `slices.Append`, `maps.New`, `mem.Alloc` - all take `mem.Allocator`.
- Prefer passing `mem.System` explicitly (passing `nil` also selects it).
- `mem` also offers an `Arena` (bump allocator over a fixed buffer), a `NoAllocator`, and a `Tracker` that counts allocations to find leaks.
- Maximum stack allocation: 64KB (`so_MaxAllocaSize`).

## Testing

So programs are tested with `so test` and the `so/testing` package. Tests live in a `test` subdirectory of the package under test; benchmarks live in `bench` and run with `so bench`.

```
so/bytes/
    bytes.go
    test/
        bytes.go     # your tests
        main.go      # generated by `so test`, committed
```

Test files are plain `.go` files (no `_test.go` suffix) in `package main`, so `go test` ignores them. Because `test` is a separate package, tests only see the exported API.

```go
package main

import (
	"solod.dev/so/bytes"
	"solod.dev/so/testing"
)

func TestEqual(t *testing.T) {
	if !bytes.Equal([]byte("abc"), []byte("abc")) {
		t.Error("Equal(abc, abc) = false, want true")
	}
}
```

- Run with `so test ./so/bytes`. `-run=TestPrefix` limits the run (plain prefix match, not a regexp).
- After adding, renaming, or removing a `TestXxx` function, re-run `so test` to regenerate `test/main.go`, and commit it.
- `T` provides `Name`, `Log`, `Error`, `Errorf`, `Fatal`, `Fatalf`, `Skip`, `Fail`, `Failed`, and `Allocator`.
- **`Fatal` and `Skip` don't stop the function** (there's no `recover`). Always `return` right after them.
- A hard crash (panic or segfault) aborts the whole run, since all tests share one process.
- Allocate through `t.Allocator()` to have the test fail on leaked memory.

Benchmarks are `BenchmarkXxx(b *testing.B)` functions with the measured code inside a `for b.Loop()` loop. `b.Loop` doesn't keep the body alive, so assign results to a `//so:volatile` package variable or pass the object to `testing.Keep` to survive optimization.

Since So code is also valid Go, `_test.go` files still work with the regular `go test` toolchain (both `so test` and `so bench` ignore them). Functions marked `//so:extern` have Go stub bodies that serve as test doubles: the Go test runner executes these stubs, while the real C implementations are used in the compiled binary.

## Generics

Very limited. Two forms:

1. **Generic extern declarations** - type params become C macro args. Implemented as `#define` macros on C side.
2. **Generic inline macros** (`//so:inline` on generic functions) - auto-generates C macros. Restrictions: single return at end, no defer, args may evaluate multiple times. Prefix locals with `_`.

See [interop.md](interop.md) for details on both forms.

## Additional references

- **C interop and generics**: See [interop.md](interop.md) for `//so:extern`, `//so:include`, `//so:link`, `//so:inline`, `//so:promote`, `//so:embed`, automatic decay, raw C intrinsics, and generic patterns.
- **Standard library**: See [stdlib.md](stdlib.md) for what each `so/*` package is for. Use `go doc solod.dev/so/<pkg>` for API details.
