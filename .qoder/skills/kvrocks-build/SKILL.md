---
name: kvrocks-build
description: Build and manage Apache Kvrocks. Use when building the project, running tests, managing CMake configuration, troubleshooting build issues, adding/updating dependencies, or when the user mentions x.py, CMake, build errors, compilation failures, or dependency problems.
---

# Kvrocks Build System

## Build Tool: `x.py`

`x.py` is the unified build script. All build operations go through it.

### Quick Reference

```bash
# Build (default: RelWithDebInfo, output to ./build/)
./x.py build
./x.py build -j 8                  # 8 parallel jobs
./x.py build --ninja               # use Ninja (faster incremental)
./x.py build --unittest            # also build unit tests
./x.py build --compiler gcc        # force GCC
./x.py build --compiler clang      # force Clang
./x.py build --skip-build          # CMake configure only

# Debug build
./x.py build -DCMAKE_BUILD_TYPE=Debug

# With TLS support
./x.py build -DENABLE_OPENSSL=ON

# Custom build dir
./x.py build my-build-dir

# Pass arbitrary CMake options
./x.py build -DENABLE_LTO=ON -DENABLE_ASAN=ON -DDISABLE_JEMALLOC=ON

# Fetch dependencies to a local dir (for offline builds)
./x.py fetch-deps                  # default: build-deps/
./x.py fetch-deps my-deps-dir

# Build with pre-fetched deps
./x.py build --dep-dir build-deps
```

### Testing

```bash
# C++ unit tests (requires --unittest build first)
./x.py test cpp                    # run all unit tests
./x.py test cpp -- --gtest_filter="HashTest.*"  # filter tests

# Go integration tests (requires redis-cli and running kvrocks binary)
./x.py test go                     # run all Go tests
./x.py test go tests/gocase/unit/...  # run specific test path
```

### Code Quality

```bash
./x.py format                      # auto-format (clang-format)
./x.py check format                # check format without modifying
./x.py check tidy                  # clang-tidy (needs compile_commands.json)
./x.py check tidy -j 8             # parallel tidy
./x.py check tidy --fix            # auto-fix tidy warnings
./x.py check golangci-lint         # Go test linting
```

### Version Requirements

| Tool | Min Version |
|------|-------------|
| CMake | 3.16.0 |
| GCC | 11 |
| Clang | 12 |
| AppleClang | 13 |
| clang-format | 18.0.0 |
| clang-tidy | 18.0.0 |
| golangci-lint | 2.9.0 |
| C++ Standard | C++20 |

## CMake Options

| Option | Default | Description |
|--------|---------|-------------|
| `CMAKE_BUILD_TYPE` | RelWithDebInfo | Build type (Debug/Release/RelWithDebInfo) |
| `DISABLE_JEMALLOC` | OFF | Disable jemalloc allocator |
| `ENABLE_ASAN` | OFF | Address sanitizer (requires DISABLE_JEMALLOC=ON) |
| `ENABLE_TSAN` | OFF | Thread sanitizer (requires DISABLE_JEMALLOC=ON) |
| `ENABLE_UBSAN` | OFF | Undefined behavior sanitizer |
| `ASAN_WITH_LSAN` | ON | Enable leak sanitizer with ASan |
| `ENABLE_STATIC_LIBSTDCXX` | ON | Link static libstdc++ |
| `ENABLE_LUAJIT` | ON | Use LuaJIT instead of Lua |
| `ENABLE_OPENSSL` | OFF | Enable TLS support |
| `ENABLE_LTO` | OFF | Link-time optimization |
| `ENABLE_NEW_ENCODING` | ON | 64-bit size + ms expire encoding |
| `PORTABLE` | 0 | Disable arch-specific optimizations |
| `SYMBOLIZE_BACKEND` | "" | cpptrace backend (libbacktrace/libdwarf) |
| `NINJA_MAKE_JOBS` | 4 | Concurrent jobs when Ninja calls make |
| `DEPS_FETCH_DIR` | "" | Directory to cache dependency archives |
| `DEPS_FETCH_PROXY` | "" | URL proxy for fetching dependencies |

