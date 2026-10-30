# Native applications and standard libraries

The compiler supports hosted x86-64 Windows/Linux and freestanding x86-64 ELF.
Windows execution and the QEMU target are verified locally. Native Linux still
needs execution in a Linux environment; it must not be inferred from Windows
test results. Hosted executables use the platform C/C++ runtime; freestanding
output has no hosted runtime dependency.

Windows graphical projects can use the typed [Windows libraries](windows-libraries.md)
and `--subsystem windows`. Freestanding projects can use the
[arena and framebuffer console](freestanding-libraries.md) alongside the shared
buffer/vector interfaces.

See [additional application libraries](expanded-libraries.md) for math, HTTP,
encoding, byte and string utilities, random numbers, and simpler Windows apps.

## Create, build and run

```sh
luasm new MyApp
luasm check MyApp
luasm run MyApp
luasm build MyApp --release
```

`new` creates `main.lasm`, a README and `.gitignore` in an absent/empty directory.
It refuses to overwrite a nonempty directory. Directory commands use `main.lasm`;
build products go to `.luasm-build/`. Single-file commands retain their previous
output behavior. `--` forwards remaining arguments only with `run`:

```sh
luasm run main.lasm -- input.txt "argument with spaces"
```

`argumentCount(): usize` includes the executable name at index zero.
`argument(index: usize): string` returns an immutable process-lifetime argument;
out-of-range access reports a runtime error. Windows arguments and file paths
are decoded to/from UTF-8 through Windows Unicode APIs. Stdout and line input
remain byte streams; terminal encoding is controlled by the terminal.

Imports try paths relative to their source first, then each `--module-path DIR`,
then the installed and configured build library directories. Explicit `./` and
`../` paths remain relative and do not use fallback search paths. The installed
standard library is available as `std/...` without copying it into every project.

## Function values

```lua
function double(value: i32): i32 return value * 2 end
function apply(operation: fn<i32, i32>, value: i32): i32
    return operation(value)
end
local operation: fn<i32, i32> = double
print(apply(operation, 21))
```

`fn<Result, ParameterTypes...>` is an eight-byte callable value. Named ordinary
and extern functions become values by referring to their name. Calls work on
locals, globals, struct fields, array elements and function-returned values.
Signatures are checked exactly (with the existing `number`/`f64` equivalence).
Scalar numeric/boolean/pointer/function types and void returns are supported;
aggregate/string/nil callback ABIs and captured closures are not. Interrupt entry
points cannot become ordinary callbacks. Functions retain static lifetimes, so
there is no captured environment to allocate or free.

Null `zero<fn<...>>()` values compare normally, but calling one reports a hosted
runtime error (exit 70), or traps freestanding. Callees are evaluated and checked
before arguments. Normal function references and equality require no low-mode
directive; raw addresses and pointer casts remain low-level operations.
All scalar definitions and calls use consistent C ABI extension attributes, so
C can call back into LuASM (the tests exercise the real C `qsort` function).
`LSA2115` reports invalid callable types/targets; argument errors retain the
existing type/count diagnostic families.

## Strings and text

`stringLength(text): usize`, `stringAt(text, index): u8` and
`stringSlice(text, offset, count): string` work in ordinary mode. Indices are
zero-based byte offsets, not Unicode codepoint indices. Slicing checks offset
and length without overflowing. Empty slices at the end are valid. Embedded NUL
bytes are preserved. Invalid ranges terminate with a runtime error or a
freestanding trap. `assert(condition: boolean)` provides the same fail-fast behavior.

`std/text` exports `startsWith`, `endsWith`, `find` and `trim`. `find` returns
`isize`, with -1 for absence; the empty needle matches at zero. `trim` removes
ASCII whitespace. Slices and text helpers produce borrowed views, not copies.
Literal and argument backing storage lives for the program's lifetime.

In low mode, `stringView(ptr<u8>, length)` explicitly borrows caller-owned bytes.
The caller must keep them readable and alive. The compiler validates null versus
nonzero length, not arbitrary mappings or ownership. Do not mutate or free that
storage while treating a view as immutable. In particular a buffer view becomes
invalid on buffer reallocation/destruction and changes when its bytes change.
There is no hidden garbage collector or ownership enforcement.

