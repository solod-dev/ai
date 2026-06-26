# So standard Library

The standard library itself is in the Go module named `solod.dev`. Make sure it is installed before use: `go get solod.dev@latest`

Each stdlib package is located at `solod.dev/so/<pkg>`. Use `go doc -all solod.dev/so/<pkg>` for full API details, but ONLY when you REALLY need them. Most So stdlib package APIs are VERY similar to Go's stdlib packages. Use information from this file and your existing Go knowledge as much as possible to write code and filter the `go doc` commands.

## Package list

- bufio - Buffered I/O
- bytes - Byte slice operations
- c - C interop helpers (see [interop.md])
- cmp - Comparison functions
- crypto/crand - Secure random
- encoding/binary - Byte order
- encoding/hex - Hex encoding/decoding
- errors - Error creation
- flag - Command-line flags
- fmt - Formatted I/O
- io - I/O interfaces
- log/slog - Structured logging
- maps - Dynamic hash maps
- math, math/bits, math/rand - Math operations
- mem - Memory allocation
- net - TCP/UDP/Unix socket networking
- net/netip - IP address value types
- os - File I/O and filesystem
- path - Path manipulation
- runtime - Runtime info
- slices - Dynamic slices
- strconv - Number/string conversion
- strings - String operations
- time - Time measurement
- unicode, unicode/utf8 - Unicode
- uuid - UUID generation (RFC 9562)

## so/bufio

Buffered I/O. Wraps an `io.Reader` or `io.Writer` with buffering and helpers for textual I/O.

```go
type ReadWriter struct
func NewReadWriter(r *Reader, w *Writer) ReadWriter
func (rw *ReadWriter) Read(p []byte) (int, error)
func (rw *ReadWriter) Write(p []byte) (int, error)

type Reader struct
func NewReader(a mem.Allocator, rd io.Reader) Reader
func NewReaderSize(a mem.Allocator, rd io.Reader, size int) Reader
func (b *Reader) Buffered() int
func (b *Reader) Discard(n int) (int, error)
func (b *Reader) Free()
func (b *Reader) Peek(n int) ([]byte, error)
func (b *Reader) Read(p []byte) (int, error)
func (b *Reader) ReadByte() (byte, error)
func (b *Reader) ReadBytes(delim byte) ([]byte, error)
func (b *Reader) ReadLine() ReadLineResult
func (b *Reader) ReadRune() io.RuneSizeResult
func (b *Reader) ReadSlice(delim byte) ([]byte, error)
func (b *Reader) ReadString(delim byte) (string, error)
func (b *Reader) Reset(r io.Reader)
func (b *Reader) Size() int
func (b *Reader) UnreadByte() error
func (b *Reader) UnreadRune() error
func (b *Reader) WriteTo(w io.Writer) (int64, error)

type Scanner struct
func NewScanner(a mem.Allocator, r io.Reader) Scanner
func (s *Scanner) Buffer(buf []byte, max int)
func (s *Scanner) Bytes() []byte
func (s *Scanner) Err() error
func (s *Scanner) Free()
func (s *Scanner) Scan() bool
func (s *Scanner) Split(split SplitFunc)
func (s *Scanner) Text() string

type SplitFunc func(data []byte, atEOF bool) SplitResult
type SplitResult struct
func ScanBytes(data []byte, atEOF bool) SplitResult
func ScanLines(data []byte, atEOF bool) SplitResult
func ScanRunes(data []byte, atEOF bool) SplitResult
func ScanWords(data []byte, atEOF bool) SplitResult

type Writer struct
func NewWriter(a mem.Allocator, w io.Writer) Writer
func NewWriterSize(a mem.Allocator, w io.Writer, size int) Writer
func (b *Writer) Available() int
func (b *Writer) AvailableBuffer() []byte
func (b *Writer) Buffered() int
func (b *Writer) Flush() error
func (b *Writer) Free()
func (b *Writer) ReadFrom(r io.Reader) (int64, error)
func (b *Writer) Reset(w io.Writer)
func (b *Writer) Size() int
func (b *Writer) Write(p []byte) (int, error)
func (b *Writer) WriteByte(c byte) error
func (b *Writer) WriteRune(r rune) (int, error)
func (b *Writer) WriteString(s string) (int, error)
```

## so/bytes

Byte slice operations. Similar API to `so/strings` but for `[]byte`.

