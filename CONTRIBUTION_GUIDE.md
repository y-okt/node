# Node.js Contribution Guide

> A comprehensive guide for making OSS contributions to Node.js

## Table of Contents
- [Getting Started](#getting-started)
- [Contribution Workflow](#contribution-workflow)
- [Commit Message Guidelines](#commit-message-guidelines)
- [Code Review Process](#code-review-process)
- [Common Subsystems](#common-subsystems)
- [Code Style Guidelines](#code-style-guidelines)
- [Testing Requirements](#testing-requirements)
- [Landing Pull Requests](#landing-pull-requests)
- [Governance Structure](#governance-structure)
- [Common Contribution Areas](#common-contribution-areas)

---

## Getting Started

### Prerequisites

1. **GitHub Account**: Create a GitHub account if you don't have one
2. **Git**: Install Git on your local machine
3. **Build Tools**: See `BUILD_SYSTEM_GUIDE.md` for platform-specific requirements
4. **Time**: Allow 48+ hours minimum for non-trivial PR reviews

### Initial Setup

```bash
# 1. Fork the repository on GitHub
# Go to https://github.com/nodejs/node and click "Fork"

# 2. Clone your fork
git clone https://github.com/YOUR-USERNAME/node.git
cd node

# 3. Add upstream remote
git remote add upstream https://github.com/nodejs/node.git

# 4. Verify remotes
git remote -v
# origin    https://github.com/YOUR-USERNAME/node.git (fetch)
# origin    https://github.com/YOUR-USERNAME/node.git (push)
# upstream  https://github.com/nodejs/node.git (fetch)
# upstream  https://github.com/nodejs/node.git (push)

# 5. Build Node.js (see BUILD_SYSTEM_GUIDE.md)
./configure
make -j4

# 6. Run tests to verify setup
make test-parallel
```

### First Contribution Ideas

**Good First Issues**:
- Documentation improvements
- Test coverage additions
- Code comments and clarifications
- Small bug fixes with clear reproduction steps

**Where to Find Issues**:
- GitHub label: `good first issue`
- GitHub label: `help wanted`
- Documentation gaps
- Test coverage improvements

---

## Contribution Workflow

### Complete Workflow Overview

```
┌─────────────────────────────────────────────────────────┐
│ 1. IDENTIFY ISSUE                                       │
│    - Browse GitHub issues                              │
│    - Find bug or propose feature                       │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 2. DISCUSS (for non-trivial changes)                   │
│    - Comment on existing issue                         │
│    - Open new issue for discussion                     │
│    - Get feedback before coding                        │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 3. CREATE BRANCH                                        │
│    git fetch upstream                                  │
│    git checkout -b my-feature upstream/main            │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 4. MAKE CHANGES                                         │
│    - Write code                                        │
│    - Add tests                                         │
│    - Update documentation                              │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 5. TEST LOCALLY                                         │
│    make lint              # Run linters                │
│    make test              # Run tests                  │
│    make test-ci           # CI-like testing            │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 6. COMMIT CHANGES                                       │
│    git add <files>                                     │
│    git commit -m "subsystem: description"              │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 7. PUSH TO FORK                                         │
│    git push origin my-feature                          │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 8. CREATE PULL REQUEST                                  │
│    - Go to GitHub                                      │
│    - Click "New pull request"                          │
│    - Fill out PR template                              │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 9. CODE REVIEW (48+ hours minimum)                     │
│    - Respond to feedback                               │
│    - Make requested changes                            │
│    - Update PR                                         │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 10. CI TESTING                                          │
│    - All platforms must pass                           │
│    - Fix any CI failures                               │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 11. APPROVAL                                            │
│    - Require 2 collaborator approvals                  │
│    - OR 1 approval if open 7+ days                     │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────┴────────────────────────────────────────┐
│ 12. LANDING                                             │
│    - Collaborator lands PR                             │
│    - Commits added with metadata                       │
│    - Branch merged to main                             │
└─────────────────────────────────────────────────────────┘
```

### Detailed Steps

#### Step 1: Stay Updated

```bash
# Fetch latest changes from upstream
git fetch upstream

# Update your main branch
git checkout main
git merge --ff-only upstream/main
git push origin main
```

#### Step 2: Create Feature Branch

```bash
# Create branch from upstream/main
git checkout -b fix-http-issue upstream/main

# Make changes...
```

#### Step 3: Make Changes

**Code Changes**:
- Follow code style guidelines (see below)
- Keep changes focused and atomic
- Don't mix unrelated changes

**Test Changes**:
- Add tests for new functionality
- Add regression tests for bug fixes
- Ensure tests pass locally

**Documentation Changes**:
- Update API docs if changing public APIs
- Update comments for clarity
- Update CHANGELOG if applicable

#### Step 4: Local Testing

```bash
# Run linters (REQUIRED)
make lint
make lint-md        # Markdown linting
make lint-cpp       # C++ linting

# Run tests
make test-parallel  # Fast parallel tests
make test           # Full test suite
make test-ci        # CI-style testing

# Test specific subsystem
node test/parallel/test-http-*.js
```

#### Step 5: Commit Changes

See [Commit Message Guidelines](#commit-message-guidelines) below

```bash
# Stage changes
git add src/node_http.cc test/parallel/test-http-fix.js

# Commit with proper message
git commit -m "http: fix memory leak in server close

When closing an HTTP server, connection handles were not
properly released, causing a memory leak over time.

This commit ensures all handles are properly cleaned up
during server shutdown.

Fixes: https://github.com/nodejs/node/issues/12345
Refs: https://github.com/nodejs/node/pull/12340"
```

#### Step 6: Push to Your Fork

```bash
git push origin fix-http-issue
```

#### Step 7: Create Pull Request

1. Go to https://github.com/YOUR-USERNAME/node
2. Click "New pull request"
3. Select your branch
4. Fill out PR template:
   - **Title**: Use commit message format (subsystem: description)
   - **Description**: Explain what, why, and how
   - **Checklist**: Complete all items
   - **Related Issues**: Link to issues

**PR Template Example**:
```markdown
### Description

This PR fixes a memory leak in HTTP server shutdown.

### What

- Modified `src/node_http.cc` to properly release connection handles
- Added regression test in `test/parallel/test-http-server-close.js`

### Why

When closing an HTTP server, connection handles were not properly
released, causing memory to grow unbounded in long-running servers.

### How

- Added handle cleanup in `Server::Close()`
- Ensured all pending connections are drained before shutdown

### Checklist

- [x] `make -j4 test` (UNIX) or `vcbuild test` (Windows) passes
- [x] tests and/or benchmarks are included
- [x] documentation is changed or added
- [x] commit message follows commit guidelines

### Related Issues

Fixes: https://github.com/nodejs/node/issues/12345
Refs: https://github.com/nodejs/node/pull/12340
```

---

## Commit Message Guidelines

### Format

```
<subsystem>: <subject>

<body>

<footer>
```

### Components

#### 1. Subsystem Prefix (REQUIRED)

Identifies which part of Node.js the change affects.

**Common Subsystems**:

| Subsystem | Description | Example |
|-----------|-------------|---------|
| `lib` | Generic JavaScript changes | `lib: improve error messages` |
| `src` | Generic C++ changes | `src: refactor buffer allocation` |
| `http` | HTTP module | `http: fix keep-alive timeout` |
| `fs` | File system | `fs: add recursive mkdir option` |
| `stream` | Streams | `stream: improve backpressure handling` |
| `net` | Networking | `net: fix socket timeout` |
| `crypto` | Cryptography | `crypto: update OpenSSL version` |
| `buffer` | Buffer | `buffer: optimize toString()` |
| `child_process` | Child processes | `child_process: fix stdio handling` |
| `cluster` | Cluster module | `cluster: improve worker management` |
| `dns` | DNS | `dns: add EDNS support` |
| `events` | Events | `events: optimize listener management` |
| `test` | Tests | `test: add crypto hash tests` |
| `doc` | Documentation | `doc: clarify stream.pipe() behavior` |
| `build` | Build system | `build: update GYP configuration` |
| `tools` | Tools and scripts | `tools: update ESLint to 8.x` |
| `deps` | Dependencies | `deps: update V8 to 11.2` |
| `worker` | Worker threads | `worker: fix message passing` |
| `async_hooks` | Async hooks | `async_hooks: fix resource tracking` |
| `perf_hooks` | Performance hooks | `perf_hooks: add new metrics` |
| `inspector` | Inspector/debugger | `inspector: improve protocol support` |
| `esm` | ES modules | `esm: fix loader resolution` |
| `repl` | REPL | `repl: improve autocomplete` |

**Multiple Subsystems**:
- Use comma separation: `http,https: fix timeout handling`
- Use parent subsystem: `lib: update multiple modules`

#### 2. Subject Line (REQUIRED)

- **Max 72 characters**
- **Imperative mood**: "fix" not "fixes" or "fixed"
- **Lowercase**: "fix bug" not "Fix bug"
- **No period at end**
- **Descriptive**: Explain what the commit does

**Good Examples**:
```
http: fix memory leak in server close
doc: clarify stream error handling
test: add regression test for buffer overflow
deps: update V8 to version 11.2.1
```

**Bad Examples**:
```
Fix bug                          # Too vague
http: Fixes memory leak.         # Wrong mood, has period
Updated documentation            # Missing subsystem, wrong mood
http: this commit fixes a bug    # Not imperative
```

#### 3. Body (RECOMMENDED)

- **Blank line** after subject
- **Wrap at 72 characters**
- **Explain what and why**, not how (code shows how)
- **Multiple paragraphs** OK

**Example**:
```
http: fix memory leak in server close

When closing an HTTP server, connection handles were not
properly released, causing a memory leak over time. This
was particularly problematic in long-running servers that
frequently restart connections.

This commit ensures all handles are properly cleaned up
during server shutdown by iterating through pending
connections and explicitly releasing resources.

The fix maintains backward compatibility while preventing
the memory leak observed in production environments.
```

#### 4. Footer (CONDITIONAL)

**Issue References**:
```
Fixes: https://github.com/nodejs/node/issues/12345
Refs: https://github.com/nodejs/node/pull/12340
Refs: https://github.com/nodejs/node/issues/11111
```

**Breaking Changes**:
```
BREAKING CHANGE: Remove deprecated API

The `util.pump()` function has been removed. Users should
migrate to `stream.pipe()` instead.
```

**Semantic Versioning Labels**:
```
Semver-Major: Breaking change
Semver-Minor: New feature
Semver-Patch: Bug fix
```

**Backport Requests**:
```
Backport-Requested-By: @username
```

### Complete Example

```
http: fix memory leak in server connection handling

When an HTTP server closed active connections, the underlying
TCP handles were not properly released. This caused memory
usage to grow over time, particularly in long-running servers
with high connection churn.

This commit modifies the server close logic to:
- Explicitly close all pending connection handles
- Release associated resources before shutdown
- Add proper error handling during cleanup

The fix is backward compatible and adds negligible performance
overhead. A regression test is included to prevent recurrence.

Fixes: https://github.com/nodejs/node/issues/12345
Refs: https://github.com/nodejs/node/pull/12340
```

### Commit Message Checklist

- [ ] Subsystem prefix included
- [ ] Subject in imperative mood
- [ ] Subject lowercase (except proper nouns)
- [ ] Subject max 72 characters
- [ ] Subject no period at end
- [ ] Blank line after subject
- [ ] Body wraps at 72 characters
- [ ] Body explains what and why
- [ ] Footer includes issue links if applicable
- [ ] Breaking changes clearly marked

---

## Code Review Process

### Timeline

- **Minimum 48 hours** for non-trivial changes
- **Longer for complex/controversial changes**
- **Weekends don't count** toward wait time
- **Urgent fixes** may be expedited with TSC approval

### Review Requirements

**Approval Thresholds**:
- **2 collaborator approvals** (standard)
- **1 collaborator approval** if PR open 7+ days
- **Additional reviews** for semver-major changes
- **TSC approval** for governance/policy changes

### Review Feedback

**Types of Feedback**:
1. **Approve**: Ready to land
2. **Request Changes**: Must address before landing
3. **Comment**: Suggestions, questions, discussion

**Responding to Feedback**:
```bash
# Make requested changes
# ... edit files ...

# Commit changes
git add <files>
git commit -m "address review feedback"

# Or amend if fixing issues in original commit
git add <files>
git commit --amend --no-edit

# Push to update PR
git push origin fix-http-issue --force-with-lease
```

### Common Review Comments

**Style Issues**:
- "Please run `make lint`" → Linting failures
- "Wrap at 72 characters" → Commit message too long
- "Use imperative mood" → Commit subject incorrect

**Technical Issues**:
- "Missing test coverage" → Add tests
- "Memory leak potential" → Fix resource handling
- "Breaking change" → Needs semver-major label
- "Performance concern" → Add benchmarks

**Process Issues**:
- "Missing Fixes: line" → Add issue reference
- "Needs API docs" → Update documentation
- "Requires backport discussion" → Discuss with releasers

---

## Common Subsystems

### HTTP (`lib/_http_*.js`, `src/node_http*.cc`)

**Scope**: HTTP/1.1 server and client implementation

**Common Changes**:
- Performance improvements
- Bug fixes in header parsing
- Keep-alive handling
- Error handling improvements

**Key Files**:
- `lib/_http_server.js` - Server implementation
- `lib/_http_client.js` - Client implementation
- `lib/_http_common.js` - Shared utilities
- `src/node_http_parser.cc` - HTTP parsing

### Streams (`lib/_stream_*.js`)

**Scope**: Stream abstraction (Readable, Writable, Duplex, Transform)

**Common Changes**:
- Backpressure improvements
- Error handling
- Performance optimizations
- Pipeline enhancements

**Key Files**:
- `lib/_stream_readable.js`
- `lib/_stream_writable.js`
- `lib/_stream_duplex.js`
- `lib/_stream_transform.js`

### File System (`lib/fs.js`, `src/node_file.cc`)

**Scope**: File system operations

**Common Changes**:
- New file system features
- Error handling improvements
- Performance optimizations
- Platform compatibility

**Key Files**:
- `lib/fs.js` - JavaScript API
- `src/node_file.cc` - C++ implementation
- `src/fs_event_wrap.cc` - File watching

### Crypto (`lib/crypto.js`, `src/crypto/`)

**Scope**: Cryptographic operations

**Common Changes**:
- Algorithm additions
- OpenSSL updates
- Security fixes
- Performance improvements

**Key Files**:
- `lib/crypto.js` - JavaScript API
- `src/crypto/` - C++ implementations

---

## Code Style Guidelines

### JavaScript Style

**Linting**: Enforced by ESLint

```bash
# Run JavaScript linter
make lint-js

# Auto-fix simple issues
make lint-js-fix
```

**Key Rules**:
- **2 spaces** for indentation
- **Single quotes** for strings
- **Semicolons** required
- **120 character** line limit
- **Strict mode** in all files
- **const/let** over var

**Example**:
```javascript
'use strict';

const http = require('http');
const { EventEmitter } = require('events');

function createServer(options) {
  if (typeof options !== 'object') {
    throw new TypeError('options must be an object');
  }

  const server = new http.Server(options);
  return server;
}

module.exports = { createServer };
```

### C++ Style

**Linting**: Enforced by cpplint

```bash
# Run C++ linter
make lint-cpp
```

**Key Rules**:
- **2 spaces** for indentation
- **80 character** line limit (soft limit)
- **Google C++ Style Guide** with Node.js modifications
- **RAII** for resource management
- **Namespaces**: Avoid `using namespace`

**Naming Conventions**:
```cpp
// Classes: PascalCase
class TCPWrap : public HandleWrap {

  // Methods: PascalCase
  void Start();
  void Stop();

  // Private members: snake_case with trailing underscore
 private:
  uv_tcp_t* handle_;
  int connection_count_;
};

// Functions: PascalCase
void Initialize(Local<Object> target);

// Constants: kPascalCase
const int kMaxConnections = 1000;
```

**Example**:
```cpp
#include "tcp_wrap.h"
#include "env-inl.h"
#include "node.h"

namespace node {

using v8::Context;
using v8::Local;
using v8::Object;

TCPWrap::TCPWrap(Environment* env, Local<Object> object)
    : HandleWrap(env, object),
      handle_(nullptr) {
  // Initialization
}

void TCPWrap::Initialize(Local<Object> target) {
  Environment* env = Environment::GetCurrent(target);

  // Setup prototype
  Local<FunctionTemplate> t = env->NewFunctionTemplate(New);
  t->InstanceTemplate()->SetInternalFieldCount(1);
  t->SetClassName(FIXED_ONE_BYTE_STRING(env->isolate(), "TCP"));

  // Export
  target->Set(env->context(),
              FIXED_ONE_BYTE_STRING(env->isolate(), "TCP"),
              t->GetFunction(env->context()).ToLocalChecked()).Check();
}

}  // namespace node
```

### Documentation Style

**API Docs**: Markdown files in `doc/api/`

```bash
# Run markdown linter
make lint-md
```

**Format**:
```markdown
## `class: Server`

Extends: {EventEmitter}

This class is used to create a TCP or IPC server.

### `server.listen(port[, host][, backlog][, callback])`

* `port` {number} Port number
* `host` {string} Host name. **Default:** `'0.0.0.0'`
* `backlog` {number} Queue size. **Default:** 511
* `callback` {Function} Callback function
* Returns: {Server}

Start listening for connections.
```

---

## Testing Requirements

### Test Categories

**Always Add Tests** for:
- New features
- Bug fixes (regression tests)
- API changes
- Edge cases

**Test Location**:
- `test/parallel/` - Most tests (can run in parallel)
- `test/sequential/` - Tests requiring serialization
- `test/cctest/` - C++ unit tests

### Writing Tests

**JavaScript Tests**:
```javascript
'use strict';

const common = require('../common');
const assert = require('assert');
const http = require('http');

// Test HTTP server close
{
  const server = http.createServer(common.mustCall((req, res) => {
    res.end('hello');
  }));

  server.listen(0, common.mustCall(() => {
    const port = server.address().port;

    http.get(`http://localhost:${port}`, common.mustCall((res) => {
      assert.strictEqual(res.statusCode, 200);
      res.resume();
      server.close(common.mustCall());
    }));
  }));
}
```

**C++ Tests**:
```cpp
#include "gtest/gtest.h"
#include "node_buffer.h"

TEST(BufferTest, Creation) {
  // Test buffer creation
  const size_t len = 1024;
  char* data = node::Buffer::Data(buffer);

  EXPECT_NE(data, nullptr);
  EXPECT_EQ(node::Buffer::Length(buffer), len);
}
```

### Running Tests

```bash
# Specific test
node test/parallel/test-http-server.js

# All parallel tests
make test-parallel

# All tests
make test

# C++ tests only
make cctest

# With coverage
make coverage
```

---

## Landing Pull Requests

**Only collaborators can land PRs**

### Landing Process

1. **Verify Requirements**:
   - [ ] 48+ hours since PR opened (or approved by TSC for urgent)
   - [ ] 2 approvals (or 1 if open 7+ days)
   - [ ] CI is green on all platforms
   - [ ] No outstanding "Request Changes" reviews
   - [ ] Commits follow commit guidelines

2. **Use `git node land` Tool**:
```bash
# Install node-core-utils
npm install -g node-core-utils

# Configure GitHub token
ncu-config set github_token <token>

# Land PR
git node land <pr-number>
```

3. **Manual Landing** (if needed):
```bash
# Fetch PR
git fetch upstream pull/<pr-number>/head:pr-<pr-number>
git checkout pr-<pr-number>

# Rebase on main
git rebase upstream/main

# Add metadata
git commit --amend

# Push to main
git push upstream HEAD:main
```

### Commit Metadata

Automatically added by `git node land`:
```
subsystem: description

Body text explaining change.

PR-URL: https://github.com/nodejs/node/pull/12345
Reviewed-By: Alice Smith <alice@example.com>
Reviewed-By: Bob Jones <bob@example.com>
Fixes: https://github.com/nodejs/node/issues/11111
```

---

## Governance Structure

### Roles

**Technical Steering Committee (TSC)**:
- **17 voting members** + regular members
- Oversee technical direction
- Approve major changes
- Manage collaborator nominations

**Collaborators** (~150+):
- Commit access to repository
- Review and land pull requests
- Triage issues
- Guide contributors

**Triagers** (~50+):
- Triage new issues
- Label and categorize
- Close invalid issues
- No commit access

### Becoming a Collaborator

**Requirements**:
- Consistent, high-quality contributions
- Understanding of project processes
- Good standing in community
- Nominated by existing collaborator
- TSC approval

**Process**:
1. Existing collaborator nominates you
2. TSC discusses nomination
3. Vote (simple majority)
4. Onboarding process

---

## Common Contribution Areas

### Documentation

**What**: Improve API docs, guides, examples

**Where**:
- `doc/api/` - API documentation
- `doc/guides/` - Contributor guides
- Code comments
- README files

**Good for**: First-time contributors

### Tests

**What**: Add test coverage, improve test quality

**Where**:
- `test/parallel/` - Parallel tests
- `test/sequential/` - Sequential tests
- `test/cctest/` - C++ unit tests

**Good for**: Learning codebase, improving quality

### Bug Fixes

**What**: Fix reported bugs

**Where**: GitHub issues with `bug` label

**Good for**: Building experience, gaining trust

### Features

**What**: Add new functionality

**Where**: Discussed in GitHub issues first

**Good for**: Experienced contributors

### Performance

**What**: Optimize hot paths, reduce overhead

**Where**: Identified through benchmarking

**Good for**: Advanced contributors

### Security

**What**: Fix security vulnerabilities

**Where**: Report to security@nodejs.org first

**Good for**: Security researchers

---

## Additional Resources

- **Official Contributing Guide**: `CONTRIBUTING.md`
- **Collaborator Guide**: `doc/contributing/collaborator-guide.md`
- **Pull Request Guide**: `doc/contributing/pull-requests.md`
- **Issue Triage**: `doc/contributing/issues.md`
- **Onboarding**: `onboarding.md`
- **Code of Conduct**: `CODE_OF_CONDUCT.md`

---

**Last Updated**: 2025-11-19
**Questions?**: Open an issue or ask in #nodejs-dev on OpenJS Foundation Slack
