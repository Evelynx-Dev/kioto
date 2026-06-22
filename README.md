# Kioto — Mire Standard Library

**Version:** 3.11.10 — **Language:** Mire (Avenys) — **License:** GPL v3.0

Kioto is the standard library for the Mire programming language. It provides
core data structures, operating system abstractions, networking, and browser
bridge primitives. Kioto is compiled by Avenys and linked against Mire's
Platform Abstraction Layer (PAL).

```
User code (.mire)
     │
     ▼
┌─────────────────────┐
│  kioto (this repo)  │  core/ + ext/  —  pure Mire wrappers
└─────────┬───────────┘
          │ extern fn pal_*  /  extern fn rt_*
          ▼
┌─────────────────────┐
│  Mire PAL + RT (C)  │  pal/linux/ + runtime/  —  OS syscalls + managed types
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Operating System    │  Linux (POSIX) | WASM (WASI + Emscripten)
└─────────────────────┘
```

---

## Quick Start

```mire
load kioto

fn main: () {
    # HTTP GET
    set res = net::http_get("https://httpbin.org/get")
    use dasu(res)
}
```

Compile with:

```bash
mire run main.mire
# or via owl:
owl run
```

---

## Module Reference

### Core modules (`kioto::core/`)

| Module    | Path              | Description |
|-----------|-------------------|-------------|
| `strings` | `core/strings`    | upper/lower, split/join, replace, trim, pad, substr, starts_with, ends_with, contains, index_of, len, from_i64, to_i64 |
| `lists`   | `core/lists`      | push/pop/get, slice, concat, sort, reverse, unique, len, create, join |
| `dicts`   | `core/dicts`      | get/set/keys/values, has, remove, merge, len, to_json |
| `time`    | `core/time`       | now_ms, now_ns, sleep_ms, sleep_ns, mark, elapsed, format |
| `fs`      | `core/fs`         | read, write, append, exists, is_dir, is_file, size, copy, move, delete, mkdir, rmdir, list, join, dir, name, ext |
| `env`     | `core/env`        | get, set, all, cwd, chdir, args |
| `proc`    | `core/proc`       | run, exec, shell, spawn, wait, kill, exit, exists |
| `async`   | `core/async`      | spawn, join, ready, failed, sleep |
| `mem`     | `core/mem`        | used, total, free, available, percent, process_bytes, snapshot, format |
| `cpu`     | `core/cpu`        | time_ns, time_ms, mark, elapsed, count, freq_mhz, loadavg |
| `gpu`     | `core/gpu`        | snapshot (GPU info) |
| `math`    | `core/math`       | sqrt, pow, abs, floor, ceil, round, sin, cos, tan, log, log2, log10, random |
| `term`    | `core/term`       | style, hr, clear |

### Networking (`kioto::core/net`)

`load kioto::net` — TCP + HTTP/HTTPS client.

| Function | Signature | Description |
|----------|-----------|-------------|
| `connect` | `(host :&str, port :i64) :i64` | Raw TCP connect, returns fd |
| `connect_timeout` | `(host :&str, port :i64, timeout_ms :i64) :i64` | TCP connect with timeout |
| `send` | `(fd :i64, data :&str) :bool` | Send raw data on TCP socket |
| `recv` | `(fd :i64, max_bytes :i64) :str` | Receive raw data from TCP socket |
| `recv_all` | `(fd :i64) :str` | Receive up to 64KiB |
| `close` | `(fd :i64)` | Close TCP/TLS socket |
| `poll` | `(fd :i64, timeout_ms :i64) :i64` | Poll socket for readability |
| `set_nonblock` | `(fd :i64, nonblock :i64)` | Set socket non-blocking |
| `resolve` | `(host :&str) :str` | Resolve hostname to IP |
| `http_get` | `(url :&str) :str` | HTTP or HTTPS GET request |
| `http_post` | `(url :&str, body :&str, content_type :&str) :str` | HTTP or HTTPS POST request |

Usage:

```mire
load kioto::net

fn main: () {
    # Plain HTTP
    set r1 = net::http_get("http://httpbin.org/get")
    use dasu(r1)

    # HTTPS (auto-detected, uses OpenSSL TLS)
    set r2 = net::http_get("https://httpbin.org/get")
    use dasu(r2)

    # POST with JSON body
    set r3 = net::http_post(
        "https://httpbin.org/post",
        "{\"key\": \"value\"}",
        "application/json"
    )
    use dasu(r3)
}
```

### WebSocket (`kioto::ext/ws`)

