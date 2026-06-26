# C interop and generics

## Contents

- Includes
- Extern declarations and options
- Automatic decay
- Inlining
- Embeds
- Raw C intrinsics
- C helpers (so/c package)
- Generic extern declarations
- Generic inline macros

## Includes

Include a C header in the `.h` file:

```go
//so:include <stdint.h>
```

Include only in the `.c` file (implementation detail):

```go
//so:include.c "internal_helper.h"
```

## Extern declarations

Mark a type as external C (excluded from emission):

```go
//so:extern
type Account struct {
    name    string
    balance int64
}
```

Mark a function as external C:

```go
//so:extern
func dec_balance(acc *Account, amount int64) int64 {
    return 42 // Go body is a test stub only
}
```

Functions without bodies are also treated as extern C declarations (no `//so:extern` needed):

```go
func fopen(path string, mode string) *File
```

### Extern options

**Name override** - specify the C symbol name:

```go
//so:extern Account
type Account struct { ... }
// Uses "Account" in C instead of "main_Account"

//so:extern maps_rehash
func rehash(m *Map) { ... }
// Uses "maps_rehash" in C instead of "rehash"
```

**Nodecay** - skip automatic type decay for So-aware C functions:

```go
//so:extern nodecay
func set_name(acc *Account, name string)
// Passes so_String directly instead of char*
```

**Combined:**

```go
//so:extern MyFunc nodecay
func MyFunc(s string)
```

## Automatic decay

When calling extern functions, So types are automatically decayed to C equivalents:

- String literals -> raw C strings (`"hello"`)
- String values -> `char*` (`.ptr`)
- Slices -> raw pointers (`.ptr`)

Use `nodecay` to pass `so_String`/`so_Slice` directly.

## Inlining

Force emission as `static inline` in the header:

```go
//so:inline
func add(a, b int) int {
    return a + b
}
```

Works with both functions and methods.

## Embeds

Embed C files directly into generated output:

```go
//so:embed mypackage.h
var mypackage_h string

//so:embed mypackage.c
var mypackage_c string
```

`.h` files embed into the generated header, `.c` files into the implementation. The `var` declarations are markers only - not emitted as C variables.

## Qualifiers and attributes

**`//so:volatile`** - mark a package-level `var` as C `volatile`:

```go
//so:volatile
var counter int
```

**`//so:thread_local`** - mark a package-level `var` as thread-local (C11 `_Thread_local`):

```go
//so:thread_local
var perThread int
```

`//so:volatile` and `//so:thread_local` are allowed only on `var` declarations and can be combined.

**`//so:attr <value>`** - add a GCC/Clang `__attribute__`. Text after `so:attr` is the attribute value. Allowed on `var`, `const`, `type`, and `func` declarations. Multiple lines combine:

```go
//so:attr packed
//so:attr aligned(16)
type header struct {
    version byte
    length  int
}
```

## Raw C intrinsics

`c.Val[T](expr)` - typed C expression:

```go
nan := c.Val[float64]("NAN")
x := c.Val[float64]("sqrt(49)")
```

`c.Raw(code)` - raw C statement block:

```go
var b int
c.Raw(`
int a = 7;
b = a * a;
`)
```

Prefer `//so:extern` and `//so:embed` over raw C when possible.

## C helpers (so/c package)

Type information:

- `c.Sizeof[T]()` - size of type T
- `c.Alignof[T]()` - alignment of type T
- `c.Zero[T]()` - zero value of type T

Memory:

- `c.Alloca[T](n)` - stack-allocate array of n elements

String conversion:

- `c.String[T](ptr)` - C string (`char*` or similar) to So string
- `c.CString(s)` - So string to null-terminated C string (stack alloca)

Pointer/slice wrapping:

- `c.Bytes(ptr, n)` - raw pointer to byte slice
- `c.Slice[T](ptr, len, cap)` - raw pointer to typed slice

Pointer arithmetic:

- `c.PtrAdd(ptr, offset)` - add byte offset to pointer
- `c.PtrAs[T](ptr)` - cast pointer to `*T`
- `c.PtrAt[T](ptr, index)` - index into pointer as `*T`

Assert:

- `c.Assert(cond, msg)` - abort with message if condition is false

Types:

- `c.Char` - C `char`
- `c.ConstChar` - C `const char`
- Numeric C types for interop: `c.Int`, `c.UInt`, `c.Long`, `c.ULong`, etc.

## Generic extern declarations

Type parameters become C macro arguments, prepended before regular arguments:

```go
//so:extern
func newObj[T any]() *T { return nil }

//so:extern
func freeObj[T any](ptr *T) {}
```

Calling:

```go
v := newObj[int]()  // C: newObj(so_int)
freeObj(v)          // C: freeObj(so_int, v) - type inferred
```

Generic extern types and methods:

```go
//so:extern
type Map[K comparable, V any] struct { ... }

//so:extern
func (m *Map[K, V]) Len() int { return m.len }
```

On C side, implement as `#define` macros receiving type arguments:

```c
#define newObj(T) (alloca(sizeof(T)))
#define main_Map_Len(K, V, m) ((m)->len)
```

Constraints (`any`, `comparable`) are for Go type-checking only.

## Generic inline macros

`//so:inline` on a generic function auto-generates a C `#define` macro:

```go
//so:inline
func identity[T any](val T) T {
    return val
}
```

Produces:

```c
#define identity(T, val_) ({ \
    (val_); \
})
```

Rules:

- Type params prepended as macro params; non-type params get `_` suffix.
- Returns use statement expression `({ ... })`; void uses `do { ... } while (0)`.
- `return expr` becomes bare `expr;` (last expression is the value).

Restrictions:

- **Single return at end** - no early returns (multiple returns won't work).
- **No defer** inside macro bodies.
- **Args may evaluate multiple times** - copy to locals to prevent.
- **No break/continue/goto** escaping the macro.

Best practices:

- Prefix local variables with `_` to avoid shadowing.
- Copy arguments to `_`-prefixed locals at top of body.
- Keep macros short (~15 lines max). Move heavy logic to non-generic helpers.
- Avoid chaining inline macro calls.

Example with best practices:

```go
//so:inline
func (m *Map[K, V]) Has(key K) bool {
    _key := key    // copy to prevent double evaluation
    _m := m.bm
    // ... use _key and _m ...
    return _found
}
```
