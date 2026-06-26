---
name: programming-solod
description: Writes programs in the Solod (So) language - a strict subset of Go that transpiles to C. Use when the user asks to write So code, work with So packages, create So programs, or asks about So language features. Covers the type system, memory model, C interop, standard library, and key differences from Go.
---

# Programming in Solod (So)

So is a strict subset of Go that transpiles to C11. All So code is syntactically valid Go. If a feature isn't listed here, it's not supported.

## The `so` command

The `so` CLI tool transpiles and compiles So programs. Make sure it is installed before use: `go install solod.dev/cmd/so@latest`.

Commands:

- `so run <package>` - transpile, compile, and run a package. Example: `so run .` or `so run ./myapp`.
- `so build <package>` - compile a package to an executable. Use `-o <file>` to set output name (default: package directory basename).
- `so translate <package>` - transpile a package to C (generates `.h` and `.c` files). Always use `-o generated` to set output directory to `generated`.
- `so version` - print compiler version.

A So project is a standard Go module. Create one with `go mod init <name>`, then add the So standard library dependency: `go get solod.dev@latest`. Third-party So packages (via `go install` or vendoring) and multi-module projects are supported.

Targets: native 64-bit and 32-bit, WebAssembly (WASI), and freestanding mode (no libc dependency).

## Key restrictions vs Go

Not supported: goroutines, channels, closures, anonymous functions, iterators, generic methods (non-inline), `recover`, `fallthrough`, type switches, interface-to-interface assertions, `delete` on maps, dynamic errors, named return values, returning interfaces or arrays from functions.

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

- Decay to pointers when passed to functions (no value semantics on calls).
- Cannot be returned from functions.
- `len()` and `cap()` are compile-time constants.
- Slicing an array produces a `so_Slice`.

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

- No closures or anonymous functions. Use named function types instead.
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

### Variadic functions

Standard `...T` syntax. Can spread slices with `s...`.

## Structs

Standard Go struct syntax. Supports anonymous structs and inner struct fields.

`new()` works with types and values:

```go
p := new(point)         // *point, zero-initialized
p2 := new(point{1, 2})  // *point with values
n := new(42)            // *int with value 42
```

## Methods

- Pointer receivers: `void* self` in C, cast internally.
- Value receivers: struct passed by value.
- Method expressions (`T.method`) supported; method values (`v.method`) not supported.

## Interfaces

Struct with `void* self` + function pointers. No runtime type info.

- Methods must use pointer receivers.
- Convert to interface via pointer: `var s Shape = &rect`.
- Type assertions: `_, ok := s.(*Rect)` or `r := s.(*Rect)` (not both).
- Empty interface (`any`/`interface{}`) = `void*`.
- No interface-to-interface assertions. No type switches.

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

## Packages

Each package -> one `.h` + `.c` pair. Multiple `.go` files merged.

- Exported symbols prefixed with package name; unexported are `static`.
- Import -> `#include`.
- **Declaration order matters** (C ordering): declare constants, types, and variables before functions that use them. If type B refers to type A, declare A first.
- One `init()` per package allowed (emitted as `__attribute__((constructor))`).

## Memory model

- **Stack by default:** `make()`, `append()`, `new()`, map literals are all stack-allocated.
- **Heap via stdlib:** `slices.Make`, `slices.Append`, `maps.New`, `mem.Alloc` - all take `mem.Allocator`.
- Pass `nil` for allocator to use the system allocator (calloc/realloc/free).
- Maximum stack allocation: 64KB (`so_MaxAllocaSize`).

## Testing

Since So code is valid Go, tests use the standard `go test` toolchain. Functions marked `//so:extern` have Go stub bodies that serve as test doubles - the Go test runner executes these stubs while the real C implementations are used in the compiled binary.

- Run all tests: `go test ./...`
- Run a specific test: `go test -run TestName ./path/to/package`
- `//so:extern` function bodies are only executed during `go test` - they are not emitted to C. Write them to return reasonable values for testing.

## Generics

Very limited. Two forms:

1. **Generic extern declarations** - type params become C macro args. Implemented as `#define` macros on C side.
2. **Generic inline macros** (`//so:inline` on generic functions) - auto-generates C macros. Restrictions: single return at end, no defer, args may evaluate multiple times. Prefix locals with `_`.

See [interop.md](interop.md) for details on both forms.

## Additional references

- **C interop and generics**: See [interop.md](interop.md) for `//so:extern`, `//so:include`, `//so:inline`, `//so:embed`, automatic decay, raw C intrinsics, and generic patterns.
- **Standard library**: See [stdlib.md](stdlib.md) for all `so/*` packages. Use `go doc -all solod.dev/so/<pkg>` for full API details.