`load kioto::ws` — WebSocket client (ws:// and wss://).

| Function | Signature | Description |
|----------|-----------|-------------|
| `connect` | `(url :&str) :i64` | Connect to ws:// or wss:// endpoint, returns fd |
| `send_text` | `(fd :i64, message :&str)` | Send text frame |
| `recv` | `(fd :i64, max_bytes :i64) :str` | Receive one text frame |
| `recv_all` | `(fd :i64) :str` | Receive up to 64KiB |
| `close` | `(fd :i64)` | Send close frame and shutdown |

WSS (WebSocket over TLS) is auto-detected via `wss://` prefix and uses port 443 by default.
Plain WS uses `ws://` prefix with port 80. Custom ports: `ws://host:9877/path`.

Usage:

```mire
load kioto::ws

fn main: () {
    # Plain WebSocket
    set fd = ws::connect("ws://127.0.0.1:9877/chat")
    if fd < 0 {
        use dasu("Connection failed")
        return
    }
    ws::send_text(fd, "Hello!")
    set msg = ws::recv_all(fd)
    use dasu("Received: " + msg)
    ws::close(fd)

    # Secure WebSocket (WSS)
    set fd2 = ws::connect("wss://echo.websocket.org/")
    ws::send_text(fd2, "ping")
    set reply = ws::recv_all(fd2)
    ws::close(fd2)
}
```

### WASM / Browser Bridge (`kioto::ext/wasm`)

`load kioto::wasm` — Browser DOM and JavaScript interop (WASM targets only).

On native (non-WASM) targets, these functions are no-ops.

| Function | Signature | Description |
|----------|-----------|-------------|
| `eval` | `(js :&str) :str` | Execute arbitrary JS, return result |
| `fetch` | `(url :&str, method :&str, body :&str) :str` | Browser fetch() API |
| `fetch_get` | `(url :&str) :str` | Convenience: fetch(url, "GET", "") |
| `get_element_by_id` | `(id :&str)` | Select DOM element by id |
| `set_inner_html` | `(id :&str, html :&str)` | Set element.innerHTML |
| `create_element` | `(tag :&str)` | Create a new DOM element |
| `append_child` | `(parent_id :&str, child_id :&str)` | Append child to parent element |
| `set_attribute` | `(id :&str, attr :&str, value :&str)` | Set HTML attribute |
| `set_style` | `(id :&str, prop :&str, value :&str)` | Set CSS style property |
| `remove_element` | `(id :&str)` | Remove element from DOM |
| `add_event_listener` | `(id :&str, event :&str, handler :str)` | Bind JS event handler |
| `local_storage_get` | `(key :&str) :str` | Read from localStorage |
| `local_storage_set` | `(key :&str, value :&str)` | Write to localStorage |
| `console_log` | `(msg :&str)` | console.log() |
| `alert` | `(msg :&str)` | window.alert() |

### Extension modules (`kioto::ext/`)

| Module   | Path          | Description |
|----------|--------------|-------------|
| `types`  | `ext/types`   | type_of, is_i64, is_str, is_bool, is_list, is_dict |
| `maybe`  | `ext/maybe`   | Option type: Some(value), None |
| `result` | `ext/result`  | Result type: Ok(value), Err(error) |
| `tuple`  | `ext/tuple`   | 2- and 3-element tuples |
| `iter`   | `ext/iter`    | map, filter, reduce, for_each over lists |
| `json`   | `ext/json`    | parse, stringify, get, set (JSON manipulation) |
| `ws`     | `ext/ws`      | WebSocket client (ws://, wss://) |
| `wasm`   | `ext/wasm`    | Browser DOM + JS bridge |

---

## How to Import

### Using `load` in your code

```mire
# Load the entire standard library (all modules)
load kioto

# Load specific modules
load kioto::net
load kioto::ws
load kioto::wasm

# Use qualified names
set r = net::http_get("https://example.com")
set fd = ws::connect("ws://127.0.0.1:9877/")
```

### Using `use` to import specific names

```mire
use kioto::net::http_get
use kioto::ws::connect as ws_connect
use kioto::strings::from_i64

set r = http_get("https://example.com")
```

---

## Project Structure

```
kioto/
  mod.mire           # Root: loads all sub-modules
  owl.toml           # Package manifest: name, version, exports
  core/
    net/mod.mire     # TCP + HTTP/HTTPS client
    strings/         # String manipulation
    lists/           # List operations
    dicts/           # Dictionary operations
    fs/              # Filesystem
    env/             # Environment variables
    proc/            # Process management
    async/           # Async task spawning
    time/            # Time utilities
    mem/             # Memory stats
    cpu/             # CPU stats
    gpu/             # GPU info
    math/            # Math functions
    term/            # Terminal styling
  ext/
    ws/mod.mire      # WebSocket client
    wasm/mod.mire    # WASM browser bridge
    json/mod.mire    # JSON parsing
    types/           # Type introspection
    maybe/           # Option[T]
    result/          # Result[T,E]
    tuple/           # Tuples
    iter/            # Iterator combinators
```

---

## Build System Integration

Kioto is registered as a dependency in `owl.toml`:

```toml
[dependencies]
kioto = { path = "../kioto" }
```

The compiler (Avenys) resolves `load kioto` → `kioto/mod.mire`, then
`owl.toml` `[exports]` map module names (e.g. `net = "core/net"`) to
filesystem paths.

---

## Documentation

- [PAL & ABI Reference](https://github.com/mire-lang/mire/blob/main/docs/PAL-ABI.md) — How to add platform functions
- [Mire Libraries Guide](https://github.com/mire-lang/mire/blob/main/docs/LIBRARIES.md) — How `load`/`use`/`owl.toml`/`exports` work
- [Mire Language Syntax](https://github.com/mire-lang/mire/blob/main/SYNTAX.md)

---

## Changelog

See [CHANGELOG.md](./CHANGELOG.md)