Functions: `Clone`, `Compare`, `Contains`, `Count`, `Cut`, `Equal`, `HasPrefix`, `HasSuffix`, `Index`, `IndexByte`, `Join`, `Replace`, `Runes`, `Split`, `SplitN`, `String`, `ToLower`, `ToUpper`, `TrimLeft`, `TrimRight`, `TrimSpace`, `TrimPrefix`, `TrimSuffix`.

Types:

```go
type Buffer struct
func NewBuffer(a mem.Allocator, buf []byte) Buffer
func NewBufferString(a mem.Allocator, s string) Buffer
func (b *Buffer) Available() int
func (b *Buffer) Bytes() []byte
func (b *Buffer) Cap() int
func (b *Buffer) Free()
func (b *Buffer) Grow(n int)
func (b *Buffer) Len() int
func (b *Buffer) Next(n int) []byte
func (b *Buffer) Peek(n int) ([]byte, error)
func (b *Buffer) Read(p []byte) (int, error)
func (b *Buffer) ReadByte() (byte, error)
func (b *Buffer) ReadBytes(delim byte) ([]byte, error)
func (b *Buffer) ReadFrom(r io.Reader) (int64, error)
func (b *Buffer) ReadRune() io.RuneSizeResult
func (b *Buffer) ReadString(delim byte) (string, error)
func (b *Buffer) Reset()
func (b *Buffer) String() string
func (b *Buffer) Write(p []byte) (int, error)
func (b *Buffer) WriteByte(c byte) error
func (b *Buffer) WriteRune(r rune) (int, error)
func (b *Buffer) WriteString(s string) (int, error)
func (b *Buffer) WriteTo(w io.Writer) (int64, error)

type Reader struct
func NewReader(b []byte) Reader
func (r *Reader) Len() int
func (r *Reader) Read(b []byte) (int, error)
func (r *Reader) ReadAt(b []byte, off int64) (int, error)
func (r *Reader) ReadByte() (byte, error)
func (r *Reader) ReadRune() io.RuneSizeResult
func (r *Reader) Reset(b []byte)
func (r *Reader) Seek(offset int64, whence int) (int64, error)
func (r *Reader) Size() int64
func (r *Reader) UnreadByte() error
func (r *Reader) UnreadRune() error
func (r *Reader) WriteTo(w io.Writer) (int64, error)

type RuneFunc func(rune) rune
type RunePredicate func(rune) bool
```

## so/errors

- `New(text string) error` - create a sentinel error at package level only. Not for use inside functions.

`error.Error` method is not supported.

## so/fmt

Formatted I/O. **Uses C format verbs** (not Go verbs): `%d`, `%s`, `%f`, etc.

```go
func Fprintf(w io.Writer, format string, a ...any) (int, error)
func Fscanf(r io.Reader, format string, a ...any) (int, error)
func Print(a ...string) (int, error)
func Printf(format string, a ...any) (int, error)
func Println(a ...string) (int, error)
func Scanf(format string, a ...any) (int, error)
func Sprintf(buf Buffer, format string, a ...any) string
func Sscanf(str string, format string, a ...any) (int, error)

type Buffer struct
func BufferFrom(buf []byte) Buffer
func NewBuffer(size int) Buffer
func (b Buffer) String() string
```

## so/io

Basic I/O interfaces.

```go
var EOF = errors.New("EOF")
var Discard Writer = &DiscardWriter{}

func Copy(dst Writer, src Reader) (int64, error)
func CopyN(dst Writer, src Reader, n int64) (int64, error)
func ReadAll(a mem.Allocator, r Reader) ([]byte, error)
func ReadFull(r Reader, buf []byte) (int, error)
func WriteString(w Writer, s string) (int, error)

type ByteReader interface
type ByteScanner interface
type ByteWriter interface
type Closer interface
type DiscardWriter struct
type LimitedReader struct
func LimitReader(r Reader, n int64) LimitedReader
type MultiReader struct
func NewMultiReader(readers ...Reader) MultiReader
type MultiWriter struct
func NewMultiWriter(writers ...Writer) MultiWriter
type NopCloser struct
func NewNopCloser(r Reader) NopCloser
type ReadCloser interface
type ReadSeekCloser interface
type ReadSeeker interface
type ReadWriteCloser interface
type ReadWriteSeeker interface
type ReadWriter interface
type Reader interface
type ReaderAt interface
type ReaderAtOffset struct
type ReaderFrom interface
type RuneReader interface
type RuneScanner interface
type RuneSizeResult struct
type SectionReader struct
func NewSectionReader(r ReaderAt, off int64, n int64) SectionReader
type Seeker interface
type StringWriter interface
type WriteCloser interface
type WriteSeeker interface
type Writer interface
type WriterAt interface
type WriterTo interface
```

