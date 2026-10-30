# Modern UI for LuASM

`windows/modern` is a retained component library for Windows x86-64. Its API is
LuASM; its drawing backend uses Direct2D and DirectWrite. It is independent of
XAML, Windows App SDK and WinUI deployment packages. The existing Windows SDK
and hosted C/C++ runtime requirements still apply. This is an initial component
set, not WinUI feature parity or a demonstrated speed advantage over WinUI.

## Quick start

```lua
--!low
import "windows/ui"
import "windows/modern"

function events(context: ptr<void>, event: ptr<ui.Event>): void
    local window = cast<ptr<ui.Window>>(context)
    if event.kind == ui.closeRequested then ui.close(load(window)) end
end

local app = ui.application()
assert(app.handle ~= 0)
local window = ui.window(app, "Modern app", 800, 600)
assert(window.handle ~= 0)
local view = modern.create(window, modern.systemTheme, 128)
assert(view.handle ~= 0)
local root = modern.root(view)
local spacing = modern.style()
spacing.padding = 24.0
modern.setStyle(root, spacing)
local card = modern.card(root, modern.cardStyle())
modern.text(card, "Hello, LuASM", modern.heading())
modern.text(card, "A native Windows interface.", modern.caption())
local action = modern.button(card, 1, "Continue", modern.style())
assert(action.handle ~= 0)
assert(ui.show(window))
assert(ui.run(app, events, cast<ptr<void>>(addressOf(window))))
assert(ui.destroy(addressOf(app)))
```

```powershell
luasm build main.lasm --subsystem windows -O2 -o ModernApp.exe
```

The complete [Studio example](../examples/windows/modern_studio.lasm) demonstrates
themes, navigation, metrics cards, a native editor, toggles and progress. It is a
component preview; applying settings updates the preview without changing files.
The sidebar hides below 800 logical pixels, while the flexible content expands.

```powershell
build/luasm.exe build examples/windows/modern_studio.lasm --subsystem windows -O2 -o build/modern-studio.exe
build/luasm.exe run examples/windows/modern_studio.lasm -- --snapshot build/modern-studio.png
```

## Ownership and capacity

Create one `View` per `ui.Window`, on the same UI thread. The view owns the
retained tree and all renderer/font/editor resources. The existing `ui.run`
message loop processes it. Destroying the parent window/application releases
the view automatically; `modern.destroy(ptr<View>)` can explicitly remove the
view while leaving its window open. `Element` and copied `View` values are
borrowed generation-checked handles. Removing a container removes its descendants
and invalidates their handles. Closed/replaced views reject old handles.

`create(window, themeMode, capacity)` reserves a fixed contiguous node pool of
2..8192 nodes, including its root. No automatic tree growth or GC occurs.
Creation fails with a zero handle on invalid settings, capacity exhaustion or
native failure. Tree depth is limited to 64, and at most 32 views can be live.
Interactive IDs must be nonzero and unique within a view. Check returned handles
and boolean results; `lastError` shares `ui.lastError` for native failures.

## Components and styles

| Function | Component |
| --- | --- |
| `column(parent, style)`, `row(parent, style)` | Automatic vertical/horizontal flow |
| `card(parent, style)` | Vertical container with rounded surface and border |
| `text(parent, contents, style)` | DirectWrite text with Unicode fallback/wrapping |
| `button(parent, id, contents, style)` | Clickable/focusable button |
| `toggle(parent, id, contents, checked, style)` | Label and on/off switch |
| `progress(parent, value, style)` | Determinate progress, value 0..1 |
| `textBox(parent, id, contents, style)` | Native single-line Windows editor |
| `divider(parent)` | One-pixel divider |

`style()` supplies defaults; `sized(width, height)`, `heading()`, `caption()` and
`cardStyle()` return reusable presets. Fields use logical 96-DPI pixels:

| Style field | Meaning |
| --- | --- |
| `width` | -1 fills the cross-axis / shares a row; nonnegative is fixed |
| `height` | 0 measures content; -1 shares available height; positive is fixed |
| `weight` | Positive share of flexible main-axis space |
| `padding`, `gap` | Container inner padding and spacing between visible children |
| `radius` | Corner radius for cards/buttons/progress/editor chrome |
| `fontSize` | Text size in DIP, 6..96 |
| `flags` | Bitwise combination of `accent`, `muted`, `semibold`, `noWrap` |

Rows divide width among flexible children after fixed widths and gaps. Columns
measure automatic heights and share remaining space among flexible-height
children. Their measured content provides a minimum height. Widths/heights are
bounded at 100000 DIP, padding/gap at 128, radius at 64, weight at 1000; NaNs and
invalid flags fail. Text wraps unless `noWrap` is set; buttons use one text line.
Containers clip overflowing descendants. Root overflow supports vertical mouse
wheel scrolling; there are no nested scroll views or horizontal scrolling yet.

