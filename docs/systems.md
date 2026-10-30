# Kernel systems libraries

These libraries are LuASM source. CPU-specific entry stubs and privileged
operations are supplied by the C++ compiler/LLVM backend, with no external
assembly or C source required from a kernel author. The demo and all boot tests
currently use one x86-64 boot CPU. APIs require serialized ownership; there are
no SMP locks or TLB shootdowns yet.

## Page tables

`memory/paging.lasm` provides four-level x86-64 page-table ownership:

| API | Meaning |
| --- | --- |
| `clone(space, physicalOwner, sourceRoot)` | Deep-copy present table levels into owned physical frames; keep leaf mappings/flags |
| `map4k(space, virtual, physical, flags)` | Add a supervisor 4 KiB mapping; refuse replacement and huge-page paths |
| `translate(space, virtual)` | Return `{present, physical, pageSize, entry}` for 4 KiB, 2 MiB or 1 GiB leaves |
| `unmap4k(space, virtual)` | Remove a 4 KiB leaf and return its physical frame; leave table storage owned |
| `destroy(space)` | Free all table frames of an **inactive** space; payload frames stay owned by the caller |

Start with `zero<paging.Space>()`. A space tracks up to 512 owned table frames.
`WRITABLE` and `NO_EXECUTE` are exported flag constants; PWT, PCD and global leaf
flags are also permitted. Present is added automatically. Virtual addresses must
be canonical for 48-bit addressing, and page inputs must be 4096-byte aligned.
Physical inputs must denote valid machine physical memory. NX requires the CPU's
NX capability and enabled EFER.NXE. No user-mode mappings are exposed yet.

The library validates arithmetic/layout inputs; the source hierarchy must be a
valid, readable boot hierarchy accessible through HHDM. Clone failures and partial
mapping failures release newly allocated table frames. Non-present entries with
software data are not replaced. The caller disables concurrent access, changes
CR3 when activating a root, and invokes `invalidatePage` after active leaf changes.
Destroying the current root is invalid. The demo rejects LA57/PCID, clones the
Limine hierarchy, switches CR3 and keeps the inherited kernel/stack/HHDM/device
mappings. Bootloader data and old page tables are not reclaimed automatically.

Tests check large leaves, deep-copy independence, permissions, failed map/clone
rollback, table accounting and real alias/TLB behavior after the CR3 switch.
The hardware layout follows Intel's [system programming manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html).

## Interrupt ABI and descriptors

```lua
--!low
import "../../libraries/hardware/interrupts"

interrupt function handler(frame: ptr<interrupts.Frame>): void
    -- Handle an interrupt without a CPU error code, then return normally.
end

interrupt function fault(frame: ptr<interrupts.Frame>, error: u64): void
    -- The second form consumes the CPU-pushed error code before IRETQ.
end
```

Interrupt functions require freestanding x86-64, return `void` and take a frame
pointer with an optional `u64` error code. They cannot be called as ordinary
functions. `functionAddress(handler): usize` obtains an entry address for an IDT
gate; it also works with ordinary declared/imported functions and extern symbols.
Ordinary callable values use the separate `fn<Result, Parameters...>` type;
interrupt entry points cannot become ordinary callable values.

The compiler generates a wrapper that saves all 15 non-stack GPRs, clears DF,
aligns the stack for the System V body, passes the hardware frame/error, restores
registers, discards the error code when present and executes IRETQ. The ordinary
LuASM body may call other integer-only LuASM functions. Freestanding compilation
uses `-mgeneral-regs-only` and disables the red zone. SIMD/FPU state, privilege
transitions and task switching are outside this ABI's current scope.

`interrupts.Frame` has five `u64` fields: `ip`, `cs`, `flags`, `sp`, `ss`.
`hardware/interrupts.initialize(normalAddress, errorAddress)` requires IF=0 and
installs a kernel GDT, TSS, 16 KiB double-fault IST stack and all 256 IDT gates.
`setGate(vector: u8, address: usize, ist: u8)` replaces a gate. The caller must
match the handler signature to whether that vector pushes an error code.
`LSA2114` rejects invalid declarations and direct calls. INT3 register preservation
and a returning page-fault handler are tested on the real guest CPU.

## Legacy IRQs, timer and keyboard

`hardware/pic.initialize()` remaps the 8259 pair to vectors 32..47 and masks all
IRQs. On the single boot CPU it disables an inherited xAPIC to permit legacy
PIC delivery; x2APIC is rejected. This is a legacy PC backend, not an APIC/SMP
interrupt router. `setMask(irq, masked)` validates IRQ 0..15 and unmasks the slave
cascade when needed. `eoi(irq)` handles master/slave ordering and spurious 7/15.