## so/maps

Generic Robin Hood hash map with automatic grow. Use instead of `make(map[K]V)` when you need dynamic maps.

```go
type Iter[K comparable, V any] struct
func (it *Iter[K, V]) Key() K
func (it *Iter[K, V]) Next() bool
func (it *Iter[K, V]) Value() V

type Map[K comparable, V any] struct {
func New[K comparable, V any](a mem.Allocator, size int) Map[K, V]
func (m *Map[K, V]) Clear()
func (m *Map[K, V]) Delete(key K)
func (m *Map[K, V]) Free()
func (m *Map[K, V]) Get(key K) V
func (m *Map[K, V]) Has(key K) bool
func (m *Map[K, V]) Iter() Iter[K, V]
func (m *Map[K, V]) Len() int
func (m *Map[K, V]) Set(key K, value V)
```

## so/mem

Pluggable allocator interface. Most allocating stdlib functions take `mem.Allocator` as first arg. Passing `nil` means using the `System` allocator, but prefer always passing `mem.System` explicitly.

```go
var ErrNoAlloc error
var ErrOutOfMemory error

func Alloc[T any](a Allocator) *T
func AllocSlice[T any](a Allocator, len int, cap int) []T
func Clear(ptr any, size int)
func Compare(a any, b any, size int) int
func Copy(dst any, src any, n int) any
func Free[T any](a Allocator, ptr *T)
func FreeSlice[T any](a Allocator, slice []T)
func FreeString(a Allocator, s string)
func Move(dst any, src any, n int) any
func ReallocSlice[T any](a Allocator, slice []T, newLen int, newCap int) []T
func Swap[T any](a *T, b *T)
func SwapByte(a any, b any, n int)
func TryAlloc[T any](a Allocator) (*T, error)
func TryAllocSlice[T any](a Allocator, len int, cap int) ([]T, error)
func TryReallocSlice[T any](a Allocator, slice []T, newLen int, newCap int) ([]T, error)

type Allocator interface {
	Alloc(size int, align int) (any, error)
	Realloc(ptr any, oldSize int, newSize int, align int) (any, error)
	Free(ptr any, size int, align int)
}

var NoAlloc Allocator = &NoAllocator{}
var System Allocator = &SystemAllocator{}

type Arena struct
func NewArena(buf []byte) Arena
func (a *Arena) Alloc(size int, align int) (any, error)
func (a *Arena) Free(ptr any, size int, align int)
func (a *Arena) Realloc(ptr any, oldSize int, newSize int, align int) (any, error)
func (a *Arena) Reset()

type NoAllocator struct
func (*NoAllocator) Alloc(size int, align int) (any, error)
func (*NoAllocator) Free(ptr any, size int, align int)
func (*NoAllocator) Realloc(ptr any, oldSize int, newSize int, align int) (any, error)

type SystemAllocator struct
func (*SystemAllocator) Alloc(size int, align int) (any, error)
func (*SystemAllocator) Free(ptr any, size int, align int)
func (*SystemAllocator) Realloc(ptr any, oldSize int, newSize int, align int) (any, error)
```

## so/net

Basic TCP, UDP, and Unix domain socket networking. Networks: `tcp`/`tcp4`/`tcp6`, `udp`/`udp4`/`udp6`, `unix` (stream) and `unixgram` (datagram). No concurrent server support.

Conns implement `io.Reader`/`io.Writer`. Unconnected UDP/Unix datagram sockets use `ReadFrom`/`WriteTo` instead. `Accept`/`Read`/`Write` block by default; bound them with `SetDeadline` (fails with `ErrTimeout` past the deadline).

