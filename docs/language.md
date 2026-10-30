# Language semantics

Use `--!low` as the first non-whitespace line to opt into systems operations.
This enables explicit unsafe pointers and hardware intrinsics; it does not change
the compilation target by itself. Numeric locals default to `i64`/`f64` in low
mode, or `number` (binary64) otherwise. Function signatures in low mode require
parameter annotations and an explicit return type when returning a value.

## Integers and bitwise operations

`i8/u8`, `i16/u16`, `i32/u32`, `i64/u64` have their stated widths; `isize/usize`
are 64 bits on the supported targets. Add, subtract, multiply and integer casts
wrap modulo the destination width. Signed `/` truncates toward zero; `//` rounds
down; `%` has the divisor's sign. MIN/-1 wraps to MIN; the remainder is zero.
Zero divisors produce runtime error 70 (or a hardware trap when freestanding).

Low mode adds `&`, `|`, binary `~` (XOR), unary `~` (NOT), `<<`, `>>`.
`^` remains reserved for unimplemented exponentiation; it is not XOR.
Right shift sign-extends a signed left operand and zero-fills an unsigned one.
Shift counts may have any integer type and must be in `[0, bitWidth(left))`.
Constant invalid counts produce `LSA2110`; dynamic invalid counts trap. The
compiler checks counts before emitting LLVM shifts to avoid LLVM poison.
[LLVM shift semantics](https://llvm.org/docs/LangRef.html#shl-instruction).

Precedence, low to high: `or`, `and`, comparisons, `|`, `~`, `&`, shifts,
reserved concatenation, `+ -`, `* / // %`, unary operators, reserved power.
Supported compound operators are `+= -= *= /= //= %= &= |= <<= >>=`.
`~=` continues to mean inequality; write `x = x ~ y` for XOR assignment.

## Pointers, aggregates and alignment

The `nil` literal becomes a typed null pointer when a `ptr<T>` is expected:

```lua
--!low
local pointer: ptr<u8> = nil
function emptyPointer(): ptr<void> return nil end
pointer = nil
assert(pointer == nil and nil == pointer)
```

This applies to function arguments (including extern/DLL calls and typed callback
arguments), returns, assignments, globals and aggregate fields/elements. Pointer
equality and inequality accept `nil` on either side. The compiler emits a native
null pointer, with no runtime conversion.

Without a pointer context, `nil` retains its existing nil type: `local value = nil`
does not infer a pointer type. Nil-typed variables are not implicitly converted to
pointers. Nil does not implicitly become an integer, boolean or `fn` value;
pointer arithmetic/ordering with `nil` remains a type error. Use an explicit pointer
type when the compiler cannot infer a pointee type. Pointers still need explicit
boolean comparisons in conditions.

Fixed arrays are zero-indexed, have checked indexing, and are stored by value.
Structs use natural alignment with explicit padding. `packed struct` removes
inter-field padding; `struct S align(32)` increases alignment and trailing size.
Nested structs and arrays, `sizeof`, `alignof`, `offsetof`, `zero<T>()` and
compile-time integer array lengths from visible local constants are supported.

```lua
--!low
struct Point
    x: i32
    y: i32
end
local points: array<Point, 2> = {Point{x = 1, y = 2}, Point{x = 3, y = 4}}
local p = addressOf(points[0])
p[1].x = 7
p += 1
print(p.x, p - addressOf(points[0]))
```

`ptr<T> + integer`, `integer + ptr<T>`, and `ptr<T> - integer` scale by `sizeof(T)`.
Subtracting equal pointer types returns an `isize` element distance. Pointer
ordering compares unsigned addresses. `ptr<void>` and zero-size pointees cannot
be used in arithmetic/indexing. Raw pointer indexing has no bounds or lifetime
checks. Pointer difference is meaningful for addresses within the same allocation.
Pointer-to-struct `p.field` implicitly dereferences the pointer.

`load`, `store`, pointer indexing and pointer field access support
`ptr<volatile<T>>`. Nested accesses and `addressOf` preserve volatility.
An explicit pointer cast may remove it. Known local allocations use the declared
alignment; raw/packed memory accesses conservatively use alignment 1. The compiler
does not silently assume a raw pointer meets its pointee's natural alignment.

Locals declared `const` and numeric loop variables cannot be assigned or exposed
by address. A constant pointer binding still permits mutation of its pointee.

## Static global storage

Low mode supports module-scope declarations with mandatory types and initializers:

```lua
--!low
global const CAPACITY: usize = 256
global buffer: array<u8, CAPACITY> = zero<array<u8, CAPACITY>>() align(32)
global cursor: ptr<u8> = addressOf(buffer[0])
export global flags: u64 = 0 section(".kernel_state") align(8)

function clearFirstByte(): void
    store(cursor, 0)
end
```

Globals have static storage duration; functions and the hosted top-level body
share the same storage. `global const` makes the binding and stored aggregate
immutable, using the same address-taking restrictions as local constants.
Initializers are evaluated at compile time: literals, pure arithmetic/casts,
layout queries, zero/aggregate construction, preceding global constants, and
addresses of preceding mutable globals/fields/constant array indices. These
addresses produce linker relocations, not runtime startup code.

There are no runtime initializer calls or reads of mutable globals during static
initialization. Forward/self-address references are not yet supported. Imported
globals initialize in dependency order; declarations in one file initialize in
source order. Global constants can provide array lengths for later globals,
function signatures and locals. Struct field layouts currently resolve before
global initializers, so fixed arrays inside struct declarations still need
literal lengths.

`section("name")` and `align(N)` are optional suffix attributes. A section name
may contain ASCII letters, digits, `.`, `_`, `-`, `$`; `llvm.metadata` is reserved.
Explicit alignment must be a power of two from the natural alignment up to 4096.
Custom-section data is retained even if unreferenced, including at `-O2`; the
linker script must keep the relevant section when external section garbage
collection is used. Mark bootloader-written requests mutable and read them
through volatile pointers. `LSA2111` reports non-static initializers and `LSA2112`
reports invalid section/alignment attributes.

## Strings and hardware

Hosted programs can use `time()` and `wait()` without an import:

```lua
local started = time()       -- monotonic seconds since program startup
local elapsed = wait(0.5)    -- wait at least half a second, return elapsed seconds
print(time() - started, elapsed)
```

`wait()` defaults to approximately 1/30 second. `wait(0)` yields the native
thread. Durations must be finite, nonnegative, and at most 31,536,000 seconds;
invalid durations cause runtime error 70. Scheduling and clock resolution can
make the actual wait longer. These functions work in ordinary and low mode,
but require a hosted runtime and are unavailable in freestanding kernels.

`time()` measures elapsed time, not the current wall-clock date or hour. Unlike
Roblox's scheduler, `wait()` blocks the calling operating-system thread; calling
it on a GUI thread also blocks that window's message loop. GUI apps should use a
timer/event instead. Both names are reserved builtins, so existing declarations
named `time` or `wait` must be renamed.

Strings contain a pointer and byte length; embedded NUL bytes are supported.
`stringLength(text)` returns `usize` in either mode. Checked `stringAt` and
`stringSlice` also work in ordinary mode. Low-mode `stringData(text)` exposes a
`ptr<u8>` to the string's bytes. This is unsafe access to immutable backing
storage: do not write through it, and do not assume NUL termination.

All hardware intrinsics require low mode and the supported x86-64 target:

| Intrinsic | Signature / behavior | Hosted |
| --- | --- | --- |
| `cpuPause()` | processor pause hint | yes |
| `compilerBarrier()` | compiler memory clobber, no hardware fence | yes |
| `memoryFence()` | sequentially consistent hardware fence | yes |
| `cpuid(leaf, subleaf)` | two `u32` inputs; `array<u32,4>` in EAX, EBX, ECX, EDX order | yes |
| `readTSC()` | `u64`; not a serializing timestamp operation | yes |
| `disableInterrupts()`, `enableInterrupts()` | CLI / STI | no |
| `halt()` | one HLT; may resume after an interrupt | no |
| `in8/16/32(port)` | `port: u16`, returns `u8/u16/u32` | no |
| `out8/16/32(port, value)` | `port: u16`, matching-width value | no |
| `readCR0/2/3/4()` | returns `u64` | no |
| `writeCR0/3/4(value)` | `value: u64` | no |
| `readFlags()`, `readCS()` | `u64` RFLAGS / `u16` code selector | no |
| `readMSR(index)`, `writeMSR(index, value)` | `index: u32`, `value/result: u64`; caller validates MSR support | no |
| `invalidatePage(address)` | `address: usize`, local INVLPG | no |
| `loadIDT(descriptor)` | Pointer to a packed 10-byte `{limit: u16, base: u64}` | no |
| `loadGDT(descriptor, code, data)` | Same descriptor, two `u16` selectors; reloads CS/DS/ES/SS | no |
| `loadTaskRegister(selector)` | `selector: u16`, LTR | no |
| `breakpoint()` | INT3; requires an installed gate | no |
| `functionAddress(name)` | Raw `usize` address; typed callable values use `fn` | yes |

Privileged operations require an explicit freestanding build. They are never
executed by the host integration tests; tests compile and inspect their assembly.
Invalid MSRs, descriptors and privileged state can fault; these intrinsics expose
the machine operation and do not validate hardware configuration automatically.
See [kernel systems](systems.md) for interrupt declarations and tested libraries,
and [macros and assembly](macros-and-assembly.md) for `#` and `@asm` syntax.
BIOS services and a 16-bit target remain unsupported.
