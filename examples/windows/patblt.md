# PatBlt window example

`patblt.lasm` is a complete Windows x64 application written in LuASM. It declares
its Windows API imports, registers a window class with a LuASM callback, creates
a resizable window and processes Windows messages until the window closes.

Run from the repository root:

```powershell
./build/install/bin/luasm.exe run examples/windows/patblt.lasm
```

Build a GUI executable without a console window:

```powershell
./build/install/bin/luasm.exe build examples/windows/patblt.lasm -O2 --subsystem windows -o build/patblt.exe
./build/patblt.exe
```

In VS Code use **LuASM: Run Current File** with `patblt.lasm` open. The main Run
button may follow another entry file configured for the project.

`fillArea` selects a solid brush, calls `PatBlt` with `PATCOPY`, restores the
previous brush and releases the brush it owns. Change the last argument to
`fillArea` to change a rectangle's color. Windows `COLORREF` values use
`0x00BBGGRR`; for example, `0x000000FF` is red.

The window repaints through `WM_PAINT`, with paired `BeginPaint`/`EndPaint` calls.
Resizing requests a new paint. Painting uses the window's own device context.
The script does not draw on the desktop or another application's window.

The optional smoke check uses a hidden window, exercises the native callback,
paints an owned memory bitmap and checks a pixel before releasing resources:

```powershell
./build/install/bin/luasm.exe run examples/windows/patblt.lasm -- --smoke
```

Native layouts are checked for the supported Windows x64 target. Strings remain
owned until the window is destroyed and its class is unregistered.

API references: [PatBlt](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-patblt),
[BeginPaint](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-beginpaint),
[WNDCLASSEXW](https://learn.microsoft.com/en-us/windows/win32/api/winuser/ns-winuser-wndclassexw).
