# Macros and inline assembly

## `#` token macros

Macros expand before parsing and type checking. Definitions are private to each
source file; imports continue to export typed declarations. Examples:

```lua
--!low
#define PAGE_BYTES 4096
#define ADD(a,b) ((a) + (b))
#define ENABLE_DIAGNOSTICS

global buffer: array<u8, #PAGE_BYTES> = zero<array<u8, #PAGE_BYTES>>()

function value(): u64
#ifdef ENABLE_DIAGNOSTICS
    return #ADD(20, 22)
#else
    return 0
#endif
end
```

Invoke object macros with `#NAME` and function macros with `#NAME(arguments)`.
There is no expansion of bare identifiers or string contents. Function parameters
follow the definition name immediately: `#define F(x) ...`; a space before `(`
instead defines an object macro whose replacement begins with a parenthesis.
An empty definition can serve as an `#ifdef` flag. `#undef NAME` permits a later
redefinition. Nested `#ifdef`, `#ifndef`, `#else` and `#endif` are supported.
Directives occupy one source line and must start that line's tokens.

Arguments respect parentheses, braces and brackets, and can contain other macro
invocations. Put generic expressions containing commas in an extra pair of
parentheses, such as `#IDENTITY((zero<array<u8, 2>>()))`. Expansion substitutes
tokens; it introduces no hidden variables or evaluation-once semantics. Macro
authors must parenthesize expressions and account for repeated side effects.

Undefined/recursive macros, duplicate definitions/parameters, incorrect argument
counts, unmatched conditionals and exceeded limits report `LSA4002`. Expansion
is bounded by depth 64 and one million processed tokens. Type errors in expanded
replacement tokens point to the invocation. Macro arguments retain their source
locations. `--!low`, when used, still goes at the start of the source file.

There is no arbitrary compile-time interpreter, `#if` expression evaluator,
token pasting, stringification, multiline continuation or exported macro namespace.
Use ordinary functions and global constants for typed reusable operations.

## `@asm` on freestanding x86-64

```lua
--!low
function swapBytes(value: u64): u64
    return @asm<u64>("bswap $0", "=r,0", value)
end

function spinHint(): void
    @asm("pause", "")
end
```

`@asm<T>(template, constraints, inputs...)` returns one integer or pointer.
Omitting `<T>`, or using `<void>`, means no output. Both strings must be literals
without NUL. The compiler emits LLVM side-effecting assembly in x86 AT&T syntax.
This facility requires both low mode and `--freestanding`; hosted programs are
rejected with `LSA1002`. Prefer typed hardware intrinsics when one exists.

Supported constraints are one `=r`/`=&r` or fixed GPR output, followed by register
inputs (`r` or fixed GPR), an optional tied `0` input matching the output type,
and clobbers. Fixed registers are `{ax}`, `{bx}`, `{cx}`, `{dx}`, `{si}`, `{di}`
and `{r8}` through `{r15}`. Clobbers use `~{register}`. Memory and flag clobbers
are added automatically. Stack/frame registers, memory/immediate operand
constraints, multiple outputs and aggregate operands are not supported.
There are at most eight input operands.

Use `$0`, `$1`, etc. for operand references and `$$` for a literal dollar sign.
The output, when present, is operand zero. Explicit hard-coded register writes
must have matching output/clobber constraints. The assembly must restore the
stack pointer and follow the compiler's integer-only ABI. Do not hide a function
call, nonlocal branch or interrupt return in ordinary inline assembly; use the
compiler's interrupt declaration for interrupt entry points.

`LSA4001` reports malformed templates/constraints, unsupported types, bad ties and
operand-count/reference errors. `check` validates these source-level rules.
Clang's assembler validates actual instructions during `build`; unsupported
instruction spellings produce a build failure, not a silent fallback. The
implementation uses [LLVM inline assembly](https://llvm.org/docs/LangRef.html#inline-assembler-expressions).

The tests compile object/assembly output at O0/O2 and reject invalid forms. The
real QEMU kernel uses `@asm` to verify saved registers across INT3 and to trigger
a page fault whose handler maps a page and successfully retries the instruction.