```go
func SplitHostPort(hostport string) (HostPort, error)
func JoinHostPort(buf []byte, host, port string) string

// TCP
func ResolveTCPAddr(network, address string) (TCPAddr, error)
func DialTCP(network string, laddr, raddr *TCPAddr) (TCPConn, error)
func ListenTCP(network string, laddr *TCPAddr) (TCPListener, error)
func (l *TCPListener) Accept() (TCPConn, error)
func (conn *TCPConn) Read(b []byte) (int, error)
func (conn *TCPConn) Write(b []byte) (int, error)
func (conn *TCPConn) Close() error

// UDP: DialUDP = connected (Read/Write), ListenUDP = unconnected (ReadFrom/WriteTo)
func ResolveUDPAddr(network, address string) (UDPAddr, error)
func DialUDP(network string, laddr, raddr *UDPAddr) (UDPConn, error)
func ListenUDP(network string, laddr *UDPAddr) (UDPConn, error)
func (conn *UDPConn) ReadFrom(b []byte) (UDPRead, error)
func (conn *UDPConn) WriteTo(b []byte, addr *UDPAddr) (int, error)

// Unix: DialUnix/ListenUnix = stream, ListenUnixgram = datagram. Socket file removed on Close.
func ResolveUnixAddr(network, address string) (UnixAddr, error)
func DialUnix(network string, laddr, raddr *UnixAddr) (UnixConn, error)
func ListenUnix(network string, laddr *UnixAddr) (UnixListener, error)
func ListenUnixgram(network string, laddr *UnixAddr) (UnixConn, error)
```

Types: `HostPort`, `TCPAddr`/`TCPConn`/`TCPListener`, `UDPAddr`/`UDPConn`, `UnixAddr`/`UnixConn`/`UnixListener`. Conns also have `LocalAddr`/`RemoteAddr` and `SetReadDeadline`/`SetWriteDeadline`. Conns must not be copied after use.

## so/os

File I/O and filesystem (POSIX-based).

```go
func Chdir(dir string) error
func Chmod(name string, mode FileMode) error
func Chown(name string, uid, gid int) error
func Chtimes(name string, atime time.Time, mtime time.Time) error
func Exit(code int)
func FreeDirEntry(a mem.Allocator, entries []DirEntry)
func Getegid() int
func Getenv(key string) string
func Geteuid() int
func Getgid() int
func Getpid() int
func Getppid() int
func Getuid() int
func Getwd(buf []byte) (string, error)
func Hostname(buf []byte) (string, error)
func Lchown(name string, uid, gid int) error
func Link(oldname, newname string) error
func LookupEnv(key string) (string, bool)
func Mkdir(name string, perm FileMode) error
func MkdirTemp(buf []byte, dir, pattern string) (string, error)
func ReadFile(a mem.Allocator, name string) ([]byte, error)
func Readlink(buf []byte, name string) (string, error)
func Remove(name string) error
func Rename(oldpath, newpath string) error
func SameFile(fi1, fi2 FileInfo) bool
func Setenv(key, value string) error
func Symlink(oldname, newname string) error
func TempDir() string
func Truncate(name string, size int64) error
func Unsetenv(key string) error
func WriteFile(name string, data []byte, perm FileMode) error

type DirEntry struct
func ReadDir(a mem.Allocator, name string) ([]DirEntry, error)

type File struct
var Stderr *File
var Stdin *File
var Stdout *File
func Create(name string) (File, error)
func CreateTemp(buf []byte, dir, pattern string) (File, error)
func Open(name string) (File, error)
func OpenFile(name string, flag int, perm FileMode) (File, error)

func (f *File) Close() error
func (f *File) Name() string
func (f *File) Read(b []byte) (int, error)
func (f *File) ReadAt(b []byte, off int64) (int, error)
func (f *File) Seek(offset int64, whence int) (int64, error)
func (f *File) Write(b []byte) (int, error)
func (f *File) WriteAt(b []byte, off int64) (int, error)
func (f *File) WriteString(s string) (int, error)

type FileInfo struct
func Lstat(name string) (FileInfo, error)
func Stat(name string) (FileInfo, error)
func (fi *FileInfo) IsDir() bool
func (fi *FileInfo) ModTime() time.Time
func (fi *FileInfo) Mode() FileMode
func (fi *FileInfo) Name() string
func (fi *FileInfo) Size() int64

type FileMode uint32
```

## so/slices

Heap-allocated dynamic slices. Use instead of `make`/`append` when you need dynamic growth.

