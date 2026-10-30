# Windows setup and removal

LuASM's Windows installer targets x86-64 Windows. It installs the compiler,
runtime, libraries, examples, documentation, Clang/LLVM tools and the LuASM
VS Code extension package to `%LOCALAPPDATA%\Programs\LuASM`.

## Installation

Run `LuASM-Setup.exe`. The two optional checkboxes let setup:

- Set `LUASM_HOME` and add the compiler's `bin` directory to your user PATH.
- Install the bundled extension into a locally installed VS Code.

If `LUASM_HOME` already points somewhere else, setup keeps the existing value
and tells you. Restart terminals and reload VS Code after installation. If VS
Code is not installed, the extension package is available at
`%LOCALAPPDATA%\Programs\LuASM\extension\luasm.vsix`; install it later via
**Extensions → ... → Install from VSIX**.

The installer includes LLVM/Clang, but Windows application builds also need the
Microsoft C++ build tools and Windows SDK. Install Microsoft's **Desktop
development with C++** workload with the x64/x86 build tools and a Windows SDK.
The installer tests a real compilation and provides a link if the Microsoft tools
are not ready. Microsoft's tools and VS Code are separate downloads; the C++ tools
may require administrator permissions. Setup itself requires .NET Framework 4.5
or later. The alpha installer is unsigned.

## Verify and run

Open a new terminal and run `luasm --version`. See [getting started](getting-started.md)
for your first program and troubleshooting. If LuASM is not on PATH, run:

```powershell
& "$env:LOCALAPPDATA\Programs\LuASM\bin\luasm.exe" --version
```

## Removal

Use Windows **Installed apps** or run `Uninstall.exe` in the LuASM installation
folder. Close LuASM programs first. The uninstaller removes recorded installation
files and environment entries added by setup. It leaves other files, user projects,
and the VS Code extension; remove the extension separately in VS Code if desired.
Older alpha setup versions did not record environment ownership, so their existing
PATH entries may need to be removed manually. Restart terminals afterwards.