## Explicitly owned collections

```lua
--!low
import "std/host"
import "std/buffer"
local value = zero<buffer.Buffer>()
buffer.initialize(addressOf(value), host.allocator())
assert(buffer.appendText(addressOf(value), "hello"))
print(buffer.view(addressOf(value)))
buffer.destroy(addressOf(value))
```

`std/memory.Allocator` contains a context pointer and two callbacks:
`allocate(context, bytes): ptr<u8>` and `release(context, pointer, bytes): void`.
Allocation failure returns null. The callback must return independent writable
storage and release only its own allocations. Host allocation uses malloc/free;
an OS can supply callbacks backed by its own heap. Never copy a live owning
buffer/vector and later destroy both copies; ownership is an explicit convention.

`std/buffer.Buffer` starts zero-initialized. `initialize` selects its allocator;
`reserve` grows storage, `append` copies bytes, `appendText` copies a string,
`view` borrows contents, `clear` resets length and `destroy` releases storage.
Growth preserves contents, rejects overflowing sizes and leaves storage intact
on allocation failure. Self-appending across reallocation is supported.
Bytes beyond `length` are not initialized or readable through the public view.

`std/vector.IntVector` shares that allocator contract and exports `initialize`,
`length`, `push`, `at`, `set`, `pop` and `destroy`. `at`/`set` and empty `pop`
assert their preconditions. This is a typed i64 collection, not a general-purpose
generic container system. Both collection modules support freestanding builds;
the QEMU fixture supplies a kernel heap and checks physical page accounting.

## Hosted files and line input

| Builtin | Contract |
| --- | --- |
| `fileOpen(path: string, mode: u8): u64` | Handle, zero on failure; mode 0 read, 1 truncate/write, 2 append, 3 existing read/write, 4 truncate read/write |
| `fileRead(handle, ptr<u8>, bytes): i64` | Bytes read, zero at EOF, -1 on error |
| `fileWrite(handle, ptr<u8>, bytes): i64` | Bytes written (possibly partial), -1 on error |
| `fileSeek(handle, offset: i64, origin: u8): boolean` | Origin 0 beginning, 1 current, 2 end |
| `fileFlush(handle): boolean` | Flush the host stdio buffer; does not promise power-loss durability |
| `fileClose(handle): boolean` | Close once; stale/invalid handles are rejected |
| `readLine(ptr<u8>, capacity): i64` | Bytes excluding newline; -1 EOF/error, -2 line too long (remainder drained) |

Read/write/input buffers require low mode and valid caller-owned storage. No NUL
terminator is appended. Input removes an optional final CR. File operations are
binary and support 64-bit seeks, reject embedded NUL paths and track handle
generations. At most 255 handles may be open simultaneously. Serialize host I/O;
these runtime tables are not a concurrency API.

`std/host` adds `allocator()`, `readFile(path, buffer)` (append complete contents)
and `writeFile(path, text)`. A failed read restores the destination's original
logical length. Write truncates its destination on open and is not an atomic
file replacement. Always inspect results and destroy owned buffers. These
functions and argument/input/file builtins are rejected for freestanding builds.

## Separate compilation

```lua
-- math.lasm
--!low
export function square(value: i64): i64 return value * value end
```

```lua
-- main.lasm
--!low
extern function square(value: i64): i64
print(square(7))
```

```sh
luasm build math.lasm --no-entry --emit-obj -o math.obj
luasm build main.lasm --object math.obj -o app.exe
```

On Linux choose `.o`/the desired executable name. `--object` is repeatable and
also works with freestanding executable linking. Hosted `--no-entry` object,
assembly and LLVM builds generate no main and reject top-level executable code.
The public boundary is explicit `export`/`extern` declarations with the supported
scalar C ABI; no automatic separately compiled import interface is implied.

Imported definitions have private object linkage. Multiple objects may import
the same source library without exporting duplicate private helpers. Compatible
C extern declarations shared across imports are emitted once; conflicting
signatures report `LSA2116`. Libraries precede the hosted runtime when linking,
so runtime references introduced by a library can resolve on Unix linkers too.
