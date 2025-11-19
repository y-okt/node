# Node.js Testing Infrastructure Guide

> Comprehensive guide to testing in Node.js for contributors

## Table of Contents
- [Overview](#overview)
- [Test Directory Structure](#test-directory-structure)
- [Test Categories](#test-categories)
- [Writing JavaScript Tests](#writing-javascript-tests)
- [Writing C++ Tests](#writing-c-tests)
- [Running Tests](#running-tests)
- [Test Utilities](#test-utilities)
- [Debugging Tests](#debugging-tests)
- [CI/CD Testing](#cicd-testing)
- [Best Practices](#best-practices)
- [Common Test Patterns](#common-test-patterns)

---

## Overview

### Testing Philosophy

Node.js uses comprehensive testing to ensure:
- **Correctness**: Features work as specified
- **Regression Prevention**: Bugs don't reappear
- **Cross-Platform**: Works on all supported platforms
- **Performance**: No unintended slowdowns
- **Backward Compatibility**: APIs remain stable

### Test Statistics

- **Total Tests**: 2000+ JavaScript test files
- **Test Lines**: Hundreds of thousands of lines
- **C++ Tests**: Comprehensive unit test coverage
- **CI Platforms**: 20+ configurations
- **Test Time**: Hours for full suite

---

## Test Directory Structure

```
test/
├── parallel/              # Tests that can run in parallel (LARGEST)
│   ├── test-http-*.js    # HTTP subsystem tests
│   ├── test-fs-*.js      # File system tests
│   ├── test-stream-*.js  # Stream tests
│   └── ...               # 1500+ test files
│
├── sequential/            # Tests requiring serialization
│   ├── test-async-wrap-*
│   ├── test-child-process-*
│   └── ...
│
├── cctest/                # C++ unit tests
│   ├── test_base64.cc
│   ├── test_node_postmortem_metadata.cc
│   └── ...
│
├── node-api/              # Node-API (N-API) tests
│   ├── test_*/           # Native addon tests
│   └── ...
│
├── js-native-api/         # JavaScript native API tests
│   └── ...
│
├── addons/                # Native addon examples and tests
│   ├── hello-world/
│   ├── async-hooks/
│   └── ...
│
├── es-module/             # ES module functionality tests
│   ├── test-esm-*.js
│   └── ...
│
├── fixtures/              # Test fixture files and data
│   ├── keys/             # SSL/TLS certificates
│   ├── snapshot/         # Snapshot test files
│   ├── wpt/              # Web Platform Tests
│   └── ...
│
├── internet/              # Tests requiring internet connectivity
│   └── test-dns-*.js
│
├── async-hooks/           # Async hooks tests
│   └── test-*.js
│
├── benchmark/             # Benchmark scripts
│   └── ...
│
├── embedding/             # Embedding Node.js tests
│   └── ...
│
├── fuzzers/               # Fuzzing tests
│   └── ...
│
├── known_issues/          # Tests for documented limitations
│   └── ...
│
├── message/               # Expected output message tests
│   ├── test.js           # Message test file
│   └── test.out          # Expected output
│
├── report/                # Diagnostic report tests
│   └── ...
│
├── pummel/                # Long-running stress tests
│   └── ...
│
├── pseudo-tty/            # TTY-specific tests
│   └── ...
│
├── tick-processor/        # V8 tick processor tests
│   └── ...
│
├── wasi/                  # WebAssembly System Interface tests
│   └── ...
│
├── wpt/                   # Web Platform Tests
│   └── ...
│
├── common/                # Test utilities and helpers
│   ├── index.js          # Common test utilities
│   ├── fixtures.js       # Fixture management
│   ├── tmpdir.js         # Temporary directory helper
│   ├── assertSnapshot.js # Snapshot testing
│   └── ...
│
└── README.md              # Testing documentation
```

---

## Test Categories

### 1. Parallel Tests (`test/parallel/`)

**Characteristics**:
- Can run simultaneously with other tests
- Most common test type
- Fast execution when parallelized
- Should not interfere with each other

**Use For**:
- Unit tests
- Most feature tests
- Isolated functionality tests

**Example Naming**:
- `test-http-server-close.js`
- `test-fs-readfile.js`
- `test-stream-pipe.js`

**Requirements**:
- Must be independent
- No shared global state
- Use random ports for servers
- Clean up resources

### 2. Sequential Tests (`test/sequential/`)

**Characteristics**:
- Run one at a time
- Can modify global state
- May be resource-intensive
- Slower overall execution

**Use For**:
- Tests that modify environment variables
- Tests that use specific ports
- Tests with global side effects
- Resource-intensive tests

**Example**:
```javascript
// test/sequential/test-async-wrap-getasyncid.js
// Tests async_hooks which affects global state
```

### 3. C++ Tests (`test/cctest/`)

**Characteristics**:
- Unit tests for C++ code
- Use Google Test (gtest) framework
- Compiled separately
- Fast execution

**Use For**:
- C++ utility functions
- Low-level API testing
- Core infrastructure testing

**Example**:
```cpp
// test/cctest/test_base64.cc
TEST(Base64, Encode) {
  const char* input = "hello";
  std::string output = base64_encode(input, 5);
  EXPECT_EQ(output, "aGVsbG8=");
}
```

### 4. Node-API Tests (`test/node-api/`)

**Characteristics**:
- Test Node-API (N-API) functionality
- Native addon tests
- Verify ABI stability

**Structure**:
```
test/node-api/test_name/
├── binding.c          # Native implementation
├── binding.gyp        # Build configuration
├── test.js            # JavaScript test
└── README.md
```

### 5. ES Module Tests (`test/es-module/`)

**Characteristics**:
- Test ES module functionality
- Import/export syntax
- Module resolution

**Use For**:
- ES module loading
- Module resolution
- Dynamic imports
- Top-level await

### 6. Message Tests (`test/message/`)

**Characteristics**:
- Test exact output messages
- Verify error formatting
- Check warning messages

**Structure**:
```
test/message/test-name.js    # Test script
test/message/test-name.out   # Expected output
```

### 7. Known Issues (`test/known_issues/`)

**Characteristics**:
- Tests for known bugs/limitations
- Expected to fail
- Document issues

**Purpose**:
- Track known problems
- Prevent regressions when fixed
- Document limitations

---

## Writing JavaScript Tests

### Basic Test Structure

```javascript
'use strict';

// 1. Require common utilities
const common = require('../common');

// 2. Skip if needed
if (!common.hasCrypto) {
  common.skip('missing crypto');
}

// 3. Require modules
const assert = require('assert');
const http = require('http');

// 4. Write tests
{
  // Test HTTP server
  const server = http.createServer(common.mustCall((req, res) => {
    assert.strictEqual(req.method, 'GET');
    res.end('hello');
  }));

  server.listen(0, common.mustCall(() => {
    const port = server.address().port;

    http.get(`http://localhost:${port}`, common.mustCall((res) => {
      assert.strictEqual(res.statusCode, 200);

      let data = '';
      res.on('data', (chunk) => {
        data += chunk;
      });

      res.on('end', common.mustCall(() => {
        assert.strictEqual(data, 'hello');
        server.close();
      }));
    }));
  }));
}
```

### Test File Requirements

1. **'use strict'**: Always first line
2. **require('../common')**: Load test utilities
3. **Assertions**: Use `assert` module
4. **Cleanup**: Close servers, clear timers, etc.
5. **Isolation**: Use blocks `{ }` for test grouping

### Naming Conventions

**Format**: `test-<subsystem>-<specific-feature>.js`

**Examples**:
- `test-http-server-close.js` - HTTP server close functionality
- `test-fs-readfile-error.js` - FS readFile error handling
- `test-stream-pipe-cleanup.js` - Stream pipe cleanup

### Using Assertions

```javascript
const assert = require('assert');

// Strict equality
assert.strictEqual(actual, expected);
assert.notStrictEqual(actual, expected);

// Deep equality
assert.deepStrictEqual(obj1, obj2);

// Type checking
assert.ok(value);  // Truthy
assert.throws(() => { /* code */ }, TypeError);
assert.rejects(asyncFn(), Error);

// Regular expressions
assert.match(string, /pattern/);

// Error checking
assert.throws(() => {
  throw new Error('message');
}, {
  name: 'Error',
  message: 'message'
});
```

### Test Patterns

#### 1. Testing Async Operations

**Callback Pattern**:
```javascript
const server = http.createServer(common.mustCall((req, res) => {
  // This callback MUST be called
  res.end();
}));
```

**Promise Pattern**:
```javascript
async function test() {
  const result = await asyncOperation();
  assert.strictEqual(result, expected);
}

test().then(common.mustCall());
```

**Event Pattern**:
```javascript
stream.on('data', common.mustCall((chunk) => {
  // Verify chunk
}, expectedCallCount));

stream.on('end', common.mustCall());
```

#### 2. Testing Errors

```javascript
// Synchronous error
assert.throws(() => {
  fs.readFileSync('/nonexistent');
}, {
  code: 'ENOENT',
  name: 'Error'
});

// Async error (callback)
fs.readFile('/nonexistent', common.mustCall((err, data) => {
  assert(err);
  assert.strictEqual(err.code, 'ENOENT');
}));

// Async error (promise)
assert.rejects(
  fs.promises.readFile('/nonexistent'),
  { code: 'ENOENT' }
).then(common.mustCall());
```

#### 3. Testing Timers

```javascript
// setTimeout
setTimeout(common.mustCall(() => {
  // Must be called
}), 10);

// setImmediate
setImmediate(common.mustCall(() => {
  // Must be called
}));

// setInterval
const interval = setInterval(common.mustCall(() => {
  clearInterval(interval);
}, 3), 10);  // Called exactly 3 times
```

#### 4. Testing Events

```javascript
const emitter = new EventEmitter();

emitter.on('event', common.mustCall((arg) => {
  assert.strictEqual(arg, 'value');
}, 2));  // Must be called exactly 2 times

emitter.emit('event', 'value');
emitter.emit('event', 'value');
```

#### 5. Testing Streams

```javascript
const readable = new Readable({
  read() {
    this.push('data');
    this.push(null);  // End
  }
});

const writable = new Writable({
  write(chunk, encoding, callback) {
    assert.strictEqual(chunk.toString(), 'data');
    callback();
  }
});

readable.pipe(writable);

writable.on('finish', common.mustCall());
```

---

## Writing C++ Tests

### Setup

**Location**: `test/cctest/`

**Framework**: Google Test (gtest)

### Basic Structure

```cpp
#include "gtest/gtest.h"
#include "node.h"
#include "node_buffer.h"
#include "v8.h"

namespace {

using node::Buffer;
using v8::Isolate;
using v8::Local;
using v8::Value;

// Test fixture (optional)
class BufferTest : public ::testing::Test {
 protected:
  void SetUp() override {
    // Setup code
  }

  void TearDown() override {
    // Cleanup code
  }
};

// Simple test
TEST(Buffer, Creation) {
  // Test buffer creation
  const size_t size = 1024;
  char* data = new char[size];

  EXPECT_NE(data, nullptr);
  EXPECT_EQ(size, 1024);

  delete[] data;
}

// Test using fixture
TEST_F(BufferTest, Length) {
  // Test using BufferTest fixture
  // Can access protected members
}

}  // namespace
```

### Assertions

```cpp
// Boolean
EXPECT_TRUE(condition);
EXPECT_FALSE(condition);

// Equality
EXPECT_EQ(expected, actual);
EXPECT_NE(val1, val2);

// Comparison
EXPECT_LT(val1, val2);  // Less than
EXPECT_LE(val1, val2);  // Less or equal
EXPECT_GT(val1, val2);  // Greater than
EXPECT_GE(val1, val2);  // Greater or equal

// Strings
EXPECT_STREQ("str1", "str2");
EXPECT_STRNE("str1", "str2");

// Pointers
EXPECT_EQ(nullptr, ptr);
EXPECT_NE(nullptr, ptr);

// Floating point
EXPECT_FLOAT_EQ(expected, actual);
EXPECT_DOUBLE_EQ(expected, actual);

// Exceptions
EXPECT_THROW(statement, exception_type);
EXPECT_NO_THROW(statement);

// Fatal assertions (ASSERT_* instead of EXPECT_*)
ASSERT_EQ(expected, actual);  // Stops test on failure
```

### Running C++ Tests

```bash
# Build and run C++ tests
make cctest

# Run specific test
./out/Release/cctest --gtest_filter=BufferTest.Creation

# List all tests
./out/Release/cctest --gtest_list_tests
```

---

## Running Tests

### Basic Commands

```bash
# All tests (long!)
make test

# Parallel tests only (faster)
make test-parallel

# Sequential tests
make test-sequential

# C++ tests
make cctest

# Specific test file
node test/parallel/test-http-server.js

# Multiple test files
node test/parallel/test-http-*.js
```

### Advanced Testing

```bash
# Run tests with specific Node.js binary
./node test/parallel/test-http-server.js

# Run with options
./node --expose-internals test/parallel/test-internal-*.js

# Run with inspector
./node --inspect-brk test/parallel/test-http-server.js

# Run tests matching pattern
make test-parallel TESTS="test-http-*"

# Run with coverage
make coverage

# Run with valgrind (memory leak detection)
make test-valgrind

# Run benchmarks
make benchmark
```

### Test Runner Options

**Using `tools/test.py`**:
```bash
# Run specific tests
python tools/test.py parallel/test-http-*

# Run with specific jobs
python tools/test.py -J

# Run in specific mode
python tools/test.py --mode=debug

# Rerun failed tests
python tools/test.py --rerun-failed

# Verbose output
python tools/test.py --verbose
```

### Platform-Specific Testing

**Unix/Linux/macOS**:
```bash
make test
make test-parallel
make test-ci
```

**Windows**:
```cmd
vcbuild.bat test
vcbuild.bat test-parallel
vcbuild.bat test-ci
```

---

## Test Utilities

### Common Test Utilities (`test/common/`)

**Location**: `test/common/index.js`

#### Must-Call Utilities

**`common.mustCall()`**:
```javascript
// Callback must be called exactly once
function test() {
  setTimeout(common.mustCall(() => {
    // This MUST be called
  }), 10);
}

// Callback must be called exactly N times
stream.on('data', common.mustCall((chunk) => {
  // Process chunk
}, 3));  // Must be called 3 times
```

**`common.mustNotCall()`**:
```javascript
// Callback must NOT be called
server.on('error', common.mustNotCall());
```

**`common.mustSucceed()`**:
```javascript
// Callback for (err, result) pattern, err must be null
fs.readFile('/file', common.mustSucceed((data) => {
  // Process data, no error expected
}));
```

#### Skip Utilities

```javascript
// Skip if crypto not available
if (!common.hasCrypto) {
  common.skip('missing crypto');
}

// Skip if IPv6 not available
if (!common.hasIPv6) {
  common.skip('missing IPv6');
}

// Skip on specific platform
if (common.isWindows) {
  common.skip('not supported on Windows');
}

// Skip if running as root
if (common.isRoot) {
  common.skip('cannot run as root');
}
```

#### Platform Detection

```javascript
common.isWindows     // true on Windows
common.isLinux       // true on Linux
common.isMacOS       // true on macOS
common.isFreeBSD     // true on FreeBSD
common.isSunOS       // true on Solaris
common.isAIX         // true on AIX
```

#### Port Utilities

```javascript
// Get random port for testing
const port = common.PORT;  // Random port > 1024

// Multiple ports
const ports = common.getPorts(3);  // Array of 3 random ports
```

#### Resource Management

```javascript
// Temporary directory
const tmpdir = require('../common/tmpdir');
tmpdir.refresh();  // Clean and recreate temp dir

// Fixtures
const fixtures = require('../common/fixtures');
const certPath = fixtures.path('keys', 'agent1-cert.pem');
const cert = fixtures.readSync('keys/agent1-cert.pem');
```

### Fixtures (`test/fixtures/`)

**Purpose**: Shared test data and resources

**Common Fixtures**:
- `keys/` - SSL/TLS certificates
- `snapshot/` - Snapshot test data
- `wpt/` - Web Platform Tests
- Sample files for various tests

**Usage**:
```javascript
const fixtures = require('../common/fixtures');

// Read file
const data = fixtures.readSync('sample.txt');

// Get path
const path = fixtures.path('keys', 'agent1-cert.pem');
```

### Temporary Directory (`test/common/tmpdir.js`)

```javascript
const tmpdir = require('../common/tmpdir');

// Clean and recreate temp directory
tmpdir.refresh();

// Get temp directory path
const dir = tmpdir.path;

// Create file in temp dir
const filePath = path.join(tmpdir.path, 'test-file.txt');
```

### Snapshot Testing (`test/common/assertSnapshot.js`)

```javascript
const { spawnSyncAndAssert } = require('../common/assertSnapshot');

// Compare output to snapshot
spawnSyncAndAssert(
  process.execPath,
  ['script.js'],
  {
    stdout: 'expected output',
    stderr: ''
  }
);
```

---

## Debugging Tests

### Print Debugging

```javascript
// Console output
console.log('Debug:', value);
console.error('Error:', err);

// Inspect objects
const util = require('util');
console.log(util.inspect(obj, { depth: null }));
```

### Using Debugger

**Chrome DevTools**:
```bash
# Run with inspector
node --inspect-brk test/parallel/test-http-server.js

# Open chrome://inspect in Chrome
# Click "inspect"
```

**Node.js Debugger**:
```bash
node inspect test/parallel/test-http-server.js

# Commands:
# cont (c) - Continue
# next (n) - Next line
# step (s) - Step into
# out (o) - Step out
# repl - Enter REPL
```

**GDB** (for C++ debugging):
```bash
# Build debug version
./configure --debug
make -j4

# Debug test
gdb --args ./node test/parallel/test-http-server.js

# GDB commands
(gdb) break node::Start
(gdb) run
(gdb) backtrace
(gdb) print variable
```

### Test Verbosity

```bash
# Run with verbose output
NODE_DEBUG=http node test/parallel/test-http-server.js

# Multiple modules
NODE_DEBUG=http,net node test/parallel/test-http-server.js

# All debug output
NODE_DEBUG=* node test/parallel/test-http-server.js
```

---

## CI/CD Testing

### GitHub Actions

**Location**: `.github/workflows/`

**Workflows**:
- `test-linux.yml` - Linux testing
- `test-macos.yml` - macOS testing
- `test-windows.yml` - Windows testing
- `coverage.yml` - Code coverage
- `linters.yml` - Linting

### CI Test Targets

```bash
# Run CI-style tests locally
make test-ci

# Coverage testing
make coverage

# Linting
make lint
```

### Platform Matrix

CI tests run on:
- **Linux**: Ubuntu (multiple versions)
- **macOS**: macOS 12, 13, 14
- **Windows**: Windows Server 2019, 2022
- **Architectures**: x64, ARM64, x86
- **Compilers**: GCC, Clang, MSVC

---

## Best Practices

### 1. Test Independence

✅ **Good**:
```javascript
{
  const server = http.createServer();
  server.listen(0);  // Random port
  // ... test ...
  server.close();
}
```

❌ **Bad**:
```javascript
const server = http.createServer();
server.listen(3000);  // Fixed port, conflicts with other tests
```

### 2. Resource Cleanup

✅ **Good**:
```javascript
const server = http.createServer();
server.listen(0, common.mustCall(() => {
  // ... test ...
  server.close();  // Always close
}));
```

❌ **Bad**:
```javascript
const server = http.createServer();
server.listen(0);
// Forgot to close, leaks resources
```

### 3. Assertions

✅ **Good**:
```javascript
assert.strictEqual(actual, expected);
assert.deepStrictEqual(obj1, obj2);
```

❌ **Bad**:
```javascript
assert.equal(actual, expected);  // Use strictEqual
assert.deepEqual(obj1, obj2);    // Use deepStrictEqual
```

### 4. Must-Call Usage

✅ **Good**:
```javascript
server.on('connection', common.mustCall((socket) => {
  // Ensures callback is called
}));
```

❌ **Bad**:
```javascript
server.on('connection', (socket) => {
  // May not be called, test passes incorrectly
});
```

### 5. Error Testing

✅ **Good**:
```javascript
assert.throws(() => {
  JSON.parse('invalid');
}, SyntaxError);
```

❌ **Bad**:
```javascript
try {
  JSON.parse('invalid');
} catch (e) {
  // Catches error but doesn't verify type
}
```

---

## Common Test Patterns

### HTTP Server Testing

```javascript
const server = http.createServer(common.mustCall((req, res) => {
  assert.strictEqual(req.method, 'GET');
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('OK');
}));

server.listen(0, common.mustCall(() => {
  const port = server.address().port;

  http.get(`http://localhost:${port}`, common.mustCall((res) => {
    assert.strictEqual(res.statusCode, 200);

    let data = '';
    res.on('data', (chunk) => { data += chunk; });
    res.on('end', common.mustCall(() => {
      assert.strictEqual(data, 'OK');
      server.close();
    }));
  }));
}));
```

### Stream Testing

```javascript
const readable = new Readable({
  read() {
    this.push('hello');
    this.push('world');
    this.push(null);
  }
});

let data = '';
readable.on('data', (chunk) => {
  data += chunk;
});

readable.on('end', common.mustCall(() => {
  assert.strictEqual(data, 'helloworld');
}));
```

### Promise Testing

```javascript
async function test() {
  const data = await fs.promises.readFile('/path');
  assert.strictEqual(data.toString(), 'expected');
}

test().then(common.mustCall());
```

### Child Process Testing

```javascript
const child = spawn(process.execPath, ['-e', 'console.log("hello")']);

let stdout = '';
child.stdout.on('data', (chunk) => {
  stdout += chunk;
});

child.on('close', common.mustCall((code) => {
  assert.strictEqual(code, 0);
  assert.strictEqual(stdout.trim(), 'hello');
}));
```

---

## Quick Reference

### Common Commands

| Task | Command |
|------|---------|
| All tests | `make test` |
| Fast test | `make test-parallel` |
| C++ tests | `make cctest` |
| Single test | `node test/parallel/test-name.js` |
| Coverage | `make coverage` |
| Lint | `make lint` |
| CI tests | `make test-ci` |

### Must-Call Examples

```javascript
// Called exactly once
common.mustCall(fn)

// Called exactly N times
common.mustCall(fn, N)

// Must not be called
common.mustNotCall()

// (err, result) pattern, err must be null
common.mustSucceed(fn)
```

---

## Additional Resources

- **Test README**: `test/README.md`
- **Common Utilities**: `test/common/index.js`
- **Writing Tests**: `doc/contributing/writing-tests.md`
- **Style Guide**: Follow contribution guidelines

---

**Last Updated**: 2025-11-19
**For More**: See `CONTRIBUTION_GUIDE.md` for full contribution process