`setStyle`, `setVertical`, `setVisible` and `remove` update the tree. Visibility
removes a subtree from layout; disabling preserves its layout. `bounds(element)`
reports its logical rectangle after view scrolling. A failed bounds query returns
zero fields and sets `lastError`; inspect that error when zero is ambiguous.
Parent-window resizing and DPI changes automatically remeasure/reposition the tree.
Use `setVertical`/`setVisible` in resize handlers for explicit responsive breakpoints.

## State, input and accessibility

Use `setText`/`getText`, `setValue`/`value`/`checked`, `setEnabled`, `focus` and
`activate` to manipulate components. `getText` copies UTF-8 into an initialized
`std/buffer.Buffer`. Inputs reject invalid UTF-8 and embedded NUL; setter inputs
are limited to 1 MiB. `isSource(event, element)` identifies a component in the
existing `ui.Event` stream. Buttons and toggles emit `ui.activated` with their ID
in `code`; toggles change value before dispatch. Editors emit `ui.textChanged`.
Programmatic value changes do not emit activation; `activate` explicitly does.

Mouse hover/press, Tab/Shift-Tab focus and Enter/Space activation are implemented.
Editors retain native IME, selection, clipboard and system accessibility.
The drawing surface supplies basic MSAA names, roles, values, states, geometry,
focus and default actions; stale accessibility objects disconnect on destruction.
Its MSAA child list is flat and reordered on structural changes. Full UIA control
patterns, complex text accessibility and assistive-technology certification are
not claimed. Multi-monitor DPI, touch, IME and screen-reader interactions still
need broader manual testing.

Themes are `systemTheme`, `lightTheme` or `darkTheme`; `setTheme` switches modes.
`lightPalette`/`darkPalette` return editable tokens. `setPalette(view, colors,
darkTitle)` applies a custom RGB palette and title-bar mode. System high-contrast
colors override all palettes. The system theme follows Windows app preferences.
The backend reuses a solid brush rather than creating brushes for each component.

## Rendering behavior and measurements

The library keeps components, fonts and text layouts alive between events.
Decorative components, buttons and toggles share one canvas HWND; only text
editors need extra HWNDs. Direct2D tries a hardware target and falls back to
software if creation fails. Dirty rectangles clip drawing, and monotonic flow
containers stop traversing siblings beyond the dirty area. Cached text layouts
are reused when text, width and typography remain unchanged. Layout runs lazily
on structural changes; hover/progress/toggle changes invalidate only their bounds.
Identical setters are no-ops. The render target retains previously drawn content.
This uses Microsoft's documented [resource reuse guidance](https://learn.microsoft.com/en-us/windows/win32/learnwin32/drawing-with-direct2d)
and [retained presentation option](https://learn.microsoft.com/en-us/windows/win32/api/d2d1/ne-d2d1-d2d1_present_options).

`beginUpdate(view)`/`endUpdate(view)` nest and coalesce structural invalidation
and accessibility reorder notifications. Balance all batches before rendering.
`flush(view)` processes pending drawing immediately, useful for deterministic
tests; ordinary applications can leave painting to `ui.run`. Idle views have no
render timer or continuous frame loop. OS exposure/DPI messages can still repaint.
Device loss releases target resources for rebuilding on the next invalidation;
a persistently failing driver does not cause an automatic retry loop.

`statistics(view)` returns cumulative node/window, paint/layout/text-layout,
visited/drawn-node, dirty-pixel-area and resource-build counters. `renderer` is 1
for a successfully created hardware target or 2 for software fallback; zero means
no target built yet. `renderError` records the last paint HRESULT.

In the Windows O0/O2 regression, a **1002-node tree uses one canvas**. Updating
its first progress bar visits/draws **3 nodes**, including the adjacent row
within the antialias guard, and creates **0 new text layouts**, **0 layout
passes** and **0 renderer resources**. The tests also verify no-op/idle behavior,
Unicode, input, state, resizing, MSAA and lifetime cleanup. These are work counters
for that scene, not frame-time/memory benchmarks or a comparison against WinUI.

All nodes are measured initially; this is not list virtualization. Work can still
scale with tree size during layout, hit-testing and scrolling past earlier rows.
The first release does not include animations, acrylic/Mica, images/icons, rich
text editors, menus, drag/drop or data binding. These boundaries keep the current
component set concrete and testable rather than implying full WinUI coverage.

`snapshot(view, path)` explicitly exports a PNG through the same draw traversal
using an offscreen software target, including editor text but excluding its
caret/selection. It allocates export resources and does not represent a hardware
screen capture or a steady-frame benchmark. The path is written directly; do not
use a valuable existing file as an export target.