```go
func Append[T any](a mem.Allocator, s []T, elems ...T) []T
func Clone[T any](a mem.Allocator, s []T) []T
func Contains[T comparable](s []T, v T) bool
func Equal[T comparable](s1, s2 []T) bool
func Extend[T any](a mem.Allocator, s []T, other []T) []T
func Free[T any](a mem.Allocator, s []T)
func Index[T comparable](s []T, v T) int
func IsSorted[T gocmp.Ordered](x []T) bool
func IsSortedFunc[T any](x []T, compare cmp.Func) bool
func IsSortedWith(s Sorter) bool
func Make[T any](a mem.Allocator, len int) []T
func MakeCap[T any](a mem.Allocator, len int, cap int) []T
func Max[T gocmp.Ordered](x []T) T
func MaxFunc[T any](x []T, compare cmp.Func) T
func Min[T gocmp.Ordered](x []T) T
func MinFunc[T any](x []T, compare cmp.Func) T
func Sort[T gocmp.Ordered](x []T)
func SortFunc[T any](x []T, compare cmp.Func)
func SortStableFunc[T any](x []T, compare cmp.Func)
func SortStableWith(s Sorter)
func SortWith(s Sorter)

type Slice struct
func Header[T any](s []T) Slice

type Sorter struct {
func NewSorter[T any](s []T, compare cmp.Func) Sorter
func (s Sorter) Compare(i, j int) int
func (s Sorter) Less(i, j int) bool
func (s Sorter) Swap(i, j int)
```

## so/strings

String operations. Allocating functions take `mem.Allocator`.

```go
func Clone(a mem.Allocator, s string) string
func Compare(a, b string) int
func Contains(s, substr string) bool
func ContainsAny(s, chars string) bool
func ContainsFunc(s string, f RunePredicate) bool
func ContainsRune(s string, r rune) bool
func Count(s, substr string) int
func Cut(s, sep string) (string, string)
func CutPrefix(s, prefix string) (string, bool)
func CutSuffix(s, suffix string) (string, bool)
func Fields(a mem.Allocator, s string) []string
func FieldsFunc(a mem.Allocator, s string, f RunePredicate) []string
func HasPrefix(s, prefix string) bool
func HasSuffix(s, suffix string) bool
func Index(s, substr string) int
func IndexAny(s, chars string) int
func IndexByte(s string, c byte) int
func IndexFunc(s string, f RunePredicate) int
func IndexRune(s string, r rune) int
func Join(a mem.Allocator, elems []string, sep string) string
func LastIndex(s, substr string) int
func LastIndexByte(s string, c byte) int
func Map(a mem.Allocator, mapping RuneFunc, s string) string
func Repeat(a mem.Allocator, s string, count int) string
func Replace(a mem.Allocator, s, old, new string, n int) string
func ReplaceAll(a mem.Allocator, s, old, new string) string
func Split(a mem.Allocator, s, sep string) []string
func SplitAfter(a mem.Allocator, s, sep string) []string
func SplitN(a mem.Allocator, s, sep string, n int) []string
func ToLower(a mem.Allocator, s string) string
func ToUpper(a mem.Allocator, s string) string
func Trim(s, cutset string) string
func TrimFunc(s string, f RunePredicate) string
func TrimLeft(s, cutset string) string
func TrimPrefix(s, prefix string) string
func TrimRight(s, cutset string) string
func TrimSpace(s string) string
func TrimSuffix(s, suffix string) string

type Builder struct
func FixedBuilder(buf []byte) Builder
func NewBuilder(a mem.Allocator) Builder
func (b *Builder) Cap() int
func (b *Builder) Free()
func (b *Builder) Grow(n int)
func (b *Builder) Len() int
func (b *Builder) Reset()
func (b *Builder) String() string
func (b *Builder) Write(p []byte) (int, error)
func (b *Builder) WriteByte(c byte) error
func (b *Builder) WriteRune(r rune) (int, error)
func (b *Builder) WriteString(s string) (int, error)

type Reader struct
func NewReader(s string) Reader
func (r *Reader) Len() int
func (r *Reader) Read(b []byte) (int, error)
func (r *Reader) ReadAt(b []byte, off int64) (int, error)
func (r *Reader) ReadByte() (byte, error)
func (r *Reader) ReadRune() io.RuneSizeResult
func (r *Reader) Reset(s string)
func (r *Reader) Seek(offset int64, whence int) (int64, error)
func (r *Reader) Size() int64
func (r *Reader) UnreadByte() error
func (r *Reader) UnreadRune() error
func (r *Reader) WriteTo(w io.Writer) (int64, error)

type RuneFunc func(rune) rune
type RunePredicate func(rune) bool
```

