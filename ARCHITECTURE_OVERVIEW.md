# Node.js Architecture Overview

> A comprehensive guide to understanding Node.js architecture for OSS contributors

## Table of Contents
- [What is Node.js?](#what-is-nodejs)
- [High-Level Architecture](#high-level-architecture)
- [Layered Architecture Pattern](#layered-architecture-pattern)
- [Core Components](#core-components)
- [Key Dependencies](#key-dependencies)
- [Repository Structure](#repository-structure)
- [Architecture Patterns](#architecture-patterns)

---

## What is Node.js?

**Node.js** is an open-source, cross-platform JavaScript runtime environment built on:
- **V8 JavaScript Engine** (from Google Chrome)
- **libuv** (asynchronous I/O library)
- **OpenSSL, nghttp2, zlib**, and other core libraries

**Key Characteristics:**
- **License**: MIT
- **Governance**: OpenJS Foundation with open governance model
- **Release Cycle**: New major version every 6 months (April & October)
- **LTS Support**: Even-numbered versions become LTS with 30 months total support
- **Team**: Technical Steering Committee (TSC) + 150+ Collaborators + 50+ Triagers

---

## High-Level Architecture

Node.js implements a **layered architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────┐
│         JavaScript User API Layer (lib/)                │
│  HTTP, Streams, FS, Crypto, Process, Events, etc.      │
├─────────────────────────────────────────────────────────┤
│         C++ Binding Layer (src/api/)                    │
│  N-API, Node.js Native API, V8 Bindings                │
├─────────────────────────────────────────────────────────┤
│         Core C++ Implementation (src/)                  │
│  Async Wrapping, Module System, Buffer, Handles, etc.  │
├─────────────────────────────────────────────────────────┤
│         External Libraries (deps/)                      │
│  V8, libuv, OpenSSL, nghttp2, llhttp, ICU, etc.       │
└─────────────────────────────────────────────────────────┘
```

---

## Layered Architecture Pattern

### 1. JavaScript API Layer (`lib/` directory)

**Purpose**: User-facing JavaScript APIs that developers interact with

**Key Modules**:
- **HTTP/Network**: `_http_*.js`, `net.js`, `dgram.js`, `tls.js`
- **File System**: `fs.js`
- **Streams**: `_stream_*.js` (readable, writable, duplex, transform)
- **Process Management**: `child_process.js`, `cluster.js`
- **Async Utilities**: `async_hooks.js`, `events.js`, `timers.js`
- **Cryptography**: `crypto.js`
- **Advanced**: `worker_threads.js`, `inspector.js`
- **Module System**: `module.js` (CommonJS), ES module support

**Example Flow**:
```javascript
// User code
const http = require('http');
server.listen(3000);

// → lib/_http_server.js (JavaScript)
// → src/tcp_wrap.cc (C++ binding)
// → deps/uv (libuv event loop)
```

### 2. C++ Binding Layer (`src/api/` directory)

**Purpose**: Bridge between JavaScript and C++ implementations

**Key Components**:
- `async_resource.cc` - AsyncResource class for tracking async operations
- `callback.cc` - Callback handling
- `environment.cc` - Environment initialization
- `encoding.cc` - Character encoding support
- `exceptions.cc` - Exception handling
- `utils.cc` - Utility functions

### 3. Core C++ Implementation (`src/` directory)

**Purpose**: Core functionality implemented in C++ for performance

**Major Components** (110+ C++ source files):

#### Infrastructure
- `node.cc` - Main entry point, initialization
- `env.h` / `env.cc` - Environment class (manages isolate, context, event loop)
- `node_main_instance.cc` - Main instance management
- `base_object.h` / `base_object.cc` - Base class for all native objects
- `async_wrap.h` / `async_wrap.cc` - Async operation tracking

#### Binding System
- `node_binding.cc` - Module binding registration
- `module_wrap.cc` - ES module wrapping
- `js_native_api_v8.cc` - N-API implementation
- `node_builtins.cc` - Built-in modules management

#### File System & I/O
- `node_file.cc` - File operations
- `fs_event_wrap.cc` - File system event monitoring
- `node_dir.cc` - Directory operations

#### Networking
- `tcp_wrap.cc` - TCP socket handling
- `udp_wrap.cc` - UDP socket handling
- `cares_wrap.cc` - DNS resolution
- `stream_base.h` - Base stream class

#### HTTP/TLS
- `node_http_parser.cc` - HTTP parsing
- `node_http2.cc` - HTTP/2 protocol support
- `src/crypto/` - OpenSSL wrapper

#### Advanced Features
- `node_worker.cc` - Worker threads
- `node_contextify.cc` - Context/Realm sandboxing
- `node_messaging.cc` - Message passing between workers

#### Diagnostics & Profiling
- `src/inspector/` - Debugging support (V8 inspector protocol)
- `node_report.cc` - Crash reporting
- `node_perf.cc` - Performance hooks

### 4. External Libraries Layer (`deps/` directory)

**Purpose**: Vendored third-party dependencies (539MB total)

---

## Core Components

### V8 JavaScript Engine

**Location**: `deps/v8/` (largest dependency)

**Purpose**:
- Compiles and executes JavaScript code
- Provides the JavaScript runtime environment
- Manages memory (garbage collection)

**Key Concepts**:
- **Isolates**: Separate JavaScript engine instances (one per thread)
- **Contexts**: Execution environments within an isolate
- **Handles**: References to V8 objects (Local vs Global)
- **Scopes**: Manage handle lifetimes

### libuv Event Loop

**Location**: `deps/uv/`

**Purpose**:
- Cross-platform asynchronous I/O
- Event loop implementation
- Thread pool for file system operations
- Platform abstraction layer

**Key Concepts**:
- **Handles**: Long-lived resources (sockets, timers, signals)
- **Requests**: One-time operations (writes, connections, DNS lookups)
- **Event Loop Phases**: Timers → I/O callbacks → Poll → Check → Close

**Event Loop Flow**:
```
   ┌───────────────────────────┐
┌─>│           timers          │ (setTimeout, setInterval)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │ (I/O callbacks deferred)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │ (internal only)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │ (retrieve new I/O events)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │ (setImmediate)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │ (socket.on('close'))
   └───────────────────────────┘
```

### Environment Class

**Location**: `src/env.h`, `src/env.cc`

**Purpose**: Central class managing per-isolate state

**Contains**:
- V8 Isolate and Context
- libuv event loop (`uv_loop_t`)
- Built-in module cache
- Async hooks state
- Performance counters
- Configuration options

**Lifecycle**: One Environment per isolate (main thread or worker thread)

---

## Key Dependencies

| Dependency | Purpose | Size | Location |
|-----------|---------|------|----------|
| **V8** | JavaScript engine | ~150MB | `deps/v8/` |
| **libuv** | Event loop & async I/O | ~10MB | `deps/uv/` |
| **OpenSSL** | Cryptography & TLS | ~50MB | `deps/openssl/` |
| **npm** | Package manager | ~100MB | `deps/npm/` |
| **llhttp** | HTTP parser | ~5MB | `deps/llhttp/` |
| **nghttp2** | HTTP/2 implementation | ~10MB | `deps/nghttp2/` |
| **ngtcp2** | QUIC protocol | ~5MB | `deps/ngtcp2/` |
| **cares** | DNS resolver | ~5MB | `deps/cares/` |
| **undici** | HTTP client | ~20MB | `deps/undici/` |
| **ICU** | Internationalization | ~30MB | `deps/icu-small/` |
| **zlib, brotli, zstd** | Compression | ~10MB | `deps/` |
| **acorn** | JavaScript parser | ~2MB | `deps/acorn/` |
| **sqlite** | Database support | ~5MB | `deps/sqlite/` |

**Total Dependencies**: 28+ libraries, 539MB

---

## Repository Structure

```
node/
├── src/                    # C++ core implementation (5.8MB, 110+ files)
│   ├── api/               # C++ binding layer
│   ├── crypto/            # Cryptography implementations
│   ├── inspector/         # Debugging support
│   ├── permission/        # Permission system
│   ├── node.cc            # Main entry point
│   ├── env.h/.cc          # Environment class
│   └── base_object.h/.cc  # Base native object class
│
├── lib/                    # JavaScript modules (4.2MB, 50+ files)
│   ├── _http_*.js         # HTTP implementation
│   ├── _stream_*.js       # Stream implementations
│   ├── fs.js              # File system API
│   ├── net.js             # TCP networking
│   ├── crypto.js          # Cryptography API
│   └── module.js          # CommonJS module system
│
├── deps/                   # External dependencies (539MB, 28+ libraries)
│   ├── v8/                # JavaScript engine
│   ├── uv/                # libuv event loop
│   ├── openssl/           # Cryptography
│   ├── npm/               # Package manager
│   └── ...                # Other dependencies
│
├── test/                   # Test suites (59MB, 2000+ test files)
│   ├── parallel/          # Parallel executable tests
│   ├── sequential/        # Sequential tests
│   ├── cctest/            # C++ unit tests
│   ├── node-api/          # N-API tests
│   ├── fixtures/          # Test fixtures
│   └── common/            # Test utilities
│
├── doc/                    # Documentation (17MB)
│   ├── api/               # API documentation
│   └── contributing/      # Contribution guides (45 files)
│
├── tools/                  # Build and development tools (5.5MB)
│   ├── gyp/               # GYP build system
│   ├── dep_updaters/      # Dependency update scripts
│   ├── eslint/            # JavaScript linting
│   └── cpplint.py         # C++ linting
│
├── node.gyp                # Main build configuration
├── configure.py            # Pre-build configuration script
├── Makefile                # Top-level Unix build
├── vcbuild.bat             # Windows build script
├── CONTRIBUTING.md         # Basic contribution guide
├── GOVERNANCE.md           # Project governance
└── onboarding.md           # Collaborator onboarding
```

---

## Architecture Patterns

### 1. BaseObject Pattern

**Purpose**: Base class for all C++ objects that wrap JavaScript values

**Key Features**:
- Stores C++ object pointer in JavaScript object's internal field
- Tracks object lifecycle and garbage collection
- Provides consistent interface for native objects

**Example**:
```cpp
// src/tcp_wrap.h
class TCPWrap : public ConnectionWrap<TCPWrap, uv_tcp_t> {
  // Inherits from BaseObject via ConnectionWrap
};
```

### 2. AsyncWrap Pattern

**Purpose**: Track asynchronous operations for debugging and monitoring

**Key Features**:
- Base class for all async operations
- Enables async_hooks API
- Maintains async context across callbacks
- Assigns unique async IDs

**Example**:
```cpp
// All async operations inherit from AsyncWrap
class FSReqCallback : public AsyncWrap {
  // File system async operations
};
```

### 3. Environment Per Isolate

**Purpose**: Manage per-isolate state in a thread-safe manner

**Key Features**:
- One Environment per V8 Isolate
- Contains event loop, context, and isolate
- Thread-local for Worker threads
- Manages built-in module cache

### 4. Handle/Request Pattern (from libuv)

**Purpose**: Distinguish between long-lived resources and one-time operations

**Handles** (long-lived):
- TCP/UDP sockets
- Timers
- File system watchers
- Signals

**Requests** (one-time):
- Write operations
- Connect operations
- DNS lookups
- File system operations

### 5. Module Binding System

**Purpose**: Register and load native modules

**Process**:
```cpp
// 1. Register module
NODE_MODULE(node_tcp_wrap, Initialize)

// 2. Initialize function
void Initialize(Local<Object> target, ...) {
  // Export C++ functions to JavaScript
}

// 3. JavaScript loads module
const binding = internalBinding('tcp_wrap');
```

---

## Important Concepts for Contributors

### V8 Isolates

- **What**: Separate JavaScript engine instance with its own heap
- **When**: Main thread has one isolate; each Worker thread has its own
- **Why**: Thread isolation and memory separation

### V8 Contexts

- **What**: Execution environment within an isolate
- **When**: Multiple contexts enable sandboxing (Realms, vm module)
- **Why**: Security and isolation

### JavaScript Value Handles

```cpp
// Local<> - Temporary, stack-allocated (must be in HandleScope)
Local<String> str = String::NewFromUtf8(isolate, "hello");

// Global<> - Long-lived, heap-allocated (manual destruction)
Global<Function> callback;
callback.Reset(isolate, function);
```

### Internal Fields

- **What**: Storage slots in JavaScript objects for C++ pointers
- **When**: Every BaseObject uses internal field 0 for C++ pointer
- **Why**: Fast access from JavaScript object to C++ object

### Aliased Buffers

- **What**: Buffers that mirror memory between C++ and JavaScript
- **When**: Performance-critical shared data (e.g., performance counters)
- **Why**: Zero-copy data sharing

---

## Codebase Statistics

- **C++ Files**: 110+ files in `src/`
- **JavaScript Files**: 50+ core modules in `lib/`
- **Test Files**: 2000+ JavaScript test files
- **C++ Lines**: ~250,000 lines
- **JavaScript Lines**: ~300,000 lines
- **Total with deps**: 3+ million lines

---

## Essential Files for New Contributors

### Entry Points
- `src/node.cc` - Main executable and initialization
- `src/node_main_instance.cc` - Instance management

### Foundational Classes
- `src/env.h` / `.cc` - Environment class (central to all operations)
- `src/base_object.h` / `.cc` - Base for native objects
- `src/async_wrap.h` / `.cc` - Async operation tracking

### Key APIs
- `src/node.h` - Public Node.js embedder API
- `src/js_native_api_v8.h` - N-API implementation
- `src/v8_util.h` - V8 utility functions

### Documentation
- `src/README.md` - C++ codebase overview
- `doc/contributing/components-in-core.md` - Core components
- `doc/contributing/cpp-style-guide.md` - C++ style guide

---

## Next Steps

1. Read `CONTRIBUTION_GUIDE.md` for detailed contribution process
2. Read `BUILD_SYSTEM_GUIDE.md` for build setup
3. Read `CORE_SUBSYSTEMS.md` for deep dives into specific areas
4. Read `TESTING_GUIDE.md` for testing infrastructure

---

**Last Updated**: 2025-11-19
**Node.js Version**: Based on current main branch