### Mutually Exclusive Options

- ASan and TSan cannot be used together
- ASan/TSan require `DISABLE_JEMALLOC=ON`
- jemalloc is auto-disabled on macOS

## Dependencies

All dependencies are built from source via CMake `FetchContent`. Each dependency has a `cmake/<name>.cmake` file.

### Core Dependencies

| Dependency | Version | Purpose | CMake File |
|-----------|---------|---------|------------|
| RocksDB | v10.10.1 | Storage engine | `cmake/rocksdb.cmake` |
| libevent | 2.1.12-stable | Async I/O event loop | `cmake/libevent.cmake` |
| jemalloc | (latest) | Memory allocator | `cmake/jemalloc.cmake` |
| snappy | - | Compression | `cmake/snappy.cmake` |
| lz4 | - | Compression | `cmake/lz4.cmake` |
| zstd | - | Compression | `cmake/zstd.cmake` |
| zlib | - | Compression | `cmake/zlib.cmake` |
| TBB | - | Thread building blocks | `cmake/tbb.cmake` |
| fmt | - | String formatting | `cmake/fmt.cmake` |
| spdlog | - | Logging | `cmake/spdlog.cmake` |
| LuaJIT/Lua | - | Scripting engine | `cmake/luajit.cmake` / `cmake/lua.cmake` |
| jsoncons | - | JSON processing | `cmake/jsoncons.cmake` |
| xxhash | - | Fast hashing | `cmake/xxhash.cmake` |
| cpptrace | - | Stack trace symbolization | `cmake/cpptrace.cmake` |
| googletest | - | Unit testing framework | `cmake/gtest.cmake` |

### Dependency Management Pattern

Dependencies use a custom `FetchContent_DeclareGitHubWithMirror` macro from `cmake/utils.cmake`:

```cmake
FetchContent_DeclareGitHubWithMirror(dep_name
  github_org/github_repo git_tag
  MD5=<hash>
)

FetchContent_MakeAvailableWithArgs(dep_name
  CMAKE_OPTION1=value1
  CMAKE_OPTION2=value2
)
```

To add a new dependency:
1. Create `cmake/<name>.cmake` following the pattern above
2. Include it in `CMakeLists.txt`: `include(cmake/<name>.cmake)`
3. Add to `EXTERNAL_LIBS`: `list(APPEND EXTERNAL_LIBS <target_name>)`
4. Optionally add a Find module in `cmake/modules/`

### Offline / Cached Builds

```bash
# Step 1: Fetch all deps to a local directory
./x.py fetch-deps build-deps

# Step 2: Build using cached deps
./x.py build --dep-dir build-deps
```

For CI or restricted network environments, use `DEPS_FETCH_PROXY`:
```bash
./x.py build -DDEPS_FETCH_PROXY=https://my-mirror.example.com/
```

## Build Targets

| Target | Description |
|--------|-------------|
| `kvrocks` | Main server binary |
| `kvrocks2redis` | Sync tool (kvrocks → redis) |
| `unittest` | C++ unit tests (GoogleTest) |

## Common Build Issues

### "autoconf is required to build jemalloc"
Install autoconf: `apt install autoconf` or `yum install autoconf`

### "cannot find static library of libstdc++"
Add `-DENABLE_STATIC_LIBSTDCXX=OFF` or install `libstdc++-static` package.

### ASan/TSan build fails
Must also pass `-DDISABLE_JEMALLOC=ON`.

### compile_commands.json not found (for clang-tidy)
Build with CMake first (`./x.py build` or `./x.py build --skip-build`) to generate it in the build directory.

### Dependency download fails
Use `--dep-dir` with pre-fetched archives, or set `DEPS_FETCH_PROXY` for network-restricted environments.

### Cross-compilation
Use `--toolchain` flag:
```bash
./x.py build --toolchain path/to/toolchain.cmake
```
A RISC-V toolchain file exists at `cmake/riscv64.cmake`.

## Additional Resources

- For full CMake option details and dependency versions, see [reference.md](reference.md)
