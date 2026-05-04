# Hunchentoot: usocket → iolib Refactor Analysis

## Overview

Hunchentoot currently uses **usocket** for all socket operations on non-LispWorks
implementations. This document explores replacing usocket with **iolib**, a
modern, CFFI-based networking library with Gray stream integration, an I/O
multiplexer (epoll/kqueue), and a cleaner API.

## Why iolib?

- **Sockets are Gray streams**: no separate `socket-stream` accessor; read/write
  directly on the socket object
- **I/O multiplexer**: `iomux` provides epoll/kqueue/poll, enabling non-blocking
  and event-driven patterns without external deps
- **Modern API**: CLOS-based, keyword arguments, address objects instead of raw
  vectors
- **Better timeout support**: built into socket operations, no per-implementation
  hacks
- **Active maintenance**: sionescu/iolib on GitHub

## Portability Impact

This is the main concern.

| Implementation | usocket | iolib |
| --- | --- | --- |
| SBCL (Linux/macOS/BSD) | ✅ | ✅ |
| CCL (Linux/macOS) | ✅ | ✅ |
| ECL | ✅ | ⚠️ partial |
| CLISP | ✅ | ❌ |
| ABCL | ✅ | ❌ |
| Clasp | ✅ | ❌ |
| Mezzano | ✅ | ❌ |
| LispWorks | own sockets | ❌ |
| Windows (any impl) | ✅ | ❌ |

**iolib is POSIX-only** (uses libfixposix/CFFI). A full replacement would drop
support for CLISP, ABCL, Clasp, Mezzano, and all of Windows.

### Recommended strategy: pluggable backend

Rather than replacing usocket entirely, make the socket layer pluggable:

```text
hunchentoot              (core, backend-agnostic)
├── hunchentoot/usocket   (current behaviour, portable)
├── hunchentoot/iolib     (modern, POSIX-only)
└── LispWorks             (existing #+lispworks paths)
```

Users choose at system load time. Default remains usocket for maximum portability.

## API Mapping

### Socket creation and lifecycle

| usocket | iolib |
| --- | --- |
| `(usocket:socket-listen host port :reuseaddress t :backlog n :element-type '(unsigned-byte 8))` | `(iolib:make-socket :connect :passive :local-host host :local-port port :backlog n :reuse-address t)` |
| `(usocket:socket-accept listener)` | `(iolib:accept-connection listener)` |
| `(usocket:wait-for-input listener :ready-only t)` | `(iolib:accept-connection listener :wait t)` or `(iomux:wait-until-fd-ready fd :input)` |
| `(usocket:socket-connect host port)` | `(iolib:make-socket :connect :active :remote-host host :remote-port port)` |
| `(usocket:socket-close socket)` | `(close socket)` |
| `(usocket:with-server-socket (var socket) &body)` | `(iolib:with-open-socket (var ...) &body)` or `unwind-protect` + `close` |

### Address and stream access

| usocket | iolib |
| --- | --- |
| `(usocket:socket-stream socket)` | socket IS the stream; use directly |
| `(usocket:get-peer-name socket)` → `(values addr port)` | `(values (iolib:remote-host socket) (iolib:remote-port socket))` |
| `(usocket:get-local-name socket)` → `(values addr port)` | `(values (iolib:local-host socket) (iolib:local-port socket))` |
| `(usocket:get-peer-address socket)` | `(iolib:remote-host socket)` |
| `(usocket:get-peer-port socket)` | `(iolib:remote-port socket)` |
| `(usocket:get-local-port socket)` | `(iolib:local-port socket)` |
| `(usocket:host-to-hostname addr)` | `(iolib:address-to-string addr)` |
| `(usocket:vector-quad-to-dotted-quad addr)` | `(iolib:address-to-string addr)` |
| `usocket:*wildcard-host*` | `iolib:+ipv4-unspecified+` or `nil` |
| `(usocket:socket usocket)` (raw fd) | `(iolib:socket-os-fd socket)` |

