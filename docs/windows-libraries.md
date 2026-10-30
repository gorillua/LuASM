# Windows application libraries

For modern retained components and rendering see [modern UI](modern-ui.md).
For direct Windows API declarations without per-function wrappers see
[DLL imports and UTF-16 interop](dll-imports.md).

The public modules are written in LuASM. A small C++ backend in the runtime
owns native handles, Unicode conversion, the window procedure and COM objects.
Applications use typed structs and events rather than Win32 message numbers.
Requires hosted Windows x86-64, the Windows SDK at build time, and Windows 10
1703+ or Windows 11 at run time. The per-monitor-v2 DPI mode needs 1703;
[Microsoft documents the underlying DPI APIs](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-adjustwindowrectexfordpi).
SDK libraries link automatically from the installed runtime archive.

All these modules require `--!low`, because their API uses structs and explicit
ownership. `requireTarget("windows-x86_64")` rejects incompatible targets with
LSA1103, including imports checked under `--freestanding`. The builtin also
accepts `linux-x86_64` and `x86_64-elf`; it takes a literal and emits no code.
Place target requirements in a function, since freestanding top-level executable
statements are prohibited. Every function is semantically checked, even unused.

## First window

Save as `main.lasm`:

```lua
--!low
import "windows/ui"
import "windows/dialogs"

function eventHandler(context: ptr<void>, event: ptr<ui.Event>): void
    local target = cast<ptr<ui.Window>>(context)
    if event.kind == ui.closeRequested then
        ui.close(load(target))
    elseif event.kind == ui.activated then
        dialogs.info(load(target), "LuASM", "Ciao dal mio programma!")
    end
end

local app = ui.application()
assert(app.handle ~= 0)
local main = ui.window(app, "La mia app", 640, 400)
assert(main.handle ~= 0)
local action = ui.button(main, 1, "Premimi", ui.rect(20, 20, 140, 36))
assert(action.handle ~= 0)
assert(ui.show(main))
assert(ui.run(app, eventHandler, cast<ptr<void>>(addressOf(main))))
assert(ui.destroy(addressOf(app)))
```

```powershell
luasm build main.lasm --subsystem windows -O2 -o MyApp.exe
```

`--subsystem windows` produces a graphical executable without a console, using
the ordinary LuASM main and argument initialization. `--subsystem console`
explicitly selects a console. The option only applies to hosted Windows
executable builds. The common-controls manifest is embedded in the executable,
so copying it does not lose visual styles or control registration.
Distributing the executable still requires its Microsoft C/C++ runtime DLLs.

## Ownership, threading and failures

Create one `Application` on one UI thread; keep all library UI calls on that
thread. This release supports one UI thread per process and one live application
on that thread. It initializes a COM STA and balances that initialization on
`destroy`; an incompatible existing COM apartment fails app creation. Do not
call `destroy` from an event callback; close windows and destroy after `run`.
`Application` owns all windows and controls. `Window`/`Control` copies are
borrowed handles. Destroying a window also destroys its controls; destroying an
application releases all remaining windows, controls, fonts and COM ownership.
Handle generations reject stale handles even after a slot is reused.

Creation returns a zero `handle` on failure; most operations return boolean.
`ui.lastError()` returns the last native Win32/HRESULT error on this thread;
read it immediately after a failed native operation. Successful native operations
clear it. It does not describe `std/buffer` allocation failures or `files.read`
errors, whose boolean result is authoritative. No automatic exception unwinding
or finalizers exist: arrange cleanup on every return path.

UI text and paths use strict UTF-8. Embedded NUL/invalid UTF-8 is rejected, rather
than truncating. UI string setters accept at most 1 MiB of UTF-8 input; edit
controls accept up to 1 Mi UTF-16 units from interactive input. `getText` copies
UTF-8 into an initialized `std/buffer.Buffer`, replacing its length on success;
the buffer owns the copy. `buffer.view` borrows that storage.

Limits: 512 combined windows/controls per application, unique nonzero control
IDs per parent, client dimensions 320×220 through 10000×10000 logical pixels.
The backend has no closure capture, custom drawing, menu, docking, webview,
clipboard or worker-thread abstraction. These are native standard controls.

## Controls, layouts and events

