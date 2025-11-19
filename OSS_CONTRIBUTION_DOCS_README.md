# Node.js OSS Contribution Documentation

> Comprehensive guides for making open-source contributions to Node.js

## Welcome!

This collection of documents provides everything you need to understand Node.js architecture and start contributing to the project. Whether you're fixing a bug, adding a feature, or improving documentation, these guides will help you navigate the codebase effectively.

---

## 📚 Documentation Index

### 1. [Architecture Overview](./ARCHITECTURE_OVERVIEW.md)
**Start here to understand Node.js architecture**

**What you'll learn**:
- High-level architecture and layered design
- Core components (V8, libuv, OpenSSL, etc.)
- Repository structure and organization
- Key design patterns (BaseObject, AsyncWrap, etc.)
- Essential files and concepts

**Best for**: Understanding how Node.js works internally

---

### 2. [Contribution Guide](./CONTRIBUTION_GUIDE.md)
**Complete guide to the contribution process**

**What you'll learn**:
- Step-by-step contribution workflow
- Commit message guidelines and best practices
- Code review process and requirements
- Common subsystems and their conventions
- Code style guidelines (JavaScript and C++)
- Pull request requirements

**Best for**: Making your first (or any) contribution

---

### 3. [Build System & Development Setup](./BUILD_SYSTEM_GUIDE.md)
**Everything about building and configuring Node.js**

**What you'll learn**:
- Platform-specific setup (Linux, macOS, Windows)
- Build system architecture (GYP, Make, etc.)
- Configuration options and flags
- Building for development vs production
- Build optimization techniques
- Troubleshooting common build issues

**Best for**: Getting your development environment ready

---

### 4. [Core Subsystems Deep Dive](./CORE_SUBSYSTEMS_GUIDE.md)
**Detailed exploration of Node.js subsystems**

**What you'll learn**:
- V8 JavaScript engine integration
- libuv event loop architecture
- HTTP/HTTPS implementation
- File system operations
- Streams architecture
- Networking (TCP/UDP)
- Crypto & TLS
- Worker threads
- Module system (CommonJS & ES modules)
- Async hooks
- Inspector & debugging
- Permission system

**Best for**: Contributing to specific subsystems

---

### 5. [Testing Infrastructure](./TESTING_GUIDE.md)
**Comprehensive testing guide**

**What you'll learn**:
- Test directory structure and organization
- Writing JavaScript tests
- Writing C++ tests (using Google Test)
- Test utilities and common patterns
- Running and debugging tests
- CI/CD testing requirements
- Best practices and common patterns

**Best for**: Writing tests for your contributions

---

## 🚀 Quick Start Guide

### For New Contributors

1. **Read**: Start with [Architecture Overview](./ARCHITECTURE_OVERVIEW.md)
2. **Setup**: Follow [Build System Guide](./BUILD_SYSTEM_GUIDE.md) to build Node.js
3. **Learn**: Read [Contribution Guide](./CONTRIBUTION_GUIDE.md)
4. **Explore**: Pick a subsystem from [Core Subsystems Guide](./CORE_SUBSYSTEMS_GUIDE.md)
5. **Test**: Learn testing from [Testing Guide](./TESTING_GUIDE.md)
6. **Contribute**: Make your first PR!

### For Specific Tasks

**Fixing a Bug**:
1. Understand the subsystem → [Core Subsystems Guide](./CORE_SUBSYSTEMS_GUIDE.md)
2. Write tests → [Testing Guide](./TESTING_GUIDE.md)
3. Follow contribution process → [Contribution Guide](./CONTRIBUTION_GUIDE.md)

**Adding a Feature**:
1. Understand architecture → [Architecture Overview](./ARCHITECTURE_OVERVIEW.md)
2. Build and configure → [Build System Guide](./BUILD_SYSTEM_GUIDE.md)
3. Study relevant subsystem → [Core Subsystems Guide](./CORE_SUBSYSTEMS_GUIDE.md)
4. Write comprehensive tests → [Testing Guide](./TESTING_GUIDE.md)
5. Follow PR process → [Contribution Guide](./CONTRIBUTION_GUIDE.md)

**Improving Tests**:
1. Learn testing infrastructure → [Testing Guide](./TESTING_GUIDE.md)
2. Understand subsystem → [Core Subsystems Guide](./CORE_SUBSYSTEMS_GUIDE.md)
3. Submit PR → [Contribution Guide](./CONTRIBUTION_GUIDE.md)

---

## 🎯 Domain Knowledge Summary

### Key Technologies

**JavaScript Runtime**:
- **V8**: JavaScript engine from Google
- **libuv**: Cross-platform asynchronous I/O
- **llhttp**: HTTP parser
- **OpenSSL**: Cryptography and TLS

**Languages**:
- **JavaScript**: API layer (`lib/`)
- **C++**: Core implementation (`src/`)
- **Python**: Build scripts (`configure.py`)
- **GYP**: Build configuration

**Build System**:
- **GYP**: Cross-platform build file generator
- **Make**: Unix/Linux/macOS builds
- **MSBuild**: Windows builds
- **Ninja**: Fast build backend (optional)

### Architecture Layers

```
┌─────────────────────────────────────┐
│     JavaScript API (lib/)           │  User-facing APIs
├─────────────────────────────────────┤
│     C++ Bindings (src/api/)         │  Bridge layer
├─────────────────────────────────────┤
│     Core C++ (src/)                 │  Implementation
├─────────────────────────────────────┤
│     External Deps (deps/)           │  V8, libuv, etc.
└─────────────────────────────────────┘
```

### Important Concepts

