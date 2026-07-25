# So standard library

The standard library lives in the Go module `solod.dev`. Install it with `go get solod.dev@latest`. Each package is imported as `solod.dev/so/<pkg>`.

## Learning a package API

This file lists what each package is for, not what's in it. Most So packages mirror their Go counterparts with fewer features, so start from your Go knowledge, then confirm the actual signatures with `go doc` before writing code:

- `go doc solod.dev/so/strings` - list the package API.
- `go doc solod.dev/so/strings Split` - a single symbol.
- `go doc -all solod.dev/so/os` - everything, including doc comments.

Check `go doc` whenever a signature isn't obvious. So APIs differ from Go's in ways that don't show up until compile time (extra allocator arguments, buffer arguments, missing functions).

## Conventions

These apply across the library:

- **Allocators**. A function that allocates takes a `mem.Allocator` as its first argument. Pass `mem.System` for the C heap, or an arena/tracker for a custom lifetime. The caller owns the result and frees it (`mem.Free`, `mem.FreeSlice`, or the type's own `Free` method).
- **Caller buffers**. Where Go would allocate, So often writes into a caller-provided `buf []byte` and returns a string or slice pointing into it. The buffer must outlive the result.
- **Free methods**. A type that owns memory (buffers, builders, readers, channels, maps) has a `Free` method. Pair every constructor with it, usually via `defer`.
- **No reflection**. Nothing in the library formats or parses arbitrary values by type. Encoding APIs are token-level or explicit.
- **Errors**. Package-level sentinel errors only, compared with `==`.

## Packages

### Core

- **c** - low-level C interop: pointer arithmetic, C string conversion, `sizeof`/`alignof`, stack allocation, assertions, and numeric C types. See [interop.md](interop.md).
- **errors** - creates sentinel errors from text. Only usable at package level.
- **mem** - memory allocation through a pluggable `Allocator` interface. Provides the system allocator (C `calloc`/`realloc`/`free`), a bump-allocating arena over a fixed buffer, a no-op allocator that always fails, and a tracker that records allocation stats and leaks. Also the typed alloc/free helpers and raw memory operations (copy, move, compare, clear).
- **runtime** - target OS and architecture, compiler version, CPU count, a cryptographically secure seed, and the current source location (file, line, function).

### Data structures

- **maps** - heap-allocated generic hash map with automatic growth (Robin Hood hashing). Use it instead of the built-in `map[K]V`, which is stack-allocated, fixed-capacity, and can't be returned from a function.
- **slices** - heap-allocated dynamic slices plus sorting and search helpers. Use it instead of built-in `make`/`append` whenever a slice must grow or outlive the current function.
- **cmp** - comparison of ordered values, and the comparison function type used by the sorting APIs in `slices`.

### Strings and text

- **strings** - operations on UTF-8 strings: search, split, join, trim, case conversion, replacement, plus an efficient string builder and a string reader. Allocating functions take an allocator; non-allocating ones (slicing, search, trim) don't.
- **bytes** - the same operations for `[]byte`, plus a growable byte buffer and a byte-slice reader.
- **strconv** - conversions between strings and numbers or booleans.
- **unicode** - properties of Unicode code points: character classes and case. Fewer categories than Go's (no graphic, mark, punctuation, or symbol tables).
- **unicode/utf8** - conversion between runes and UTF-8 byte sequences.

### I/O

- **io** - the basic `Reader`, `Writer`, `Seeker`, and `Closer` interfaces every other I/O package implements, plus copy helpers, limited and section readers, multi readers/writers, and a discard writer.
- **bufio** - buffering on top of an `io.Reader` or `io.Writer`, and a scanner for token-based reading (lines, words, runes, or a custom split function).
- **os** - files, directories, file metadata, temporary files, working directory, environment variables, process and user IDs, and program exit. Built on POSIX APIs.
- **fmt** - formatted I/O modeled on C's `printf`/`scanf`. **Format verbs are C verbs, not Go verbs**: `%d`, `%s`, `%f`, `%p`. `Print`/`Println` take only strings, so the built-in `print`/`println` are usually more convenient.
- **flag** - command-line flag parsing, close to Go's `flag`.
- **log/slog** - leveled key-value logging with a text handler. Formats records without allocating.

### Encoding

- **encoding/binary** - fixed-size numbers to and from byte sequences in little- or big-endian order.
- **encoding/hex** - hexadecimal encoding and decoding, including streaming encoders/decoders and `hexdump -C` style dumps.
- **encoding/json** - streaming, token-level JSON. With no reflection there is no `Marshal`/`Unmarshal`: the decoder pulls one validated token at a time, and the encoder writes tokens while inserting commas and colons. One root value per document; multi-document streams (NDJSON) aren't supported.

### Math

- **math** - constants and floating-point functions, same API as Go's `math`. Results aren't guaranteed bit-identical across architectures.
- **math/bits** - bit counting and manipulation, same API as Go's `math/bits`.
- **math/rand** - pseudo-random numbers from a PCG source, modeled on Go's `math/rand/v2`. Not for security-sensitive work. Top-level functions use a global generator seeded by `runtime.Seed`.
- **crypto/crand** - cryptographically secure random bytes and random text.

### System and network

- **net** - TCP, UDP, and Unix domain socket networking: dialing, listening, accepting, and datagram send/receive. Connections implement `io.Reader`/`io.Writer`. Calls block by default; bound them with deadlines. No concurrent server support.
- **net/netip** - small value types for IP addresses, address-port pairs, and CIDR prefixes. IPv6 zones are numeric scope IDs, not strings.
- **path** - lexical manipulation of slash-separated paths, and shell-pattern matching.
- **time** - measuring and formatting time. Always UTC internally, with a fixed offset applied at formatting time; **formatting and parsing use C strftime/strptime verbs** (`%Y-%m-%d %H:%M:%S`), not Go layouts. Also durations and sleeping.
- **uuid** - UUID generation and parsing per RFC 9562 (v4 and v7). Random components come from a cryptographically secure RNG.

### Concurrency

So has no goroutines or channels at the language level. These packages provide the equivalents, backed by pthreads. Importing any of them links `-lpthread` automatically.

- **conc** - concurrency primitives: OS threads running a `func(any) any`, a generic thread-safe FIFO channel (buffered or rendezvous, with timeout variants), and a bounded worker pool for fork-join parallelism. Values are carried by copy.
- **sync** - mutual exclusion locks, condition variables, and once-only execution. These types must be initialized, must not be copied, and must be freed.
- **sync/atomic** - lock-free atomic integers, booleans, and pointers. The zero value is ready to use; must not be copied after first use.

### Testing

- **testing** - the `T` and `B` types used by tests and benchmarks, along with the runners the generated `main.go` calls. Tests live in a package's `test` subdirectory and run with `so test`; benchmarks live in `bench` and run with `so bench`. See the testing section of [SKILL.md](SKILL.md).