### Conditions

| usocket | iolib |
| --- | --- |
| `usocket:connection-aborted-error` | `iolib:socket-connection-aborted-error` |
| `usocket:timeout-error` | `isys:ewouldblock` / `iomux:poll-timeout` |
| `(usocket:with-mapped-conditions () &body)` | Not needed; iolib signals POSIX conditions directly |

### Timeouts

iolib supports socket-level timeouts directly via socket options, eliminating
most of the per-implementation code in `set-timeouts.lisp`:

```lisp
;; iolib approach
(setf (iolib:socket-option socket :receive-timeout) read-timeout)
(setf (iolib:socket-option socket :send-timeout) write-timeout)
```

This replaces the entire `set-timeouts.lisp` file (currently 90 lines of
`#+sbcl #+ccl #+ecl #+clisp #+abcl #+mezzano` reader conditionals).

## Files Affected

| File | usocket refs | Scope of change |
| --- | --- | --- |
| `acceptor.lisp` | 13 | listen, accept loop, shutdown wake, close |
| `set-timeouts.lisp` | 12 | Entirely replaced by iolib socket options |
| `compat.lisp` | 5 | get-peer/local-address, socket-stream |
| `taskmaster.lisp` | 5 | client-as-string, peer address logging |
| `util.lisp` | 3 | with-mapped-conditions macro |
| `headers.lisp` | 1 | timeout-error in condition handler |
| `ssl.lisp` | 1 | Comment only |
| `hunchentoot.asd` | 1 | Dependency declaration |
| **Total** | **~40** | **7 source files + .asd** |

## Refactor Plan

### Phase 1: Abstract the socket interface (this branch)

Define a thin internal protocol that both backends implement:

```lisp
;; package: hunchentoot.sockets
(defgeneric listen-socket (backend host port &key backlog reuse-address))
(defgeneric accept-socket (backend listener))
(defgeneric wait-for-connection (backend listener))
(defgeneric close-socket (backend socket))
(defgeneric connect-socket (backend host port))
(defgeneric socket-stream-of (backend socket))
(defgeneric peer-address (backend socket))   ; → (values host-string port)
(defgeneric local-address (backend socket))  ; → (values host-string port)
(defgeneric set-socket-timeouts (backend socket read-timeout write-timeout))
```

### Phase 2: Implement usocket backend

Move existing usocket code behind the protocol. This is a pure refactor with no
behaviour change.

### Phase 3: Implement iolib backend

Write the iolib implementation of the same protocol. Key simplifications:

- `socket-stream-of` is identity (sockets are streams)
- `set-socket-timeouts` is ~4 lines (socket options)
- `with-mapped-conditions` disappears
- `set-timeouts.lisp` is not loaded

### Phase 4: Wire up system definition

```lisp
;; hunchentoot.asd
(defsystem "hunchentoot"
  :depends-on (... #-lispworks "hunchentoot/backend-usocket")
  ...)

(defsystem "hunchentoot/backend-usocket"
  :depends-on ("usocket") ...)

(defsystem "hunchentoot/backend-iolib"
  :depends-on ("iolib") ...)
```

## Effort Estimate

- **Phase 1** (abstraction layer): ~2 hours. Define protocol, no behaviour change
- **Phase 2** (usocket backend): ~3 hours. Mechanical extraction
- **Phase 3** (iolib backend): ~3 hours. New implementation, simpler code
- **Phase 4** (system wiring): ~1 hour. ASDF changes, feature flag

**Total: ~1-2 days of focused work.**

The risk is low because Phase 2 is a pure refactor of existing code, and iolib
is a well-tested library used by Woo and other production servers.

## Decision Points

1. **Pluggable vs full replacement?** Pluggable recommended, preserves portability.
2. **Default backend?** usocket, backward compatible.
3. **Feature flag mechanism?** ASDF selection at load time.
4. **Upstream PR or fork-only?** Start as fork; propose upstream once stable.
