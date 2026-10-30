# Modules and exports

```lua
-- math.lasm
export function square(value: number): number
    return value * value
end

function helper(): number
    return 1
end
```

```lua
-- app.lasm
import "math"
import "math.lasm" as arithmetic
print(math.square(3), arithmetic.square(4))
```

Imports are relative to the importing source file. `.lasm` is appended when no
extension is present. The default namespace is the filename stem; `as` gives an
explicit identifier. An import may appear only at module scope. Dependencies
are canonicalized and loaded once, including transitively shared dependencies.

Only `export function`, `export struct` and `export global` declarations are visible through an
import namespace. Private names are distinct between modules. Public struct
names work in annotations, constructors and generic layout/pointer queries:

```lua
--!low
import "graphics/framebuffer" as graphics
function draw(display: ptr<graphics.Framebuffer>): void
    graphics.putPixel(display, 0, 0, 0x336699)
end
```

An ordinary root cannot import a low-mode module. A low-mode root can import
ordinary modules; each function retains its source module's numeric/mode rules.
Imported modules
contain declarations and imports only. Typed globals and `global const` are
supported, including `export global const`; executable module initialization
remains unsupported. Dependencies' global initializers resolve before the
importer's, while source order applies within each file. Ordinary top-level
`const` remains local to the hosted program body; use `global const` for a
constant shared with functions or imports. Locals and
parameters cannot shadow an import alias or a declaration in their own module.
Builtins and primitive type names cannot be redefined by global declarations.

`export extern function ...` can expose a C ABI declaration from a module;
ordinary `extern function ...` remains private to that module. Imported functions
and structs are analyzed together and lowered into one LLVM module. There is no
incremental module cache or separately compiled import interface yet. Root
exports have stable source names for object/library integration. Freestanding
root functions retain source names even without `export`, for compatibility
with ordinary standalone kernel/library object generation.

Imports first try their source-relative path, then explicit `--module-path`
directories, then installed/build library roots. For example `import "std/text"`
works after installation. Explicit `./` and `../` paths never use fallback roots.
Imported definitions retain private linker visibility; only the root unit's
exports form its public object interface. Identical extern declarations across
imports are deduplicated; incompatible ones report `LSA2116`. See
[separate compilation](native-applications.md#separate-compilation) for
`--no-entry` and `--object`.

Diagnostic families:

| Code | Meaning |
| --- | --- |
| `LSA3001` | missing or unreadable imported source |
| `LSA3002` | import cycle or import depth above 128 |
| `LSA3003` | invalid, duplicate or conflicting import alias |
| `LSA3004` | executable imported module or mixed source modes |
| `LSA3005` | unknown namespace or member not exported |
| `LSA2003` | duplicate declaration or conflicting local |

Diagnostics retain the originating file, source line and caret for imported code.
