# Node.js Core Subsystems Deep Dive

> Detailed guide to Node.js core subsystems for contributors

## Table of Contents
- [Overview](#overview)
- [V8 JavaScript Engine Integration](#v8-javascript-engine-integration)
- [libuv Event Loop](#libuv-event-loop)
- [HTTP Subsystem](#http-subsystem)
- [File System](#file-system)
- [Streams](#streams)
- [Networking (TCP/UDP)](#networking-tcpudp)
- [Crypto & TLS](#crypto--tls)
- [Worker Threads](#worker-threads)
- [Module System](#module-system)
- [Async Hooks](#async-hooks)
- [Inspector & Debugging](#inspector--debugging)
- [Permission System](#permission-system)

---

## Overview

Node.js consists of multiple interconnected subsystems, each responsible for specific functionality. Understanding these subsystems is crucial for effective contributions.

### Subsystem Map

```
┌─────────────────────────────────────────────────────────┐
│                   User Application                      │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│              JavaScript API Layer                       │
│  HTTP │ FS │ Streams │ Net │ Crypto │ Events │ etc.   │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│           C++ Binding & Core Layer                      │
│  Module Binding │ Async Wrap │ BaseObject │ Env        │
└────┬────────┬──────────┬──────────┬──────────┬─────────┘
     │        │          │          │          │
┌────┴───┐ ┌─┴─────┐ ┌──┴─────┐ ┌──┴─────┐ ┌─┴────────┐
│   V8   │ │ libuv │ │OpenSSL │ │ nghttp2│ │ Others   │
└────────┘ └───────┘ └────────┘ └────────┘ └──────────┘
```

---

## V8 JavaScript Engine Integration

### What is V8?

V8 is Google's open-source JavaScript engine that:
- Compiles JavaScript to native machine code
- Manages memory and garbage collection
- Provides the JavaScript runtime
- Implements ECMAScript standards

### Location

- **Dependency**: `deps/v8/`
- **Integration**: `src/` (various files)
- **API Headers**: `deps/v8/include/v8*.h`

### Key Concepts

#### 1. Isolates

**What**: Independent instance of V8 engine with its own heap

**Purpose**:
- Thread isolation
- Memory separation
- Independent execution contexts

**In Node.js**:
- Main thread has one isolate
- Each Worker thread has its own isolate
- One Environment per Isolate

**Code Example**:
```cpp
// src/node_main_instance.cc
v8::Isolate* isolate = NewIsolate(allocator, event_loop);
v8::Isolate::Scope isolate_scope(isolate);
```

#### 2. Contexts

**What**: Execution environment within an isolate

**Purpose**:
- Sandboxing
- Separate global objects
- Security boundaries

**In Node.js**:
- Main context per Environment
- Additional contexts for vm module
- Realms for compartmentalization

**Code Example**:
```cpp
v8::Local<v8::Context> context = v8::Context::New(isolate);
v8::Context::Scope context_scope(context);
// JavaScript code executes in this context
```

#### 3. Handles

**Local Handles** (stack-allocated):
```cpp
v8::Local<v8::String> str =
    v8::String::NewFromUtf8(isolate, "hello").ToLocalChecked();
// Automatically cleaned up when scope exits
```

**Global Handles** (heap-allocated):
```cpp
v8::Global<v8::Function> persistent_callback;
persistent_callback.Reset(isolate, callback);
// Must be manually cleaned up
persistent_callback.Reset();  // Clean up
```

**HandleScope**:
```cpp
{
  v8::HandleScope handle_scope(isolate);
  v8::Local<v8::String> str = /* ... */;
  // All local handles created here are cleaned up at scope exit
}
// str is no longer valid
```

#### 4. Templates

**Function Templates** (for classes/constructors):
```cpp
v8::Local<v8::FunctionTemplate> t =
    v8::FunctionTemplate::New(isolate, Constructor);
t->SetClassName(
    v8::String::NewFromUtf8(isolate, "MyClass").ToLocalChecked());
t->InstanceTemplate()->SetInternalFieldCount(1);
```

**Object Templates** (for objects):
```cpp
v8::Local<v8::ObjectTemplate> obj_template =
    v8::ObjectTemplate::New(isolate);
obj_template->Set(
    v8::String::NewFromUtf8(isolate, "myMethod").ToLocalChecked(),
    v8::FunctionTemplate::New(isolate, MyMethod));
```

### Important Files

- `src/node.h` - Node.js embedder API
- `src/env.h` - Environment (contains Isolate & Context)
- `src/node_main_instance.cc` - Isolate creation and management
- `src/api/environment.cc` - Environment API

### V8 Integration Patterns

**Accessing Isolate**:
```cpp
// From Environment
v8::Isolate* isolate = env->isolate();

// From arguments
void MyFunction(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();
}
```

**Creating JavaScript Values**:
```cpp
// String
v8::Local<v8::String> str =
    v8::String::NewFromUtf8(isolate, "hello").ToLocalChecked();

// Number
v8::Local<v8::Number> num = v8::Number::New(isolate, 42);

// Object
v8::Local<v8::Object> obj = v8::Object::New(isolate);

// Array
v8::Local<v8::Array> arr = v8::Array::New(isolate, 10);
```

**Calling JavaScript from C++**:
```cpp
v8::Local<v8::Function> callback = /* ... */;
v8::Local<v8::Value> argv[] = {
    v8::String::NewFromUtf8(isolate, "arg1").ToLocalChecked()
};
v8::MaybeLocal<v8::Value> result =
    callback->Call(context, v8::Null(isolate), 1, argv);
```

---

## libuv Event Loop

### What is libuv?

libuv is a multi-platform library that provides:
- Asynchronous I/O
- Event loop implementation
- Thread pool
- Platform abstraction

### Location

- **Dependency**: `deps/uv/`
- **Integration**: Throughout `src/`
- **Headers**: `deps/uv/include/uv.h`

### Event Loop Architecture

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout(), setInterval()
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O callbacks deferred to next iteration
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  Internal use only
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │  setImmediate()
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  socket.on('close', ...)
   └───────────────────────────┘
```

### Key Concepts

#### 1. Handles (Long-lived Resources)

**Types**:
- `uv_tcp_t` - TCP sockets
- `uv_udp_t` - UDP sockets
- `uv_timer_t` - Timers
- `uv_fs_event_t` - File system events
- `uv_signal_t` - Signal handlers
- `uv_process_t` - Child processes

**Example** (TCP handle):
```cpp
// src/tcp_wrap.cc
class TCPWrap : public ConnectionWrap<TCPWrap, uv_tcp_t> {
  uv_tcp_t handle_;

  void Listen(int backlog) {
    int err = uv_listen(
        reinterpret_cast<uv_stream_t*>(&handle_),
        backlog,
        OnConnection);
  }
};
```

#### 2. Requests (One-time Operations)

**Types**:
- `uv_write_t` - Write operations
- `uv_connect_t` - Connection requests
- `uv_fs_t` - File system operations
- `uv_getaddrinfo_t` - DNS lookups
- `uv_work_t` - Thread pool work

**Example** (Write request):
```cpp
uv_write_t* req = new uv_write_t;
req->data = callback_data;

uv_buf_t buf = uv_buf_init(data, size);
uv_write(req, stream, &buf, 1, OnWriteComplete);
```

#### 3. Thread Pool

**Purpose**: Execute blocking operations without blocking event loop

**Default Size**: 4 threads (configurable via `UV_THREADPOOL_SIZE`)

**Used For**:
- File system operations (except `fs.watch()`)
- DNS lookups (`getaddrinfo`)
- Some crypto operations
- `zlib` compression

**Example**:
```cpp
void DoWork(uv_work_t* req) {
  // Runs in thread pool
  WorkData* data = static_cast<WorkData*>(req->data);
  data->result = expensive_computation();
}

void AfterWork(uv_work_t* req, int status) {
  // Runs in event loop thread
  WorkData* data = static_cast<WorkData*>(req->data);
  // Process result, call JavaScript callback
}

uv_queue_work(env->event_loop(), req, DoWork, AfterWork);
```

### Important Files

- `src/env.h` - Environment contains `uv_loop_t*`
- `src/handle_wrap.h` - Base class for handle wrappers
- `src/req_wrap.h` - Base class for request wrappers
- `src/stream_base.h` - Stream abstraction
- `src/tcp_wrap.cc` - TCP handle wrapper
- `src/udp_wrap.cc` - UDP handle wrapper
- `src/fs_event_wrap.cc` - FS event wrapper

### Event Loop Integration

**Getting Event Loop**:
```cpp
Environment* env = /* ... */;
uv_loop_t* loop = env->event_loop();
```

**Running Event Loop**:
```cpp
// src/node_main_instance.cc
do {
  uv_run(env->event_loop(), UV_RUN_DEFAULT);
  more = uv_loop_alive(env->event_loop());
  if (more && !env->is_stopping()) continue;

  env->RunBeforeExitCallbacks();
  more = uv_loop_alive(env->event_loop());
} while (more && !env->is_stopping());
```

---

## HTTP Subsystem

### Architecture

```
┌─────────────────────────────────────────────────┐
│  lib/_http_server.js, lib/_http_client.js      │  JavaScript API
└────────────────┬────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────┐
│  src/node_http_parser.cc                       │  HTTP parsing (llhttp)
└────────────────┬────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────┐
│  src/tcp_wrap.cc                               │  TCP layer
└────────────────┬────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────┐
│  deps/uv (libuv)                               │  I/O handling
└─────────────────────────────────────────────────┘
```

### Key Components

#### 1. HTTP Parser (llhttp)

**Location**: `deps/llhttp/`

**Integration**: `src/node_http_parser.cc`

**Purpose**: Parse HTTP requests and responses

**Features**:
- HTTP/1.1 protocol parsing
- Header parsing
- Chunked encoding
- Keep-alive support

#### 2. HTTP/1.1 Implementation

**Files**:
- `lib/_http_server.js` - HTTP server
- `lib/_http_client.js` - HTTP client
- `lib/_http_incoming.js` - IncomingMessage
- `lib/_http_outgoing.js` - OutgoingMessage
- `lib/_http_common.js` - Common utilities

**Server Flow**:
```javascript
// lib/_http_server.js
const server = http.createServer((req, res) => {
  // Request received
});

// Internally:
// 1. TCP connection accepted (tcp_wrap.cc)
// 2. Data received → HTTP parser (node_http_parser.cc)
// 3. Headers parsed → emit 'request' event
// 4. Body chunks → IncomingMessage readable stream
// 5. Response → OutgoingMessage writable stream
```

#### 3. HTTP/2 Implementation

**Location**: `src/node_http2.cc`, `lib/internal/http2/`

**Dependency**: `deps/nghttp2/`

**Features**:
- Binary framing
- Multiplexing
- Server push
- Header compression (HPACK)

#### 4. HTTPS (HTTP over TLS)

**Files**:
- `lib/https.js` - HTTPS module
- `lib/_tls_wrap.js` - TLS wrapper

**Flow**:
```
HTTP Request
    ↓
TLS Encryption (lib/_tls_wrap.js)
    ↓
TCP Socket (src/tcp_wrap.cc)
    ↓
Network
```

### Common Contribution Areas

- Performance optimizations
- Keep-alive handling improvements
- Error handling enhancements
- Header parsing edge cases
- Memory leak fixes

### Important Files

| File | Purpose |
|------|---------|
| `lib/_http_server.js` | HTTP server implementation |
| `lib/_http_client.js` | HTTP client (request) |
| `lib/_http_agent.js` | Connection pooling |
| `src/node_http_parser.cc` | llhttp integration |
| `src/node_http2.cc` | HTTP/2 implementation |
| `deps/llhttp/` | HTTP parser |
| `deps/nghttp2/` | HTTP/2 library |

---

## File System

### Architecture

```
┌────────────────────────────────┐
│  lib/fs.js                     │  Promise API, callback API
│  lib/fs/promises.js            │
└────────────┬───────────────────┘
             │
┌────────────┴───────────────────┐
│  src/node_file.cc              │  C++ implementation
└────────────┬───────────────────┘
             │
┌────────────┴───────────────────┐
│  deps/uv (libuv)               │  Platform abstraction
└────────────────────────────────┘
```

### Key Features

**Synchronous Operations**:
```javascript
const data = fs.readFileSync('/path/to/file', 'utf8');
```
- Block until complete
- Run on event loop thread
- Use sparingly

**Asynchronous Operations** (Callback):
```javascript
fs.readFile('/path/to/file', 'utf8', (err, data) => {
  // Runs in thread pool
});
```
- Non-blocking
- Run in thread pool
- Preferred for I/O

**Asynchronous Operations** (Promise):
```javascript
const data = await fs.promises.readFile('/path/to/file', 'utf8');
```
- Non-blocking
- Modern async/await syntax
- Recommended

### C++ Implementation

**File Operations** (`src/node_file.cc`):
```cpp
void Open(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);

  // Extract arguments
  BufferValue path(env->isolate(), args[0]);
  int flags = args[1].As<Int32>()->Value();
  int mode = args[2].As<Int32>()->Value();

  // Create request
  FSReqWrap* req_wrap = /* ... */;

  // Queue to thread pool
  int err = uv_fs_open(env->event_loop(),
                       req_wrap->req(),
                       *path,
                       flags,
                       mode,
                       AfterOpen);
}
```

### File System Watching

**Location**: `src/fs_event_wrap.cc`

**Mechanism**:
- Linux: inotify
- macOS: FSEvents
- Windows: ReadDirectoryChangesW

**API**:
```javascript
fs.watch('/path', (eventType, filename) => {
  console.log(`${filename} changed: ${eventType}`);
});
```

### Important Files

| File | Purpose |
|------|---------|
| `lib/fs.js` | Main FS module |
| `lib/fs/promises.js` | Promise-based API |
| `src/node_file.cc` | C++ file operations |
| `src/node_dir.cc` | Directory operations |
| `src/fs_event_wrap.cc` | File watching |

---

## Streams

### Stream Types

1. **Readable**: Read data from source
2. **Writable**: Write data to destination
3. **Duplex**: Both readable and writable
4. **Transform**: Duplex that modifies data

### Architecture

```
┌─────────────────────────────────────┐
│  User Code                          │
└────────────┬────────────────────────┘
             │
┌────────────┴────────────────────────┐
│  lib/_stream_readable.js            │  Readable Stream
│  lib/_stream_writable.js            │  Writable Stream
│  lib/_stream_duplex.js              │  Duplex Stream
│  lib/_stream_transform.js           │  Transform Stream
└────────────┬────────────────────────┘
             │
┌────────────┴────────────────────────┐
│  lib/internal/streams/buffer_list.js│  Internal buffering
└─────────────────────────────────────┘
```

### Readable Stream

**Flow Control**:
```javascript
const readable = fs.createReadStream('/file');

// Flowing mode (automatic)
readable.on('data', (chunk) => {
  console.log('Received chunk:', chunk);
});

// Paused mode (manual)
readable.on('readable', () => {
  let chunk;
  while ((chunk = readable.read()) !== null) {
    console.log('Read chunk:', chunk);
  }
});
```

**Internal Implementation**:
- `_read()` - Fill internal buffer
- `push()` - Add data to buffer
- `read()` - Consume data from buffer

### Writable Stream

**Writing**:
```javascript
const writable = fs.createWriteStream('/file');

writable.write('hello');
writable.write('world');
writable.end();  // Finish writing
```

**Backpressure**:
```javascript
const canContinue = writable.write('data');
if (!canContinue) {
  // Internal buffer full, pause until 'drain'
  writable.once('drain', () => {
    // Can write more
  });
}
```

**Internal Implementation**:
- `_write()` - Write chunk
- `_writev()` - Write multiple chunks (optimization)
- `_final()` - Cleanup before close

### Transform Stream

**Example**:
```javascript
const { Transform } = require('stream');

class UpperCase extends Transform {
  _transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
}

process.stdin
  .pipe(new UpperCase())
  .pipe(process.stdout);
```

### Pipeline

**Recommended Pattern**:
```javascript
const { pipeline } = require('stream');

pipeline(
  fs.createReadStream('input.txt'),
  zlib.createGzip(),
  fs.createWriteStream('output.txt.gz'),
  (err) => {
    if (err) console.error('Pipeline failed:', err);
    else console.log('Pipeline succeeded');
  }
);
```

### Important Files

| File | Purpose |
|------|---------|
| `lib/_stream_readable.js` | Readable stream implementation |
| `lib/_stream_writable.js` | Writable stream implementation |
| `lib/_stream_duplex.js` | Duplex stream (both) |
| `lib/_stream_transform.js` | Transform stream |
| `lib/_stream_passthrough.js` | Pass-through (no transform) |
| `lib/stream.js` | Public API exports |

---

## Networking (TCP/UDP)

### TCP

**Location**: `lib/net.js`, `src/tcp_wrap.cc`

**Server**:
```javascript
const server = net.createServer((socket) => {
  socket.on('data', (data) => {
    socket.write(data);  // Echo server
  });
});

server.listen(3000);
```

**C++ Implementation**:
```cpp
// src/tcp_wrap.cc
void TCPWrap::Listen(const FunctionCallbackInfo<Value>& args) {
  TCPWrap* wrap;
  ASSIGN_OR_RETURN_UNWRAP(&wrap, args.Holder());

  int backlog = args[0]->Int32Value();
  int err = uv_listen(
      reinterpret_cast<uv_stream_t*>(&wrap->handle_),
      backlog,
      OnConnection);
  args.GetReturnValue().Set(err);
}
```

### UDP

**Location**: `lib/dgram.js`, `src/udp_wrap.cc`

**Server**:
```javascript
const server = dgram.createSocket('udp4');

server.on('message', (msg, rinfo) => {
  console.log(`Received from ${rinfo.address}:${rinfo.port}`);
});

server.bind(3000);
```

### DNS

**Location**: `lib/dns.js`, `src/cares_wrap.cc`

**Dependency**: `deps/cares/` (c-ares library)

**Resolution**:
```javascript
dns.lookup('example.com', (err, address) => {
  console.log('IP:', address);
});
```

### Important Files

| File | Purpose |
|------|---------|
| `lib/net.js` | TCP networking |
| `src/tcp_wrap.cc` | TCP C++ wrapper |
| `src/connection_wrap.cc` | Connection base class |
| `src/stream_wrap.cc` | Stream wrapper |
| `lib/dgram.js` | UDP networking |
| `src/udp_wrap.cc` | UDP C++ wrapper |
| `lib/dns.js` | DNS resolution |
| `src/cares_wrap.cc` | c-ares wrapper |

---

## Crypto & TLS

### Crypto

**Location**: `lib/crypto.js`, `src/crypto/`

**Dependency**: `deps/openssl/`

**Features**:
- Hashing (SHA, MD5, etc.)
- HMAC
- Ciphers (AES, DES, etc.)
- Public/private key crypto
- Random number generation
- Certificates

**Example**:
```javascript
const crypto = require('crypto');

// Hash
const hash = crypto.createHash('sha256');
hash.update('hello');
console.log(hash.digest('hex'));

// HMAC
const hmac = crypto.createHmac('sha256', 'secret');
hmac.update('data');
console.log(hmac.digest('hex'));
```

### TLS/SSL

**Location**: `lib/tls.js`, `lib/_tls_wrap.js`

**Server**:
```javascript
const tls = require('tls');
const fs = require('fs');

const server = tls.createServer({
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
}, (socket) => {
  socket.write('secure data');
});

server.listen(8000);
```

**Client**:
```javascript
const socket = tls.connect(8000, {
  ca: [fs.readFileSync('ca-cert.pem')]
}, () => {
  console.log('Connected securely');
});
```

### Important Files

| File | Purpose |
|------|---------|
| `lib/crypto.js` | Crypto module API |
| `src/crypto/` | Crypto C++ implementations |
| `src/crypto/crypto_hash.cc` | Hashing |
| `src/crypto/crypto_cipher.cc` | Ciphers |
| `src/crypto/crypto_keys.cc` | Key management |
| `lib/tls.js` | TLS module |
| `lib/_tls_wrap.js` | TLS stream wrapper |
| `deps/openssl/` | OpenSSL library |

---

## Worker Threads

### Purpose

- Run JavaScript in parallel threads
- CPU-intensive operations
- Isolate execution contexts

### Architecture

```
┌─────────────────────────────┐
│  Main Thread                │
│  - Main isolate             │
│  - Main event loop          │
└────────────┬────────────────┘
             │
     ┌───────┴───────┐
     │               │
┌────┴─────┐  ┌─────┴────┐
│ Worker 1 │  │ Worker 2 │
│ - Isolate│  │ - Isolate│
│ - Loop   │  │ - Loop   │
└──────────┘  └──────────┘
```

### API

**Creating Worker**:
```javascript
const { Worker } = require('worker_threads');

const worker = new Worker('./worker.js', {
  workerData: { value: 42 }
});

worker.on('message', (msg) => {
  console.log('From worker:', msg);
});

worker.on('error', (err) => {
  console.error('Worker error:', err);
});

worker.on('exit', (code) => {
  console.log('Worker exited:', code);
});
```

**Worker Script**:
```javascript
const { workerData, parentPort } = require('worker_threads');

// Access worker data
console.log('Worker data:', workerData.value);

// Send message to parent
parentPort.postMessage({ result: 'done' });
```

### Message Passing

**Structured Clone**:
```javascript
// Copied (cloned)
worker.postMessage({ data: [1, 2, 3] });
```

**Transfer** (move ownership):
```javascript
const buffer = new ArrayBuffer(1024);
worker.postMessage({ buffer }, [buffer]);
// buffer is now unusable in main thread
```

### C++ Implementation

**Location**: `src/node_worker.cc`

**Key Features**:
- Creates new Isolate per worker
- Manages message queues
- Handles worker lifecycle

### Important Files

| File | Purpose |
|------|---------|
| `lib/worker_threads.js` | Worker threads API |
| `src/node_worker.cc` | Worker C++ implementation |
| `src/node_messaging.cc` | Message passing |

---

## Module System

### CommonJS

**Location**: `lib/module.js`

**Resolution Algorithm**:
```
require('foo')
  → Check if core module
  → Check node_modules/foo
  → Check parent node_modules
  → Repeat until found or root reached
```

**Implementation**:
```javascript
// lib/module.js
Module.prototype.require = function(id) {
  return Module._load(id, this);
};

Module._load = function(request, parent) {
  // 1. Resolve filename
  const filename = Module._resolveFilename(request, parent);

  // 2. Check cache
  const cached = Module._cache[filename];
  if (cached) return cached.exports;

  // 3. Create module
  const module = new Module(filename, parent);
  Module._cache[filename] = module;

  // 4. Load module
  module.load(filename);

  return module.exports;
};
```

### ES Modules (ESM)

**Location**: `lib/internal/modules/esm/`

**Files**:
- `loader.js` - Module loader
- `module_job.js` - Module loading job
- `module_map.js` - Module cache
- `resolve.js` - Resolution algorithm

**C++ Integration**: `src/module_wrap.cc`

**Loading**:
```javascript
import { foo } from './module.mjs';
```

### Built-in Modules

**Location**: `lib/`, `src/node_builtins.cc`

**Embedding**:
- JavaScript modules are embedded at build time
- `tools/js2c.cc` converts JS to C++ headers
- Loaded from memory, not disk

### Important Files

| File | Purpose |
|------|---------|
| `lib/module.js` | CommonJS implementation |
| `lib/internal/modules/esm/loader.js` | ESM loader |
| `src/module_wrap.cc` | ESM C++ wrapper |
| `src/node_builtins.cc` | Built-in module management |
| `src/node_binding.cc` | Native module registration |

---

## Async Hooks

### Purpose

Track asynchronous operations for:
- Debugging
- Async context tracking
- Performance monitoring
- APM tools

### API

```javascript
const async_hooks = require('async_hooks');

const hook = async_hooks.createHook({
  init(asyncId, type, triggerAsyncId, resource) {
    console.log(`Async op created: ${type} (${asyncId})`);
  },
  before(asyncId) {
    console.log(`Before async op: ${asyncId}`);
  },
  after(asyncId) {
    console.log(`After async op: ${asyncId}`);
  },
  destroy(asyncId) {
    console.log(`Async op destroyed: ${asyncId}`);
  }
});

hook.enable();
```

### C++ Implementation

**Location**: `src/async_wrap.cc`

**AsyncWrap Class**:
```cpp
class AsyncWrap : public BaseObject {
 public:
  AsyncWrap(Environment* env,
            Local<Object> object,
            ProviderType provider);

  inline double get_async_id() const { return async_id_; }
  inline double get_trigger_async_id() const { return trigger_async_id_; }

 private:
  double async_id_;
  double trigger_async_id_;
};
```

**All Async Operations Inherit**:
```cpp
class TCPWrap : public ConnectionWrap<TCPWrap, uv_tcp_t> {
  // Inherits from AsyncWrap through ConnectionWrap
};
```

### Important Files

| File | Purpose |
|------|---------|
| `lib/async_hooks.js` | Async hooks API |
| `src/async_wrap.cc` | AsyncWrap implementation |
| `src/async_context_frame.cc` | Async context tracking |

---

## Inspector & Debugging

### Inspector Protocol

**Location**: `src/inspector/`, `deps/inspector_protocol/`

**Purpose**: Chrome DevTools Protocol support

**Features**:
- Set breakpoints
- Step through code
- Inspect variables
- CPU/heap profiling
- Coverage collection

### Usage

```bash
# Start with inspector
node --inspect script.js

# Pause on start
node --inspect-brk script.js

# Custom port
node --inspect=9229 script.js
```

**Connect**:
1. Open Chrome
2. Navigate to `chrome://inspect`
3. Click "inspect" on target

### C++ Implementation

**Inspector Session**:
```cpp
// src/inspector/main_thread_interface.cc
// Handles inspector protocol messages
```

### Important Files

| File | Purpose |
|------|---------|
| `src/inspector/` | Inspector implementation |
| `src/inspector_agent.cc` | Inspector agent |
| `src/inspector_socket.cc` | WebSocket transport |
| `lib/inspector.js` | Inspector JavaScript API |

---

## Permission System

### Purpose

Control access to:
- File system
- Network
- Child processes
- Worker threads

### Location

`src/permission/`

### API

```bash
# Restrict file system
node --experimental-permission --allow-fs-read=/path script.js

# Restrict network
node --experimental-permission --allow-network script.js
```

### Important Files

| File | Purpose |
|------|---------|
| `src/permission/permission.h` | Permission base class |
| `src/permission/fs_permission.cc` | File system permissions |
| `lib/internal/process/permission.js` | Permission API |

---

## Summary

Each subsystem in Node.js has:
- **JavaScript API** layer (`lib/`)
- **C++ implementation** (`src/`)
- **External dependency** (`deps/`) where applicable

**To contribute effectively**:
1. Understand the subsystem architecture
2. Read both JavaScript and C++ code
3. Study how layers interact
4. Test thoroughly across all layers

**Next Steps**:
- Read specific subsystem code
- Run examples to see data flow
- Write tests to understand behavior
- Start with small bug fixes

---

**Last Updated**: 2025-11-19
**For More**: See `ARCHITECTURE_OVERVIEW.md` and official documentation
