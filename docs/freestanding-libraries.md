# Arena allocation and framebuffer text

These modules work in hosted low mode and freestanding x86-64 ELF. They neither
allocate from the hosted heap nor invoke Windows APIs. Import the installed
libraries directly as `memory/arena` and `graphics/console`.
The installed [console example](../examples/low/console.lasm) combines both with
`std/buffer`. From the repository, build a bootable image with:

```powershell
python tools/kernel_image.py --compiler build/luasm.exe --limine-dir build/boot-tools/limine-binary --media uefi-dir --source examples/low/console.lasm --output build/console-demo
```

## `memory/arena`

```lua
--!low
import "memory/arena"
local storage = zero<array<u8, 4096>>()
local scratch = zero<arena.Arena>()
assert(arena.initialize(addressOf(scratch), addressOf(storage[0]), 4096))
local checkpoint = arena.mark(addressOf(scratch))
local bytes = arena.allocateZeroed(addressOf(scratch), 128, 32)
assert(address(bytes) ~= 0 and address(bytes) % 32 == 0)
-- Use bytes here, before rewind invalidates them.
assert(arena.rewind(addressOf(scratch), checkpoint))
```

Zero-initialize `Arena` before first initialization. The caller owns and keeps
both the arena and backing storage alive. `allocate(arena, bytes, alignment)`
accepts positive lengths and power-of-two alignment; it aligns the actual pointer
even if the supplied storage begins unaligned. Null/empty inputs, exhaustion and
overflow fail without advancing `used`. `allocateZeroed` additionally clears the
returned range. There is no per-allocation release.

`Mark` records the arena address, generation and offset. `rewind` accepts only a
mark from that arena's current generation and no farther than its current used
offset. A rewind, reset or reinitialize invalidates **all** previous marks. A
mark is single-use, and marks do not support nested rollback. `reset` invalidates
all allocations; callers must stop using those pointers/views first. Old bytes
are not wiped by reset/rewind. State is single-threaded and manually owned.

`arena.allocator(arena)` adapts to `std/memory.Allocator`, with 16-byte alignment.
Its release callback is a no-op; every buffer growth consumes additional arena
space until reset. Destroy all buffers/vectors that use it **before** resetting,
rewinding or reinitializing the backing arena. This adapter is for temporary
batch workloads, not a replacement for a reclaiming heap.

## `graphics/console`

```lua
--!low
import "graphics/console"
-- surface is a validated graphics/framebuffer.Framebuffer supplied by the caller.
local terminal = zero<console.Console>()
assert(console.initialize(addressOf(terminal), surface, 2, 0xE2E8F0, 0x0F172A))
console.writeLine(addressOf(terminal), "LuASM kernel ready")
console.write(addressOf(terminal), "Free pages: ")
console.writeUnsigned(addressOf(terminal), freePages, 10)
```

Supply a valid mapped 32-bit framebuffer whose native pixel format matches the
chosen colors. `initialize` copies the descriptor and clears visible pixels,
but does not own or free the mapping. It validates geometry/pitch/overflow and
requires at least one 6×8 character cell at scale 1..8. `columns`/`rows` and
`column`/`row` report the text grid and cursor (zero based). Do not mutate these
fields arbitrarily. Keep the framebuffer alive while using the console.

The original bitmap font in `graphics/font5x7` covers printable ASCII 32..126.
Non-ASCII bytes render `?`; this is not a Unicode terminal. `put` handles LF,
CR, backspace within the current line, and tabs at four-column stops. `write`
prints a byte string, `writeLine` adds LF, and `writeUnsigned` handles decimal
and hexadecimal values through u64's maximum. Each character clears its cell
before drawing; reaching the right edge immediately advances/wraps the cursor.
Reaching the bottom scrolls upward by one text line. `clear` resets the cursor;
`colors` affects subsequent drawing/erasing without recoloring existing glyphs.

Drawing and scrolling access mapped pixels through volatile pointers. Pitch
padding is preserved, including when scrolling. The module has no heap use,
interrupt output synchronization, ANSI escapes or hardware discovery. Serialize
access yourself; it is intended for kernel diagnostics and simple text UIs.

Native O0/O2 tests verify alignment, exhaustion/overflow, foreign/stale marks,
allocator adapters, glyphs, wrapping, scrolling and guard bytes around pitched
framebuffers. `tests/freestanding_graphics.lasm` runs those same checks in QEMU,
then prints onto the actual Limine framebuffer at O0/O2. The captured test screen
is `build/graphics-console.png` when requested with `--screenshot` in
`tools/kernel_image.py`. These modules do not require a particular bootloader;
only that test obtains its framebuffer through Limine.
