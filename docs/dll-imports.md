# Direct Windows DLL imports

Declare Windows API functions directly in low-mode LuASM. No per-API C++ wrapper,
import library, header translation step or additional linker argument is needed.

```lua
--!low
extern "kernel32.dll" function GetTickCount64(): u64
extern "kernel32.dll" function Sleep(milliseconds: u32): void

local started = GetTickCount64()
Sleep(10)
print(GetTickCount64() - started)
```

Build/run as usual: `luasm run main.lasm` or `luasm build main.lasm -O2`.
These declarations require hosted Windows x86-64 and report LSA1103 for other
targets. Ordinary `extern function` continues to refer to a symbol supplied by
an object/archive at link time. `extern "..." function` uses run-time DLL lookup.

## Names and exports

The declared name is the DLL entry-point name unless an alias is provided:

```lua
extern "kernel32.dll" function sleep(milliseconds: u32): void = "Sleep"
extern "user32.dll" function screenMetric(index: i32): i32 = "GetSystemMetrics"
```

Entry-point names are case-sensitive ASCII strings. Explicit `A`/`W` suffixes
are needed for APIs that C headers normally select through macros. C++ overloaded
exports need their actual exported symbol name, or a DLL that exposes a C ABI.
Named function exports are supported; ordinal/data imports are not.

`export extern "kernel32.dll" function ticks(): u64 = "GetTickCount64"`
lets other LuASM modules import the declaration. Names remain scoped to their
source module; two modules may use the same friendly name for different DLLs.
Imported function values and `fn<...>` callbacks work with these declarations.
The DLL wrapper is internal to the generated module, not a new DLL export.

## Resolution and cost

The first call loads the library with `LoadLibraryExW` and resolves the named
export with `GetProcAddress`. Later calls use an atomically cached pointer;
there is no repeated name lookup, marshalling framework or reflection. Normal
LLVM optimization can inline the small LuASM wrapper. Concurrent first callers
are supported. Loaded modules stay loaded for the process lifetime so cached
function pointers remain valid. Never manually unload a module backing an import.

Unused imports do not load a DLL or report a missing-export error. A failed
first call prints the library, symbol and Windows error number, then exits 70;
this release has no recoverable optional-import API. The resolver preserves the
calling thread's Win32 last-error value, including a first call to an imported
`GetLastError` itself. Call `GetLastError` immediately after the Windows API that
sets it, before invoking unrelated runtime/library functions.

For a bare name, lookup uses the executable directory and System32. It does not
search the working directory or PATH. An explicit relative path is resolved
against the executable directory. Absolute paths are supported, including UTF-8
names. Path-qualified DLL dependencies search the DLL's directory and System32.
This follows Microsoft's documented
[LoadLibraryEx search flags](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibraryexw).
When using `luasm run`, the executable lives in a temporary directory; build an
executable beside a custom DLL or declare its absolute path for that workflow.

## Windows types

Use the `nil` literal for a null pointer argument. The declaration supplies its
pointer type, so `GetModuleHandleW(nil)` works when its parameter is `ptr<u16>`.
You can also write `local owner: ptr<void> = nil` and compare pointers with
`owner == nil` or `owner ~= nil`. An unannotated `local owner = nil` retains the
separate nil type and does not infer a pointer.

You declare the exact native signature; the compiler cannot inspect a DLL to
infer its types. Windows x64's unified calling convention handles ordinary Win32
and CALLBACK functions. Scalars and pointers are passed directly:

| Windows type | LuASM type |
| --- | --- |
| `BOOL`, `LONG`, `HRESULT`, `int` | `i32` |
| `DWORD`, `UINT`, `ULONG` | `u32` |
| `BYTE`, `WORD`, `WCHAR` | `u8`, `u16`, `u16` |
| C/C++ `bool` | `boolean` |
| `SIZE_T`, `ULONG_PTR` | `usize` |
| `HANDLE`, `HWND`, `HMODULE` | `ptr<void>` |
| `LPCWSTR`, `LPWSTR` | `ptr<u16>` |
| `LPCSTR`, `LPSTR` | `ptr<u8>` with explicitly terminated storage |
| `LPVOID` | `ptr<void>` |
| Function-pointer parameter | Exact `fn<Result, Parameters...>` signature |
| `void` return | `void` |

In particular `BOOL` is **32 bits**, distinct from C++ `bool`. Struct parameters
should be pointers to correctly laid-out low-mode structs. By-value struct
arguments/returns, variadic calls, COM method declarations and automatic string
or buffer marshalling are not supported. Buffer lengths are native API units:
some count bytes, others UTF-16 code units. Windows APIs cannot check the memory
lifetime/size implied by a caller's pointer. Follow each API's contract.

## Unicode helpers

`windows/ffi` provides strictly validated, explicitly owned UTF-16 strings:

```lua
--!low
import "windows/ffi"

extern "user32.dll" function messageBox(owner: ptr<void>, contents: ptr<u16>,
    title: ptr<u16>, flags: u32): i32 = "MessageBoxW"

local title = ffi.wide("My LuASM app")
local contents = ffi.wide("Hello — 日本 😀")
assert(title.ok and contents.ok)
messageBox(zero<ptr<void>>(), ffi.data(contents), ffi.data(title), 0)
ffi.destroy(addressOf(contents))
ffi.destroy(addressOf(title))
```

`wide` returns a `WideString` with an `ok` flag and hosted-allocator storage. Its
data includes a trailing UTF-16 NUL; `length` excludes that terminator and counts
code units, not Unicode scalar values. Invalid UTF-8, overlong sequences,
surrogate code points and embedded NUL fail with `ok=false` and no allocation
remaining. `data` returns null for a failed string. Destroy each owner once;
copies borrow the same allocation and must not be destroyed separately.

For an API writing into `array<u16, N>`, use
`ffi.fromWide(pointer, maximumUnits, outputBuffer)`. It requires a NUL within the
given bound, validates surrogate pairs, and replaces an initialized
`std/buffer.Buffer` only on success. The original output remains intact on
failure. The conversion implementation is LuASM code and needs no per-API C++.

The complete [DLL example](../examples/windows/dll_imports.lasm) calls clocks,
sleep and `GetModuleFileNameW` directly. Add `-- --message` when running it to
also display a Unicode message box. Tests cover O0/O2 Windows calls, aliases,
scoped imports, missing libraries/exports, UTF-16, narrow/float/boolean ABI and
concurrent first resolution from native callback threads.
