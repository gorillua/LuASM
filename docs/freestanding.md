# Freestanding compilation and linking

```sh
luasm build kernel.lasm --freestanding --target x86_64-elf --emit-obj -o kernel.o
luasm build kernel.lasm --freestanding --target x86_64-unknown-none-elf --entry start --linker-script kernel.ld -o kernel.elf
luasm build kernel.lasm --freestanding --linker-script kernel.ld --format binary -o kernel.bin
```

The two bare-metal target spellings normalize to `x86_64-unknown-none-elf`.
Unsupported architectures/triples are rejected. Hosted target aliases remain
`windows-x86_64` and `linux-x86_64` (only the current host is supported).
Freestanding builds require low mode, and cannot be executed by `luasm run`.
An ELF executable needs a defined zero-argument void entry function; its default
name is `kernelMain`. Objects may contain ordinary library functions without an
entry. Top-level executable statements are rejected; typed globals use static
initializers and linker relocations, with no runtime startup constructor calls.

## Dependency contract

No hosted LuASM archive, C/C++ standard library, CRT startup or host SDK is linked
into bare-metal artifacts. Clang compiles integer-only code without a red zone,
stack protector, unwind tables, PIC or vector-register code generation, using
the kernel code model. Generated functions carry `noredzone`, `nounwind`,
`no-builtins` and `no-stack-arg-probe` attributes. LLD links ELF directly with
`--no-undefined`; it does not invoke a host compiler driver or startup objects.

`print`, `alloc`, `realloc` and `free` produce `LSA1100` in freestanding mode.
Floating-point types/values produce `LSA1101`; this avoids implicit math/soft-float
dependencies until floating-point context support is designed. Strings and
integer aggregates are supported. Runtime checks become `llvm.trap` (UD2 on x86),
not calls into the hosted runtime. A kernel must install exception handling before
it can recover from such traps; LuASM does not yet supply an IDT.

Clang/LLVM may lower aggregate copies into memory library calls even for
freestanding code. LuASM therefore emits weak, self-contained byte-loop versions
of `memcpy`, `memmove`, `memset`, `memcmp`, plus internal string comparison.
They require no allocation, OS calls or C runtime. Their implementations are
retained and excluded from loop-idiom optimization to prevent recursive libcalls.
A final static object can override the weak memory definitions. This is explicit
compiler support embedded in the module, not the hosted runtime archive.
[Clang freestanding requirements](https://clang.llvm.org/docs/UsersManual.html#freestanding-builds).

Objects are tested with a large aggregate and dynamic checked operations at
`-O0/-O2`; linked ELF symbol tables are inspected for undefined global symbols.

## Tools and output

`--emit-llvm`, `--emit-asm`, `--emit-obj`, `--emit-ast` work in freestanding mode.
`--format elf` is the default executable format; `--format binary` uses
llvm-objcopy to produce a flat image. A flat image alone is not a BIOS bootloader.
Choose a linker script for a deliberate physical/virtual memory layout.

`--clang`, `--linker`, `--objcopy` select tools directly, without shell expansion.
`--linker` requires a freestanding executable build. `--objcopy` requires a flat
binary build. `--library PATH` is repeatable and passes explicit static archives
to hosted or freestanding executable linking. Linker scripts and input archives
must already exist. The output must not overwrite input source files, imported
sources, archives, the compiler executable or the linker script.

## Extern ABI

```lua
--!low
extern function combine(a: u64, b: u64): u64
print(combine(4, 2))
```

Link its C-compatible definition with `--library support.a`. Extern declarations
require explicit parameter and return annotations, use unmangled symbols and
support numeric, boolean, pointer and void returns. Aggregate/string/nil ABI,
variadic calls, DLL loading and C++ mangled names are unsupported. Typed scalar
callbacks use `fn<Result, Parameters...>` and the same C ABI extensions.
Small integer/boolean ABI extensions are emitted on declarations and calls.
The tests build actual C-linkage static archives and call signed 8-bit, unsigned
16-bit, boolean and 64-bit functions. ELF archives are also cross-linked.

Exported aggregate-valued LuASM functions use LuASM's LLVM-level aggregate ABI;
they must not be assumed compatible with a C compiler's aggregate ABI.
Global declaration prefixes `luasm_` and `__luasm_` are reserved for compiler support.
