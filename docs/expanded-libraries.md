# Additional application libraries

All examples use `--!low` unless stated otherwise. Libraries are installed under
the LuASM library search path; no files need to be copied into your project.

## Math

```lua
--!low
import "std/math"
print(math.sqrt(81.0), math.sin(math.pi / 2.0))
print(math.clamp(12.0, 0.0, 10.0), math.lerp(2.0, 6.0, 0.5))
```

`std/math` provides `pi`, `tau`, `e`, `sin`, `cos`, `tan`, `asin`, `acos`,
`atan2`, `sqrt`, `pow`, `exp`, `log`, `floor`, `ceil`, `round`, `isFinite`,
`isNaN`, `abs`, `min`, `max`, `clamp`, `lerp`, `radians` and `degrees`. Angles
use radians. Operations use the hosted platform math library; domain errors and
special values follow its floating-point behavior. `clamp` requires low <= high;
`round` rounds halfway cases away from zero. This module requires a hosted build.

## Text, encoding and data

All buffers must first be initialized with an allocator and destroyed by their
owner. Encoding and path functions append to the output, preserve the original
contents on failure, and accept input that borrows from the same buffer.

| Module | Functions |
| --- | --- |
| `std/text` | `startsWith`, `endsWith`, `find`, `contains`, `trim`, `stripPrefix`, `stripSuffix`, `equalsIgnoreCaseASCII`. Slices borrow the source; case comparison is ASCII only. Works in ordinary mode too. |
| `std/strings` | `append`, `line`, `appendSigned`, `appendUnsigned` for building text in a buffer, including full-width integer extrema. |
| `std/bytes` | `readU16LE`, `readU32LE`, `readU32BE`, `appendU16LE`, `appendU32LE`, `appendU32BE`. Reads check length and preserve output on failure. Callers remain responsible for the actual allocation behind a pointer. |
| `std/base64` | `encode`, `decode`; strict RFC 4648 alphabet, required padding, and zero pad bits. Binary strings including NUL are supported. |
| `std/url` | `encodeComponent`, `decodeComponent`; percent-encode raw UTF-8 bytes. Decoding is strict, can produce NUL, and leaves `+` unchanged. |
| `std/path` | `filename`, `extension`, `isAbsolute`, `join`. Handles slash and backslash lexically. It does not normalize `..`, access the filesystem, or enforce a security boundary. |

These six modules can also be used freestanding with a suitable allocator; their
integer-only use is exercised inside QEMU. `std/base64` and `std/url` don't
validate decoded bytes as UTF-8. Buffer views remain borrowed and can become
invalid when their backing buffer is mutated, resized, or destroyed.

`std/random` offers `create(seed)`, `nextU64`, `below(bound)` and `unit()` using
explicit SplitMix64 state. Sequences are repeatable; `below` requires a nonzero
bound and avoids modulo bias. **This is not a cryptographic random generator.**
The current module also contains floating-point code and is therefore hosted-only.

## HTTP and HTTPS on Windows

```lua
--!low
import "std/http"
import "std/host"
import "std/buffer"

local body = zero<buffer.Buffer>()
buffer.initialize(addressOf(body), host.allocator())
local response = http.get("https://example.com/", addressOf(body))
if response.ok then
    print("Status:", response.status)
    print(buffer.view(addressOf(body)))
else
    print("HTTP error:", response.kind, response.error)
end
buffer.destroy(addressOf(body))
```

`std/http` and `windows/http` provide `get`, `post`, `request`, and `successful`.
`post(url, contentType, body, output)` sends the exact bytes of `body`. For custom
methods, headers, limits and timeouts use
`request(method, url, headers, body, timeoutMs, maxBody, output)`. Headers are
UTF-8 lines separated by CRLF. `post` rejects CR/LF/NUL in its content type.

`Response.ok` means the transfer completed, even for HTTP 404 or 500. Check
`status` or `successful` for a 2xx response. `kind` is zero on success, or one of
`invalidInput`, `networkError`, `responseTooLarge`, `outOfMemory`; `error` holds a
Windows error code when available. The output buffer is appended to on success
and left unchanged on failure. Defaults are an 8 MiB body limit and WinHTTP
timeout settings of 15 seconds; the maximum configurable body is 64 MiB. URL
credentials are rejected, URL fragments are omitted, redirects are returned as
3xx rather than followed, and HTTPS certificate checks use Windows defaults.

The client is synchronous and uses WinHTTP without a proxy. **Do not call it on
your GUI thread** if responsiveness matters. Timeout values configure WinHTTP's
operation timeouts; they are not a guaranteed overall deadline. In local testing,
a stalled response could exceed a short configured timeout. Redirects, proxies,
response headers, cancellation, streaming output, decompression and non-Windows
backends are not yet implemented. Local HTTP GET/POST, binary data, chunked
responses, status handling and size limits are covered by tests. HTTPS certificate
validation is delegated to WinHTTP; a live HTTPS server was not tested here.

## Simpler Windows apps and calendar time

`windows/app.create(title, width, height)` creates an `App` holding a native
application and primary window. Check `app.valid`, call `app.show`, then
`app.run` or `app.runWith` and finally `app.destroy(addressOf(owner))`. The loop
automatically closes the primary window on a close request; use `windows/ui`
directly for close vetoes or multi-window lifecycle management. The handler passed
to `runWith` receives the same borrowed event and context as `ui.run`.

`windows/clock.localNow()` and `clock.utcNow()` return a `DateTime` value with
year, month, day, dayOfWeek (Sunday = 0), hour, minute, second and milliseconds.
Use this calendar time for a clock display; use the `time()` builtin for elapsed
durations. See `examples/windows/simple_window.lasm`, `http_client.lasm` and
`examples/library_tour.lasm` for complete examples.

Windows-specific modules require hosted Windows x86-64. Native Linux has not
been executed in the current test environment.
