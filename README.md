# LuASM

LuASM is an experimental language with Lua-inspired syntax that compiles to
native code. It is designed for Windows applications and x86-64 systems projects.
LuASM is a separate language; it does not run Roblox or Luau scripts.

```lua
local name = "LuASM"
print("Hello from", name)
```

## Windows alpha

The Windows x86-64 installer includes the LuASM compiler, runtime, standard and
Windows libraries, examples, documentation, Clang/LLVM tools, and a VS Code
extension. The compiler is distributed as an executable.

When a release is available, download `LuASM-Setup.exe` from this repository's
[Releases page](https://github.com/gorillua/LuASM/releases) and run it. The installer can add LuASM to your user PATH and
install the extension in VS Code. Restart your terminal and reload VS Code after
installation.

To build Windows executables, install Microsoft's C++ build tools and a Windows
SDK separately if they are not already installed. The setup provides a link and
checks whether it can build a program. VS Code itself is a separate installation.

Save the example above as `hello.lasm`, then click **Run** in VS Code or use:

```powershell
luasm run hello.lasm
```

Installed examples and guides are in `%LOCALAPPDATA%\Programs\LuASM\share\luasm`.
Uninstall using Windows **Installed apps** or the `Uninstall.exe` in the LuASM
installation folder. Your projects and the VS Code extension remain after removal.

## Documentation

- [Getting started](docs/getting-started.md) and [Windows setup](docs/windows-setup.md): installation, your first program,
  variables, functions, low mode, buffers and troubleshooting.
- [Language semantics](docs/language.md) and [modules](docs/modules.md): types,
  pointers, strings, globals, imports and exports.
- [Application libraries](docs/expanded-libraries.md): math, text, binary data,
  Base64, HTTP and Windows helper modules.
- [Windows DLL imports](docs/dll-imports.md), [Windows libraries](docs/windows-libraries.md)
  and [modern UI](docs/modern-ui.md): APIs, windows, controls and events.
- [Freestanding builds](docs/freestanding.md) and [freestanding libraries](docs/freestanding-libraries.md):
  x86-64 kernels and low-level libraries.
- [Examples](examples): a native window, a library tour and an HTTP client.

## License

LuASM can be used free for personal and commercial projects. The compiler is
distributed as a binary; see the [LuASM binary license](LICENSE.md). Bundled
LLVM/Clang and other third-party components have their own license terms.

## Current limitations

- This is an alpha; expect changes and bugs.
- Windows x86-64 is the locally tested desktop platform. Linux has not been
  verified in this environment.
- The installer is unsigned and Windows may display a warning.
- LuASM can compile freestanding x86-64 kernels, but it is not a complete OS.
- The compiler's source code is not included in this distribution. The libraries
  and examples include LuASM source files, and the package includes separately
  licensed LLVM components and the VS Code extension.

Please report installation problems or language bugs in this repository's
Issues section. Include your Windows version, what you tried, and the error.
