# Getting started with LuASM

LuASM is a statically typed, Lua-inspired language that compiles to native code.
This guide uses the Windows x86-64 alpha. LuASM is a separate language and does
not run Roblox Luau scripts.

## 1. Install

Download the installer from the repository's Releases page when a release is
available. Run `LuASM-Setup.exe`. It installs for your Windows account and offers
to set `LUASM_HOME`, add LuASM to your user PATH, and install the VS Code
extension. You can leave these options selected.

The package includes the compiler, libraries, runtime and LLVM/Clang. Windows
executable builds also need Microsoft's C++ build tools and Windows SDK. Setup
tries a real compilation and points you to the Windows build tools if that fails.
VS Code is a separate download. The current alpha installer is unsigned.

After setup, restart your terminal and reload VS Code. In a new PowerShell window:

```powershell
luasm --version
```

If Windows cannot find `luasm`, restart the terminal or run the compiler using
its full path:

```powershell
& "$env:LOCALAPPDATA\Programs\LuASM\bin\luasm.exe" --version
```

## 2. Run a program

Create `hello.lasm`:

```lua
local greeting = "Hello, LuASM!"
print(greeting)
```

Run the program in a terminal:

```powershell
luasm run hello.lasm
```

Or open the file in VS Code and click **Run** in the editor. The extension also
provides highlighting, completion, hover information and compiler diagnostics.
Use **LuASM: Select Compiler** in the command palette if it cannot find LuASM.
Compilation and execution require a trusted workspace.

## 3. Build an executable

```powershell
luasm build hello.lasm -o hello.exe
.\hello.exe
luasm build hello.lasm --release -o hello-release.exe
```

`--release` selects `-O2`. LuASM emits LLVM IR, then invokes Clang and the
Windows toolchain to compile and link the program. `luasm check hello.lasm`
checks the program without producing an executable.

To create a project directory with a starting `main.lasm`:

```powershell
luasm new MyApp
luasm run MyApp
luasm build MyApp --release
```

Directory builds use `main.lasm` as the entry file and put build output under
`.luasm-build`. If you use VS Code, open the project folder and its `main.lasm`.

## 4. Variables and control flow

```lua
local count = 3

if count > 0 then
    print("Starting")
end

for i = 1, 3 do
    print(i)
end

while count > 0 do
    print(count)
    count -= 1
end
```

Locals have a fixed type; LuASM infers it where possible. `if`, `while`, numeric
`for`, `repeat`, `break`, and `continue` are available. Use `and`, `or`, `not`,
and `~=` for inequality.

## 5. Functions and imports

```lua
function greet(name: string): void
    print("Hello", name)
end

greet("world")
```

Imports provide a module namespace. Standard modules are included with the SDK:

```lua
import "std/text"
print(text.trim("  hello  "))
```

For project-local modules use a relative path, for example
`import "./helpers"`. Only exported declarations are accessible through the
namespace. See [modules](modules.md) for exports, aliases and search paths.

## 6. Low mode

Put `--!low` at the top of a file to enable LuASM's low-level features. It gives
you structs, typed pointers, fixed arrays, integer types and other systems tools.
It does not itself switch the compiler to a freestanding target.

```lua
--!low
struct Point
    x: i32
    y: i32
end

local point = Point{x = 10, y = 20}
local p = addressOf(point)
p.x = 30
print(point.x, point.y)
```

Pointers must refer to valid memory. A pointer can be compared with `nil`, but
an untyped `local p = nil` does not infer a pointer type. `zero<T>()` produces a
zero-initialized value of type `T`. See [language semantics](language.md) for
the exact pointer, integer and array rules.

## 7. Buffers and Base64

Many low-level libraries accept a buffer supplied by the caller. Initialize the
buffer with an allocator, check operations that can fail, and destroy it when done:

```lua
--!low
import "std/buffer"
import "std/base64"
import "std/host"

local output = zero<buffer.Buffer>()
buffer.initialize(addressOf(output), host.allocator())
assert(base64.encode("Hello World!", addressOf(output)))
print(buffer.view(addressOf(output)))
buffer.destroy(addressOf(output))
```

This prints `SGVsbG8gV29ybGQh`. Encoding appends to the output buffer; it does
not clear it. A view borrows its buffer's storage and can become invalid when the
buffer is resized, modified, or destroyed. See [additional libraries](expanded-libraries.md)
for math, random numbers, binary data, text, paths, URLs and HTTP.

## 8. Time and waiting

`time()` measures monotonic elapsed seconds since the process started.
`wait(seconds)` blocks the current native thread and returns the elapsed wait:

```lua
local start = time()
wait(0.5)
print("Elapsed:", time() - start)
```

Use `windows/clock` for calendar date and time. Do not call blocking `wait` or
synchronous HTTP on a GUI thread if the window must stay responsive.

## 9. Windows APIs

Windows DLL functions can be declared directly in low mode:

```lua
--!low
extern "kernel32.dll" function GetTickCount64(): u64
print(GetTickCount64())
```

Match the Windows function's ABI and types exactly. For example, Win32 `BOOL`
is generally represented by `i32`, while pointers can be `ptr<void>` or a more
specific pointer type. Strings in LuASM are not automatically converted to
NUL-terminated UTF-16 strings. See [DLL imports](dll-imports.md) for supported
signatures, aliases and Unicode helpers.

For a higher-level native window, try
[`simple_window.lasm`](../examples/windows/simple_window.lasm). See
[Windows libraries](windows-libraries.md) and [modern UI](modern-ui.md) for
controls, events, layout, and the retained UI library.

## 10. Freestanding projects

LuASM can compile x86-64 freestanding code, without the hosted runtime. This is
an advanced path requiring an entry point, linker configuration and an appropriate
boot environment; `--!low` alone is not enough. The installer includes the LLVM
tools used for the freestanding target, but does not bundle QEMU or a bootloader.
See [freestanding builds](freestanding.md) and [kernel guide](kernel.md).

## Troubleshooting

- **`luasm` is not found:** restart the terminal, check the PATH option, or use
  the full compiler path shown above.
- **Linking fails:** install Microsoft's C++ build tools and a Windows SDK. Re-run
  setup's check after installing them.
- **VS Code does not recognize the file:** reload VS Code, choose LuASM as the
  language mode, and check the extension is installed and enabled.
- **An import cannot be found:** check its spelling and whether it is relative to
  the source file or is one of the installed SDK modules.
- **Your program exits at `assert`:** an operation returned false; check its
  inputs and initialize required resources before use.
- **A native Windows DLL call crashes:** review the function's exact ABI, integer
  widths, pointer lifetimes, string encoding and return type.

When reporting a problem, include the LuASM version, Windows version, source
that reproduces it, the command used, and the full error message. Avoid posting
passwords, access tokens or other private data.