Types: `Builder` (efficient string building), `Reader` (reads from string).

## so/time

Time measurement. Always UTC internally. Formatting uses C strftime/strptime verbs (`%Y-%m-%d %H:%M:%S`).

```go
type Duration int64
func Since(t Time) Duration
func Until(t Time) Duration
func (d Duration) Abs() Duration
func (d Duration) Hours() float64
func (d Duration) Microseconds() int64
func (d Duration) Milliseconds() int64
func (d Duration) Minutes() float64
func (d Duration) Nanoseconds() int64
func (d Duration) Round(m Duration) Duration
func (d Duration) Seconds() float64
func (d Duration) String(buf []byte) string
func (d Duration) Truncate(m Duration) Duration

type Month int
const January Month = 1 + iota ...

type Offset int
const UTC Offset = 0

type Time struct
func Date(year int, month Month, day, hour, min, sec, nsec int, offset Offset) Time
func Now() Time
func Parse(layout string, value string, offset Offset) (Time, error)
func Unix(sec int64, nsec int64) Time
func UnixMicro(usec int64) Time
func UnixMilli(msec int64) Time

func (t Time) Add(d Duration) Time
func (t Time) AddDate(years int, months int, days int) Time
func (t Time) After(u Time) bool
func (t Time) Before(u Time) bool
func (t Time) Clock(offset Offset) CalClock
func (t Time) Compare(u Time) int
func (t Time) Date(offset Offset) CalDate
func (t Time) Day() int
func (t Time) Equal(u Time) bool
func (t Time) Format(buf []byte, layout string, offset Offset) string
func (t Time) Hour() int
func (t Time) ISOWeek() (int, int)
func (t Time) IsZero() bool
func (t Time) Minute() int
func (t Time) Month() Month
func (t Time) Nanosecond() int
func (t Time) Round(d Duration) Time
func (t Time) Second() int
func (t Time) String(buf []byte) string
func (t Time) Sub(u Time) Duration
func (t Time) Truncate(d Duration) Time
func (t Time) Unix() int64
func (t Time) UnixMicro() int64
func (t Time) UnixMilli() int64
func (t Time) UnixNano() int64
func (t Time) Weekday() Weekday
func (t Time) Year() int
func (t Time) YearDay() int

type Weekday int
const Sunday Weekday = iota ...
```

## Other packages

- **so/cmp** - comparison. `Compare`, `Equal`, `Less`, `Func`, `FuncFor[T]`.
- **so/crypto/crand** - `Read`, `Text` for cryptographically secure random data.
- **so/encoding/binary** - `LittleEndian`, `BigEndian` byte order.
- **so/encoding/hex** - hex encoding/decoding. `Encode`, `Decode`, `Dump`, `Encoder`, `Decoder`, `Dumper`.
- **so/flag** - command-line flag parsing. `BoolVar`, `IntVar`, `StringVar`, `Parse`, `Args`.
- **so/log/slog** - structured logging. `Debug`, `Info`, `Warn`, `Error` + key-value attrs.
- **so/math** - same API as Go's `math`.
- **so/math/bits** - same API as Go's `math/bits`.
- **so/math/rand** - PCG-based PRNG. `Int`, `IntN`, `Float64`, etc. Global seeded by `runtime.Seed`.
- **so/net/netip** - same API as Go's `net/netip`.
- **so/path** - path manipulation. `Base`, `Dir`, `Ext`, `Join`, `Split`, `Clean`, `IsAbs`, `Match`.
- **so/runtime** - `GOOS`, `GOARCH`, `Version()`, `Seed()`.
- **so/strconv** - conversions between numbers and strings.
- **so/unicode** - `IsDigit`, `IsLetter`, `IsSpace`, `IsLower`, `IsUpper`, `ToLower`, `ToUpper`.
- **so/unicode/utf8** - `DecodeRune`, `EncodeRune`, `RuneCount`, `ValidString`.
- **so/uuid** - same as Go's `uuid` (Go 1.27+).
