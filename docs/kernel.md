# Kernel milestone: Limine x86-64 and serial output

The higher-half ELF kernel has been booted with Limine 12.9.4 and QEMU 11.1.0
under UEFI/TCG at both `-O0` and `-O2`. Tests require its serial greeting and
verify the CPU's halted state through QMP. Additional tests verify physical page
allocation/reuse, a kernel-owned page hierarchy, interrupt/exception return,
PIT ticks, a heap, RAMFS, PS/2 input and a pixel from the real guest framebuffer.
Legacy BIOS/ISO boot has not been verified; xorriso is still optional and absent.

```sh
cmake --build build --target kernel
```

This produces `build/kernel.elf` using:

```sh
build/luasm build examples/low/kernel.lasm --freestanding --target x86_64-elf --boot-protocol limine --linker-script examples/low/kernel.ld -O2 -o build/kernel.elf
```

The entry function disables interrupts and initializes COM1. It obtains the
memory map, direct-map offset and framebuffer through `libraries/boot/limine.lasm`,
counts usable RAM, initializes the [physical-page allocator](memory.md), checks
two complete pages through HHDM, clones the boot page tables and switches CR3.
It verifies an alias mapping and its TLB invalidation, installs its own GDT/TSS/IDT,
checks saved registers across INT3, retries a page-faulting instruction after
mapping its page, receives timer IRQs and initializes the keyboard. It exercises
small/multi-page heap allocations and RAMFS directory/file operations before
drawing a 64-by-32 RGB rectangle. It then writes
`LuASM kernel: hello from x86-64!`, and remains in a halt loop. The greeting comes
after the checks, so reaching it demonstrates that the responses were usable.
UART output and pixel writes are ordinary LuASM library code. Failure paths
print a failure reason and halt without the success greeting.

## Boot contract

`--boot-protocol limine` emits aligned start/end request markers and a writable
base-revision-6 tag. The linker script keeps the requests in a writable,
non-executable load segment, separately from code and read-only strings. A
volatile revision check runs before the LuASM entry body; an unsupported
revision stops in a CLI/HLT loop. Use a Limine release supporting base revision 6.
The bootloader supplies long mode and a stack; the entry must never return.
These details follow the [Limine protocol](https://github.com/limine-bootloader/limine-protocol/blob/trunk/PROTOCOL.md).

Metadata is emitted once per entry translation unit. Do not request it again
when compiling auxiliary objects. `--boot-protocol limine` requires freestanding
ELF output. It does not turn a flat binary into a legacy 16-bit BIOS boot sector.
Requests are typed mutable globals in `.limine_requests`, retained through
`llvm.used`. The library reads bootloader-written response pointers through
volatile views to prevent constant folding of their initial null values.
The allocator uses usable RAM only and reserves its own bitmaps there. Its
self-test checks distinct pages, zeroed allocation, release, double-free
rejection, reuse and restored free-page counts. Inherited mappings remain in the
kernel-owned hierarchy; old bootloader tables/data are not reclaimed. See
[system interfaces and limitations](systems.md) and [macro/assembly syntax](macros-and-assembly.md).

## Packaging and actual boot verification

### Portable Windows UEFI test (verified)

```powershell
.\tools\setup_boot_tools.ps1
python tools/kernel_image.py --compiler build/luasm.exe --limine-dir build/boot-tools/limine-binary --media uefi-dir --output build/kernel-uefi --qemu build/boot-tools/qemu/qemu-system-x86_64.exe --firmware build/boot-tools/qemu/share/edk2-x86_64-code.fd --firmware-vars build/boot-tools/qemu/share/edk2-i386-vars.fd --test --screenshot build/kernel-screen.png
```

The setup script downloads versioned archives from the official
[Limine releases](https://github.com/limine-bootloader/limine/releases/tag/v12.9.4),
[QEMU Windows distributor](https://qemu.weilnetz.de/w64/) linked by the
[QEMU project](https://www.qemu.org/download/), and
[7-Zip](https://www.7-zip.org/download.html). Downloads are pinned by SHA-256 or
SHA-512. The archives are extracted in `build/boot-tools`; installers are not run.
It does not change the registry or system PATH. Re-running it reuses checked downloads.

UEFI directory mode needs BOOTX64.EFI, QEMU and split CODE/VARS firmware. It uses
QEMU's [virtual FAT directory support](https://www.qemu.org/docs/master/system/images.html#virtual-fat-disk-images)
and needs no xorriso or Limine host installer. A disposable copy of the directory
and a private VARS copy are given to QEMU; the original firmware is unchanged.
The generated directory carries a `.luasm-image` marker and the tool refuses to
overwrite an unrelated nonempty directory.

Logs are saved beside the image: `.serial.log`, `.serial.qmp.log` and
`.serial.result.json`. The latter records greeting detection, `HLT=1`, register
state and elapsed time. `--log` chooses the serial log path, `--optimization=-O0`
selects an unoptimized build, and `--source` selects another kernel source.
`--screenshot` captures the halted guest through QMP, without a visible window.
Every test terminates its emulator, including failure/timeout paths.

Enable the same tests in CMake on Windows:

```powershell
cmake -S . -B build -DLUASM_BOOT_TESTS=ON -DLUASM_BOOT_MEDIA=uefi-dir -DLUASM_LIMINE_DIR="$PWD/build/boot-tools/limine-binary" -DLUASM_QEMU="$PWD/build/boot-tools/qemu/qemu-system-x86_64.exe" -DLUASM_FIRMWARE="$PWD/build/boot-tools/qemu/share/edk2-x86_64-code.fd" -DLUASM_FIRMWARE_VARS="$PWD/build/boot-tools/qemu/share/edk2-i386-vars.fd"
ctest --test-dir build --output-on-failure
```

The two boot configurations use independent output directories. Negative tests
also boot real kernels: a stale greeting must not pass, and a busy kernel that
prints the greeting but never halts must fail. The feature test checks a PNG
pixel against RGB(51,153,204), not merely a printed claim about drawing. It also
requires the allocator self-test marker and checks its counts and physical
addresses. Both optimization levels run the full memory self-test before their
final greeting.

### BIOS/UEFI ISO packaging (not verified here)

Obtain a local Limine binary distribution with its host `limine`/`limine.exe`
utility, install xorriso and qemu-system-x86_64, then run:

```sh
python tools/kernel_image.py --compiler build/luasm --limine-dir /path/to/limine-binary --output build/kernel.iso --test
```

On Windows use `--compiler build/luasm.exe` (or the appropriate configuration
subdirectory) and Windows-compatible tool paths. The script packages a BIOS/UEFI
optical image, invokes Limine's installer on that image only, and runs headless
QEMU with serial capture. A boot test succeeds only when the expected kernel
greeting appears and the CPU reaches HLT; a timeout without both fails. The dependencies are
checked before creating an image; the script downloads nothing. Its packaging
layout follows the [official template](https://github.com/limine-bootloader/limine-c-template/blob/trunk/GNUmakefile)
and [configuration format](https://github.com/limine-bootloader/limine/blob/v12.x/CONFIG.md).

For ISO tests configure `-DLUASM_BOOT_TESTS=ON -DLUASM_BOOT_MEDIA=iso`
and `-DLUASM_LIMINE_DIR=/path/to/limine-binary`.
Ordinary compiler tests do not require these optional boot dependencies.