Install `functionAddress(timer.onTick)` at vector 32. `timer.start(hz)` programs
PIT channel 0 in mode 3 for 19..1193182 Hz (integer-divisor approximation), then
unmasks IRQ0. `ticks()` reads a volatile 64-bit counter; `stop()` masks IRQ0.
The example waits for actual IRQs using STI/HLT with a disabled-interrupt condition
check, then stops the timer before the final halt.

`hardware/keyboard.initialize()` configures the i8042 first port and keyboard
set 2 with controller translation to set 1. It uses bounded waits and ACK/resend
handling. Initialize it with IRQ1 masked and IF=0; install `keyboard.onKey` at
vector 33 before unmasking IRQ1. The IRQ drains input into a 63-character queue;
`read()` returns one byte or zero, preserving the caller's interrupt-enable state.
`dropped()` counts queue overflow. The decoder in `input/set1.lasm` can run hosted:
US ASCII, make/break, independent shifts, caps, control letters and extended/Pause
prefixes. USB keyboards, layout selection and terminal escape sequences need
separate drivers/policies. QEMU tests inject a real virtual PS/2 key at O0/O2;
a wrong-key boot is explicitly rejected.

## Heap

`memory/heap.lasm` exposes `Heap`, `initialize(heap, physicalOwner)`,
`allocate(heap, bytes)`, `allocateZeroed(heap, bytes)` and `release(heap, pointer)`.
Start with `zero<heap.Heap>()`. Allocation returns a 16-byte-aligned `ptr<u8>`;
zero sizes, integer overflow, fragmentation or exhaustion return null. Release
returns a boolean and rejects foreign, interior and already released pointers.

Chunks come from contiguous physical runs and are accessed through HHDM. Blocks
split on allocation and adjacent free blocks coalesce. Entirely free chunks are
returned to the physical allocator. Small allocations share a page; large ones
use multi-page runs. A large allocation can fail under physical fragmentation
even when enough noncontiguous frames remain. `pages` and `allocations` are
read-only accounting fields. Serialize access and stop using released pointers.
Physical `allocateRun(owner, count)` and `releaseRun(owner, base, count)` are also
available independently; run validation/release is atomic with respect to failure,
not concurrent CPUs.

## RAM filesystem

`filesystem/ramfs.lasm` owns file contents through a heap. Initialize
`zero<ramfs.FileSystem>()` with `initialize(fs, heap)`. It supports 64 entries
including root, names of 1..63 printable ASCII bytes, and files up to 1 MiB.
Paths are absolute and normalized; empty components, trailing slashes, `.` and
`..` are rejected. This is a volatile in-memory filesystem with no disk driver.

| API | Result |
| --- | --- |
| `open(fs, path)` / `create(fs, path, directory)` | `u64` handle, zero on failure |
| `read(fs, handle, offset, buffer, count)` | Bytes copied, bounded by file size |
| `write(fs, handle, offset, buffer, count)` | Boolean; binary data and zero-filled gaps supported |
| `truncate(fs, handle, newSize)` | Shrink only; erased tail cannot reappear on later extension |
| `rename(fs, oldPath, newPath)` | No overwrite; retain handles and reject directory cycles |
| `remove(fs, path)` | Reject root and nonempty directories |
| `size`, `isDirectory`, `child(fs, parentHandle, ordinal)` | Metadata and directory enumeration |
| `name(fs, handle, buffer, capacity)` | Name bytes copied, without a NUL terminator |
| `destroy(fs)` | Release all contents and end this filesystem lifetime |

Handles contain a slot generation so deletion/reuse rejects stale handles.
Handles belong to one filesystem instance/lifetime. Writes allocate replacement
storage before releasing old contents, including aliased input during growth;
allocation failure preserves the existing file. Callers supply valid buffers and
serialize operations. No persistence, permissions or VFS mount dispatch is implied.

## Framebuffer

`graphics/framebuffer.lasm` now has `valid`, `putPixel`, `fillRect` and `clear`.
Rectangles clip without overflowing unsigned coordinates; pitch is in bytes and
row padding stays untouched. Validation rejects null/empty surfaces, short pitch
and address-span overflow. The caller supplies a real mapping and chooses its
32-bit pixel format; validation cannot prove a raw pointer is mapped. Host guard
tests and the guest screenshot verify clipping, padding and actual RGB pixels.
