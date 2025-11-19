# Node.js Build System & Development Setup Guide

> Complete guide to building, configuring, and developing Node.js

## Table of Contents
- [Overview](#overview)
- [Build System Architecture](#build-system-architecture)
- [Platform-Specific Setup](#platform-specific-setup)
- [Configuration Options](#configuration-options)
- [Building Node.js](#building-nodejs)
- [Development Workflows](#development-workflows)
- [Build Optimization](#build-optimization)
- [Troubleshooting](#troubleshooting)
- [Advanced Topics](#advanced-topics)

---

## Overview

### Build Tools Used

Node.js uses a multi-layered build system:

1. **GYP (Generate Your Projects)**: Primary build configuration system
2. **GN (Genie)**: Alternative build system (experimental)
3. **Make**: Build orchestration on Unix/Linux/macOS
4. **MSBuild**: Build orchestration on Windows
5. **Python**: Configuration scripts
6. **Ninja**: Fast build backend (optional)

### Build Process Flow

```
┌──────────────────────────────────────────────────┐
│  ./configure [options]                           │
│  - Detects platform                              │
│  - Checks dependencies                           │
│  - Generates config.mk and config.gypi           │
└────────────────┬─────────────────────────────────┘
                 │
┌────────────────┴─────────────────────────────────┐
│  make / vcbuild.bat                              │
│  - Runs GYP to generate build files              │
│  - Compiles C++ sources                          │
│  - Links binaries                                │
│  - Generates snapshots                           │
└────────────────┬─────────────────────────────────┘
                 │
┌────────────────┴─────────────────────────────────┐
│  Output: ./node (or node.exe on Windows)         │
└──────────────────────────────────────────────────┘
```

---

## Build System Architecture

### Key Build Files

| File | Purpose |
|------|---------|
| `configure` | Shell wrapper for configure.py |
| `configure.py` | Main configuration script (89KB) |
| `Makefile` | Unix/Linux/macOS build orchestration |
| `vcbuild.bat` | Windows build script |
| `node.gyp` | Main GYP build configuration |
| `node.gypi` | Shared GYP configuration |
| `common.gypi` | Common build settings |
| `BUILD.gn` | GN build configuration (experimental) |
| `config.mk` | Generated configuration (created by configure) |
| `config.gypi` | Generated GYP configuration |

### GYP (Generate Your Projects)

**Purpose**: Cross-platform build file generator

**How it Works**:
1. Reads `.gyp` and `.gypi` files
2. Generates native build files:
   - **Unix/Linux/macOS**: Makefiles or Ninja files
   - **Windows**: Visual Studio project files (`.vcxproj`)
3. Native build system compiles the code

**Key GYP Files**:

**`node.gyp`** - Main build configuration (~150+ lines):
```python
{
  'targets': [
    {
      'target_name': 'node',
      'type': 'executable',
      'sources': [
        'src/node.cc',
        'src/node_main.cc',
        # ... 100+ source files
      ],
      'dependencies': [
        '<(node_core_target_name)',
        'deps/uv/uv.gyp:libuv',
        'deps/v8/gypfiles/v8.gyp:v8',
        # ... other dependencies
      ],
    },
  ],
}
```

**`common.gypi`** - Shared settings:
- Compiler flags
- Linker flags
- Platform-specific configurations
- Warning levels

### Directory Structure for Build

```
node/
├── out/                        # Build output directory
│   ├── Release/               # Release build
│   │   ├── node               # Node.js executable
│   │   ├── obj/              # Object files
│   │   └── lib/              # Static libraries
│   ├── Debug/                 # Debug build
│   └── Makefile              # Generated Makefile
│
├── config.mk                   # Generated config (Unix)
├── config.gypi                 # Generated config (GYP)
├── config_fips.gypi           # FIPS configuration
└── icu_config.gypi            # ICU configuration
```

---

## Platform-Specific Setup

### Linux

**Prerequisites**:
```bash
# Debian/Ubuntu
sudo apt-get update
sudo apt-get install -y \
  python3 \
  g++ \
  make \
  git

# Fedora/RHEL/CentOS
sudo dnf install -y \
  python3 \
  gcc-c++ \
  make \
  git

# Arch Linux
sudo pacman -S python gcc make git
```

**Optional Tools**:
```bash
# ccache (faster rebuilds)
sudo apt-get install ccache

# Ninja (faster builds)
sudo apt-get install ninja-build
```

**Build**:
```bash
./configure
make -j$(nproc)
```

### macOS

**Prerequisites**:
```bash
# Install Xcode Command Line Tools
xcode-select --install

# Or install full Xcode from App Store
```

**Optional Tools**:
```bash
# Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ccache
brew install ccache

# Ninja
brew install ninja
```

**Build**:
```bash
./configure
make -j$(sysctl -n hw.ncpu)
```

### Windows

**Prerequisites**:

1. **Visual Studio 2022** (or 2019):
   - Download from https://visualstudio.microsoft.com/
   - Install "Desktop development with C++" workload
   - Includes MSBuild and compilers

2. **Python 3.x**:
   - Download from https://www.python.org/
   - Add to PATH during installation

3. **Git**:
   - Download from https://git-scm.com/

**Optional Tools**:
- **Debugging Tools for Windows** (from Windows SDK)

**Build**:
```cmd
REM Open "x64 Native Tools Command Prompt for VS 2022"

REM Full build
vcbuild.bat

REM Debug build
vcbuild.bat debug

REM Release build (default)
vcbuild.bat release

REM Run tests
vcbuild.bat test
```

**Alternative (using Make on Windows)**:
```bash
# Using WSL (Windows Subsystem for Linux)
# Follow Linux instructions inside WSL
```

### Cross-Compilation

**Android**:
```bash
./configure --dest-os=android --dest-cpu=arm64 \
  --cross-compiling \
  --without-snapshot
make -j$(nproc)
```

**ARM on x64**:
```bash
# Install cross-compiler
sudo apt-get install g++-aarch64-linux-gnu

./configure --dest-cpu=arm64 --cross-compiling
make -j$(nproc)
```

---

## Configuration Options

### Running Configuration

```bash
# Basic configuration
./configure

# View all options
./configure --help
```

### Common Options

#### Build Type

```bash
# Debug build (default for development)
./configure --debug

# Release build (optimized)
./configure

# Release build with debug symbols
./configure --debug-node
```

#### Architecture

```bash
# x64 (default on 64-bit systems)
./configure --dest-cpu=x64

# ARM64
./configure --dest-cpu=arm64

# x86 (32-bit)
./configure --dest-cpu=x86

# Other: ia32, ppc64, s390x, mips64el
```

#### Platform

```bash
# Linux (default on Linux)
./configure --dest-os=linux

# macOS
./configure --dest-os=mac

# Windows
./configure --dest-os=win

# Android
./configure --dest-os=android

# Other: freebsd, openbsd, solaris, aix
```

#### Dependencies

```bash
# Use system OpenSSL instead of bundled
./configure --shared-openssl

# Use system zlib
./configure --shared-zlib

# Use system libuv
./configure --shared-libuv

# Disable OpenSSL entirely
./configure --without-ssl

# Enable full ICU (internationalization)
./configure --with-intl=full-icu

# Disable ICU
./configure --with-intl=none
```

#### Features

```bash
# Enable FIPS (Federal Information Processing Standards)
./configure --openssl-fips

# Enable Large Pages support
./configure --enable-large-pages

# Enable LTO (Link Time Optimization)
./configure --enable-lto

# Enable Performance Tracing
./configure --with-perfctr

# Enable DTrace support
./configure --with-dtrace
```

#### Node.js Features

```bash
# Without npm
./configure --without-npm

# Without corepack
./configure --without-corepack

# Without node-options
./configure --without-node-options

# Without inspector (debugger)
./configure --without-inspector
```

#### Development Options

```bash
# Use ccache for faster rebuilds
./configure --with-ccache

# Use Ninja instead of Make
./configure --ninja

# Build with code coverage
./configure --coverage

# Enable ASan (Address Sanitizer)
./configure --enable-asan

# Enable UBSan (Undefined Behavior Sanitizer)
./configure --enable-ubsan
```

### Configuration Examples

**Fast Development Build**:
```bash
./configure --debug --ninja --with-ccache
ninja -C out/Debug
```

**Minimal Build** (smallest binary):
```bash
./configure \
  --without-npm \
  --without-corepack \
  --without-inspector \
  --with-intl=none
make -j$(nproc)
```

**Full-Featured Build**:
```bash
./configure \
  --with-intl=full-icu \
  --openssl-is-fips
make -j$(nproc)
```

**Production-Like Build**:
```bash
./configure \
  --enable-lto \
  --with-intl=full-icu
make -j$(nproc)
```

---

## Building Node.js

### Basic Build

**Unix/Linux/macOS**:
```bash
# 1. Configure
./configure

# 2. Build (using all CPU cores)
make -j$(nproc)        # Linux
make -j$(sysctl -n hw.ncpu)  # macOS

# 3. Test installation
./node --version

# 4. Run tests
make test
```

**Windows**:
```cmd
REM Open "x64 Native Tools Command Prompt for VS 2022"

REM Build
vcbuild.bat

REM Test
vcbuild.bat test
```

### Build Targets

**Common Targets** (Unix/Linux/macOS):

```bash
# Build Node.js binary only
make node

# Build and run tests
make test

# Build with specific jobs
make -j4

# Clean build artifacts
make clean

# Full clean (including dependencies)
make distclean

# Install to system
sudo make install

# Install to custom prefix
./configure --prefix=/opt/node
make -j$(nproc)
sudo make install

# Build documentation
make doc

# Lint code
make lint

# Format C++ code
make format-cpp

# Run benchmarks
make benchmark
```

**Test Targets**:
```bash
make test                    # All tests
make test-parallel          # Parallel tests (fast)
make test-sequential        # Sequential tests
make cctest                 # C++ unit tests
make test-valgrind          # Memory leak detection
make test-ci                # CI-like testing
make coverage               # Code coverage
```

**Additional Targets**:
```bash
make binary                 # Create release tarball
make pkg                    # macOS installer package
make install-docs           # Install documentation
make uninstall             # Uninstall Node.js
```

### Incremental Builds

After making changes:

```bash
# Rebuild only changed files
make -j$(nproc)

# If you only changed JavaScript files
make js2c                   # Rebuild embedded JS
./node                      # Run new version
```

### Ninja Builds (Faster)

**Setup**:
```bash
# Configure with Ninja
./configure --ninja

# Build
ninja -C out/Release

# Specific target
ninja -C out/Release node
```

**Advantages**:
- Faster parallel builds
- Better dependency tracking
- Faster incremental rebuilds

---

## Development Workflows

### Quick Development Cycle

```bash
# 1. Configure once (with ccache and ninja)
./configure --debug --ninja --with-ccache

# 2. Build
ninja -C out/Debug

# 3. Test specific change
./out/Debug/node test/parallel/test-http-server.js

# 4. Make changes...

# 5. Rebuild (only changed files)
ninja -C out/Debug

# 6. Test again
./out/Debug/node test/parallel/test-http-server.js
```

### Developing C++ Code

**Workflow**:
```bash
# 1. Edit C++ file (e.g., src/node_http.cc)
vim src/node_http.cc

# 2. Rebuild
make -j$(nproc)

# 3. Test
./node test/parallel/test-http-*.js

# 4. Debug if needed
gdb --args ./node test/parallel/test-http-server.js
```

**Faster Iteration**:
```bash
# Use ccache
./configure --with-ccache

# Subsequent builds are faster
make -j$(nproc)
```

### Developing JavaScript Code

**Direct File Loading** (no rebuild needed):

```bash
# Build once
make -j$(nproc)

# Edit JavaScript file
vim lib/http.js

# Test immediately (JS loaded from disk in debug builds)
./node test/parallel/test-http-server.js
```

**Note**: Release builds embed JavaScript, requiring rebuild

### Testing Changes

```bash
# Run specific test
./node test/parallel/test-http-server.js

# Run all HTTP tests
./node test/parallel/test-http-*.js

# Use test runner
./node test/parallel/test-runner.js

# Run with test options
./node --expose-internals test/parallel/test-http-server.js
```

### Debugging

**GDB (Linux/macOS)**:
```bash
# Build with debug symbols
./configure --debug
make -j$(nproc)

# Debug Node.js
gdb --args ./node test/parallel/test-http-server.js

# GDB commands
(gdb) break node::Start
(gdb) run
(gdb) backtrace
(gdb) print variable
(gdb) continue
```

**LLDB (macOS)**:
```bash
lldb -- ./node test/parallel/test-http-server.js

# LLDB commands
(lldb) breakpoint set --name node::Start
(lldb) run
(lldb) bt
(lldb) print variable
(lldb) continue
```

**Visual Studio Debugger (Windows)**:
```cmd
REM Build debug version
vcbuild.bat debug

REM Open in Visual Studio
devenv out\Debug\node.sln

REM Or use WinDbg
windbg node.exe
```

**Chrome DevTools**:
```bash
# Run with inspector
./node --inspect-brk test/parallel/test-http-server.js

# Open chrome://inspect in Chrome
# Click "inspect" on the target
```

---

## Build Optimization

### Using ccache

**Setup**:
```bash
# Install ccache
sudo apt-get install ccache    # Linux
brew install ccache            # macOS

# Configure Node.js with ccache
./configure --with-ccache

# Verify
ccache -s  # Show statistics
```

**Performance**:
- First build: Normal speed
- Subsequent builds: 5-10x faster

### Using Ninja

**Setup**:
```bash
# Install Ninja
sudo apt-get install ninja-build    # Linux
brew install ninja                  # macOS

# Configure
./configure --ninja

# Build
ninja -C out/Release
```

**Performance**:
- 20-30% faster than Make
- Better parallelism
- Faster startup

### Parallel Builds

```bash
# Use all CPU cores
make -j$(nproc)        # Linux
make -j$(sysctl -n hw.ncpu)  # macOS

# Use specific number of jobs
make -j4

# Ninja (auto-detects cores)
ninja -C out/Release
```

### Link Time Optimization (LTO)

```bash
# Configure with LTO
./configure --enable-lto

# Build (slower compilation, faster binary)
make -j$(nproc)
```

**Trade-offs**:
- **Build time**: +50-100% slower
- **Binary size**: -10-20% smaller
- **Runtime speed**: +5-15% faster

### Debug vs Release Builds

**Debug Build**:
```bash
./configure --debug
make -j$(nproc)
```
- **Pros**: Easier debugging, faster compilation
- **Cons**: Slower runtime (2-3x)
- **Size**: Larger binary

**Release Build**:
```bash
./configure
make -j$(nproc)
```
- **Pros**: Faster runtime, smaller binary
- **Cons**: Harder to debug
- **Size**: Optimized

**Hybrid** (release with debug info):
```bash
./configure --debug-node
make -j$(nproc)
```
- **Pros**: Fast runtime + debug symbols
- **Best for**: Performance debugging

---

## Troubleshooting

### Common Build Errors

#### Python Version Issues

**Error**: `Python 2.x is required`

**Solution**:
```bash
# Ensure Python 3 is used
python3 --version

# Set Python 3 as default (if needed)
export PYTHON=python3
./configure
```

#### Compiler Not Found

**Error**: `C++ compiler not found`

**Solution**:
```bash
# Linux
sudo apt-get install g++

# macOS
xcode-select --install

# Verify
g++ --version
```

#### Out of Memory

**Error**: `virtual memory exhausted`

**Solution**:
```bash
# Reduce parallel jobs
make -j2

# Or build sequentially
make
```

#### Ninja Not Found

**Error**: `ninja: command not found`

**Solution**:
```bash
# Install Ninja
sudo apt-get install ninja-build

# Or reconfigure without Ninja
./configure
make
```

#### Permission Denied on Install

**Error**: `Permission denied` during `make install`

**Solution**:
```bash
# Use sudo
sudo make install

# Or install to user directory
./configure --prefix=$HOME/.local
make install
```

### Clean Builds

**Remove build artifacts**:
```bash
# Clean object files (keeps configuration)
make clean

# Full clean (removes configuration)
make distclean

# Manual clean
rm -rf out/
rm config.mk config.gypi
```

### Dependency Issues

**Updating dependencies**:
```bash
# Pull latest changes
git pull upstream main

# Reconfigure
./configure

# Clean rebuild
make distclean
./configure
make -j$(nproc)
```

### Platform-Specific Issues

**macOS - Xcode Updates**:
```bash
# After Xcode update, reinstall command line tools
sudo rm -rf /Library/Developer/CommandLineTools
xcode-select --install
```

**Linux - Missing Libraries**:
```bash
# Install development packages
sudo apt-get install build-essential
```

**Windows - Visual Studio Version**:
```cmd
REM Ensure correct VS version
REM Open "x64 Native Tools Command Prompt for VS 2022"
REM NOT "Developer Command Prompt"
```

---

## Advanced Topics

### Custom V8 Builds

```bash
# Use custom V8
./configure --v8-options="--max-old-space-size=8192"

# Build V8 separately
cd deps/v8
gn gen out/Release
ninja -C out/Release
```

### Building with Custom OpenSSL

```bash
# Use system OpenSSL
./configure --shared-openssl

# Use specific OpenSSL
./configure --shared-openssl \
  --shared-openssl-includes=/opt/openssl/include \
  --shared-openssl-libpath=/opt/openssl/lib
```

### Static Linking

```bash
# Fully static binary
./configure --fully-static

# Semi-static (static C++ libs, dynamic libc)
./configure --partly-static
```

### Cross-Compilation

**ARM64 on x64**:
```bash
# Install cross-compiler
sudo apt-get install g++-aarch64-linux-gnu

# Configure
./configure \
  --dest-cpu=arm64 \
  --cross-compiling \
  CC=aarch64-linux-gnu-gcc \
  CXX=aarch64-linux-gnu-g++

# Build
make -j$(nproc)
```

### Building Specific Versions

```bash
# Checkout specific version
git checkout v20.10.0

# Configure and build
./configure
make -j$(nproc)
```

### Code Coverage

```bash
# Configure with coverage
./configure --coverage

# Build
make -j$(nproc)

# Run tests (generates coverage data)
make coverage

# View coverage report
open coverage/index.html
```

### Sanitizers (Debugging Tools)

**Address Sanitizer (ASan)**:
```bash
./configure --debug --enable-asan
make -j$(nproc)
./node test/parallel/test-http-server.js
```

**Undefined Behavior Sanitizer (UBSan)**:
```bash
./configure --debug --enable-ubsan
make -j$(nproc)
```

**Memory Sanitizer (MSan)**:
```bash
./configure --debug --enable-msan
make -j$(nproc)
```

### Embedding Node.js

**Building as Library**:
```bash
./configure --shared
make -j$(nproc)
```

**Output**: `libnode.so` (or `.dylib` on macOS, `.dll` on Windows)

---

## Quick Reference

### Common Build Commands

| Task | Command |
|------|---------|
| Configure | `./configure` |
| Build | `make -j$(nproc)` |
| Debug build | `./configure --debug && make -j$(nproc)` |
| Clean | `make clean` |
| Full clean | `make distclean` |
| Test | `make test` |
| Fast test | `make test-parallel` |
| Lint | `make lint` |
| Install | `sudo make install` |
| Uninstall | `sudo make uninstall` |

### Configuration Shortcuts

| Goal | Configuration |
|------|---------------|
| Fast dev build | `./configure --debug --ninja --with-ccache` |
| Small binary | `./configure --without-npm --with-intl=none` |
| Production | `./configure --enable-lto` |
| Debugging | `./configure --debug` |
| Coverage | `./configure --coverage` |

---

## Additional Resources

- **Building Guide**: `BUILDING.md`
- **Configure Help**: `./configure --help`
- **Makefile Targets**: `make help`
- **GYP Documentation**: https://gyp.gsrc.io/
- **V8 Building**: `deps/v8/BUILD.gn`

---

**Last Updated**: 2025-11-19
**Tested On**: Ubuntu 22.04, macOS 14, Windows 11