| Module/API | Use |
| --- | --- |
| `ui.window`, `show`, `hide`, `setTitle`, `resize`, `clientSize`, `close` | Top-level windows |
| `button`, `label`, `textBox`, `listBox`, `progressBar` | Native controls; each takes parent, ID and `Rect` |
| `setText`, `getText`, `move`, `enable`, `focus` | Control state |
| `addItem`, `selectItem`, `selection` | List items; indices start at zero, -1 means no selection |
| `setProgress` | Percentage 0..100 |
| `startTimer`, `stopTimer` | Window-owned repeating timer, nonzero ID, 10..3600000 ms |
| `requestClose`, `click` | Queue a close request / activate a button |
| `layout.inset`, `row`, `column`, `take`, `remaining` | Consume rectangles with padding and gaps |

All coordinates are logical 96-DPI pixels. The backend rescales fonts and
controls on DPI changes. Recompute rectangles on `ui.resized`; `clientSize`
returns logical client dimensions. For example:

```lua
local size = ui.clientSize(main)
local column = layout.column(layout.inset(ui.rect(0, 0, size.width, size.height), 16), 12)
local toolbarArea = layout.take(addressOf(column), 34)
ui.move(editor, layout.remaining(addressOf(column)))
```

`ui.run(app, handler, context)` blocks and dispatches a borrowed `ptr<ui.Event>`
to `fn<void, ptr<void>, ptr<ui.Event>>`. Keep context alive until it returns.
The event pointer is valid only during the callback. `nextEvent(app, out, false)`
polls instead; `true` waits. False with `lastError()==0` means no pending event
(poll) or no remaining windows / message-loop quit (wait). `run` returns false
on native failure. A callback should not recursively run another event loop.

| Event kind | Fields |
| --- | --- |
| `closeRequested` | `window`: handler decides whether to call `close` or keep it open |
| `closed` | `window`: already destroyed, handle is now stale |
| `activated` | `window`, `control`, `code`=ID; button click or list selection change |
| `resized` | `window`, `width`, `height`; zero dimensions are possible while minimized |
| `textChanged` | `window`, `control`, `code`=ID; user edits or `setText` |
| `timerElapsed` | `window`, `code`=timer ID |
| `eventsDropped` | `code`=number of dropped events |

Use `isWindow`/`isControl` to compare event sources. `x`/`y` are reserved and
currently zero. Resize/text-change events coalesce; the bounded queue holds 256
events and reports overflow. Close requests use a separate per-window flag and
are not discarded when the queue is full. Pending events can refer to a window
already closed by your callback; handle validation still rejects further access.

## Dialogs, files and utilities

`dialogs.info`/`error` show message boxes and return boolean. `confirm` returns
`accepted` (Yes), `cancelled` (No, the default), or `failed`. File dialogs use
[Microsoft's Common Item Dialog](https://learn.microsoft.com/en-us/windows/win32/shell/common-file-dialog)
and return the same three states. `openFile(window, pathBuffer)` and
`saveFile(window, pathBuffer)` replace the initialized path buffer only on
acceptance; cancellation preserves its bytes and length. They reserve 128 KiB
before showing the dialog, enough for a maximum Windows path encoded as UTF-8.
Filters are Text files / All files, with `.txt` as save default. Save confirms
overwriting but does not itself write a file. Paths remain subject to host
Windows filesystem and long-path settings.

`files.read(path, buffer)` appends bytes and restores the original length on
failure. `files.writeAtomic(path, text)` writes binary data to a uniquely created
temporary in the destination directory, flushes/closes it, then replaces the
target. Failures before replacement leave the existing target intact and clean
up the temporary; this does not promise power-loss durability, preservation of
the old file's ACLs/streams, or multi-file transactions. `exists` checks either a
file or directory; `createDirectory` accepts an existing directory but refuses
an existing file. It creates one directory level, not missing parents.

`system.stopwatch()` returns a high-resolution clock; `elapsedSeconds(clock)`
returns `f64`. `sleep(ms)` blocks the calling thread, so avoid it in UI handlers.
`executablePath`, `currentDirectory` and `temporaryDirectory` replace an
initialized buffer on success.

The [Notes example](../examples/windows/notes.lasm) combines responsive layout,
text editing, open/save dialogs, atomic saves and unsaved-change confirmation.
Regression tests execute native hidden windows at O0/O2, test modal cancellation
and message-box responses, and inspect both PE subsystems and embedded manifest.
File-dialog acceptance and multi-monitor DPI transitions still require manual
testing; cancellation and the coordinate calculations are exercised automatically.
