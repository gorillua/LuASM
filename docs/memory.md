# Physical pages

`libraries/memory/physical.lasm` manages individual 4096-byte physical pages in
ordinary LuASM. It has no hosted allocator, assembly source or C runtime dependency.
The kernel example initializes it from Limine's memory map and HHDM response.
It uses only entries with `kind == 0` (usable RAM). Limine defines those entries
as free RAM, distinct from the executable, boot information and reserved data;
the HHDM maps them. See the [protocol's memory map](https://github.com/limine-bootloader/limine-protocol/blob/trunk/PROTOCOL.md#memory-map-feature)
and [memory layout](https://github.com/limine-bootloader/limine-protocol/blob/trunk/PROTOCOL.md#memory-layout-at-entry).

## Interface

All functions take `ptr<physical.Allocator>` as their first argument. Create
exactly one owner for a set of pages, initially `zero<physical.Allocator>()`.

| Operation | Result |
| --- | --- |
| `initialize(owner, map, directOffset)` | Boolean; validates the map, reserves bitmap storage and initializes the owner |
| `allocate(owner)` | Physical `u64` address; zero on exhaustion or uninitialized/null owner; contents unspecified |
| `allocateZeroed(owner)` | Same, with all 4096 bytes cleared before returning |
| `mapped(owner, physical)` | `ptr<u8>` through the supplied direct mapping; null for invalid or currently free pages |
| `release(owner, physical)` | Boolean; makes an owned allocated page available again |

For example, after the initialization in `examples/low/kernel.lasm`:

```lua
local owner = addressOf(pageAllocator)
local frame = physical.allocateZeroed(owner)
if frame ~= 0 then
    local bytes = physical.mapped(owner, frame)
    bytes[4095] = 42
    local released = physical.release(owner, frame)
end
```

The exported constants are `PAGE_SIZE: u64 = 4096` and `MAX_REGIONS: usize = 128`.
Owner fields are implementation state; callers may read `initialized`, `count`,
`totalPages`, `freePages`, `metadataPages` and `offset`, but must not modify them.
`totalPages` excludes bitmap storage and physical page zero. It stays constant;
`freePages` decreases on allocation and increases on successful release.

## Storage and validation

Each usable region is rounded inward to whole pages. Physical page zero is
excluded so zero can consistently mean allocation failure. For a region with
`N` remaining pages, the first `ceil(N / 32768)` pages hold its bitmap. Bits for
these metadata pages are permanently occupied. Regions too small to contain
both metadata and a payload page are skipped. This avoids a bitmap spanning
physical address holes and does not require an existing heap.

Initialization rejects null response/entry pointers, an empty or unsorted map,
address arithmetic overflow, overlaps involving usable RAM, an unaligned HHDM
offset, more than 128 productive regions, and a map without allocatable pages.
Overlaps between non-usable regions are permitted. All structural validation
finishes before any bitmap is written. A rejected initialization leaves both the
owner and usable RAM unchanged; initializing an already initialized owner also
fails without losing its allocation state.

Allocation uses a cursor in each region; releasing a lower page moves its cursor
back so the page can be reused. Release and mapping check alignment, region
membership, metadata exclusion and the current allocation bit. A double release
or a release of a never-allocated page fails without changing the counters.
Neither allocation failure nor validation failure traps deliberately.

## Ownership and current limits

This is a single-owner allocator for boot and early kernel use. Serialize every
operation; the example runs on the boot CPU with interrupts disabled. Do not copy
an initialized owner, initialize another owner over the same RAM, or release a
page still used by another subsystem. The owner and the supplied map structures
must reside outside the RAM described as usable during initialization.

Response pointers must already refer to valid readable storage, and
`physical + directOffset` must provide writable mappings of all managed pages.
The library checks integer overflow but cannot validate arbitrary pointers or
repair missing page tables. It retains its own region descriptions, not the
memory-map response pointers. HHDM mappings must remain valid for its lifetime.
`release` does not remove a mapping or erase contents; stop using old pointers
before releasing their page. Use `allocateZeroed` when previous contents must
not be observed by a new owner.

`allocateRun(owner, count)` and `releaseRun(owner, physical, count)` additionally
manage contiguous runs within one region. A failed run release changes no bits.
Runs never include metadata or cross physical holes. They back the
[coalescing heap](systems.md#heap). The [paging library](systems.md#page-tables)
also uses this allocator for kernel-owned tables. There are no SMP locks or
automatic reclamation of bootloader/ACPI memory.

## Verification

`compiler.physical_pages` executes the same library at `-O0` and `-O2` against
page-aligned host buffers and synthetic memory maps. It checks reserved-memory
guards, partial pages, allocation exhaustion, every byte of distinct pages,
zeroing on reuse, invalid/double releases, malformed maps with no partial writes,
the 128-region limit, and a region needing two bitmap pages.

The real kernel boot tests also allocate two pages, read/write all their bytes
through volatile HHDM pointers, release/reallocate one, check its zeroed contents
and its neighbor, then release both and verify accounting. Their success marker
precedes the final serial greeting. The feature test checks reported page counts
and aligned, distinct physical addresses as well as the framebuffer screenshot.