1. **Isolates**: Independent V8 instances (one per thread)
2. **Event Loop**: libuv manages asynchronous operations
3. **Handles**: Long-lived resources (sockets, timers)
4. **Requests**: One-time operations (writes, connects)
5. **AsyncWrap**: Tracks async operations for hooks
6. **BaseObject**: Base class for native objects
7. **Environment**: Per-isolate state management

---

## 📖 Contribution Workflow Summary

1. **Fork & Clone**
   ```bash
   git clone https://github.com/YOUR-USERNAME/node.git
   cd node
   git remote add upstream https://github.com/nodejs/node.git
   ```

2. **Build**
   ```bash
   ./configure
   make -j$(nproc)
   ```

3. **Create Branch**
   ```bash
   git checkout -b my-feature upstream/main
   ```

4. **Make Changes**
   - Write code
   - Add tests
   - Update docs

5. **Test**
   ```bash
   make lint
   make test-parallel
   ```

6. **Commit**
   ```bash
   git commit -m "subsystem: description

   Detailed explanation of changes.

   Fixes: https://github.com/nodejs/node/issues/12345"
   ```

7. **Push & PR**
   ```bash
   git push origin my-feature
   # Create PR on GitHub
   ```

8. **Review**
   - Respond to feedback
   - Update PR as needed
   - Wait for approvals (minimum 48 hours)

9. **Landing**
   - Collaborator lands PR
   - Your contribution is merged!

---

## 🔑 Key Directories

| Directory | Purpose |
|-----------|---------|
| `src/` | C++ core implementation (5.8MB) |
| `lib/` | JavaScript modules (4.2MB) |
| `deps/` | External dependencies (539MB) |
| `test/` | Test suites (59MB, 2000+ files) |
| `doc/` | Documentation (17MB) |
| `tools/` | Build and development tools (5.5MB) |

---

## 💡 Common Contribution Areas

**Good for Beginners**:
- Documentation improvements
- Test coverage additions
- Code comments
- Small bug fixes

**Intermediate**:
- Bug fixes in subsystems
- Performance improvements
- Error handling enhancements
- API additions

**Advanced**:
- Core subsystem changes
- V8/libuv integration
- Security fixes
- Architecture improvements

---

## 🤝 Getting Help

- **Issues**: Browse [GitHub issues](https://github.com/nodejs/node/issues)
- **Labels**:
  - `good first issue` - Beginner-friendly
  - `help wanted` - Community help needed
- **Discussion**: Open an issue for questions
- **IRC/Slack**: OpenJS Foundation Slack (#nodejs-dev)
- **Documentation**: `doc/contributing/` directory

---

## 📋 Pre-Contribution Checklist

Before submitting a PR, ensure:

- [ ] Read relevant documentation from this collection
- [ ] Built Node.js successfully
- [ ] Ran tests locally (`make lint` and `make test`)
- [ ] Followed commit message guidelines
- [ ] Added tests for changes
- [ ] Updated documentation if needed
- [ ] PR description is complete and clear

---

## 📚 Additional Official Resources

**In Repository**:
- `CONTRIBUTING.md` - Basic contribution guide
- `BUILDING.md` - Build instructions
- `GOVERNANCE.md` - Project governance
- `CODE_OF_CONDUCT.md` - Community guidelines
- `doc/contributing/` - 45+ detailed guides

**External**:
- [Node.js Website](https://nodejs.org/)
- [Node.js Documentation](https://nodejs.org/docs/)
- [OpenJS Foundation](https://openjsf.org/)

---

## 🎓 Learning Path

### Week 1: Foundation
- Read [Architecture Overview](./ARCHITECTURE_OVERVIEW.md)
- Build Node.js using [Build System Guide](./BUILD_SYSTEM_GUIDE.md)
- Run tests using [Testing Guide](./TESTING_GUIDE.md)

### Week 2: Deep Dive
- Study one subsystem from [Core Subsystems Guide](./CORE_SUBSYSTEMS_GUIDE.md)
- Read related code in `lib/` and `src/`
- Run related tests

### Week 3: Practice
- Find a "good first issue"
- Follow [Contribution Guide](./CONTRIBUTION_GUIDE.md)
- Make your first PR

### Week 4+: Regular Contributor
- Engage with community
- Help review PRs
- Take on larger contributions

---

## 🏆 Success Tips

1. **Start Small**: Documentation and tests are great first contributions
2. **Ask Questions**: Community is helpful, don't hesitate to ask
3. **Be Patient**: PR review takes time (minimum 48 hours)
4. **Read Code**: Best way to learn is reading existing implementations
5. **Test Thoroughly**: Always add tests for your changes
6. **Follow Guidelines**: Commit messages and code style matter
7. **Stay Updated**: Pull from upstream frequently
8. **Have Fun**: Contributing to Node.js is rewarding!

---

## 📝 Document Updates

**Last Updated**: 2025-11-19
**Node.js Version**: Based on current main branch
**Maintained By**: OSS contributors

---

## 🙏 Acknowledgments

These documents were created to help new contributors understand Node.js architecture and contribution process. They complement the official Node.js documentation and aim to provide a comprehensive, structured learning path.

**Happy Contributing! 🚀**

---

## Quick Navigation

- [Architecture Overview](./ARCHITECTURE_OVERVIEW.md)
- [Contribution Guide](./CONTRIBUTION_GUIDE.md)
- [Build System Guide](./BUILD_SYSTEM_GUIDE.md)
- [Core Subsystems Guide](./CORE_SUBSYSTEMS_GUIDE.md)
- [Testing Guide](./TESTING_GUIDE.md)
