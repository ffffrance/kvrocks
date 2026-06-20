# Kvrocks Build Reference

## x.py Subcommands Detail

### build

```
x.py build [BUILD_DIR] [options]

Positional:
  BUILD_DIR             Build output directory (default: build)

Options:
  -j N, --jobs N        Parallel build jobs
  --ninja               Use Ninja build system
  --unittest            Build unittest target alongside kvrocks
  --compiler {auto,gcc,clang}  Compiler selection (default: auto)
  --toolchain FILE      CMake toolchain file for cross-compilation
  --cmake-path PATH     Path to cmake binary (default: cmake)
  -D key=value          Extra CMake definitions (repeatable)
  --skip-build          Run CMake configure only, skip build
  --dep-dir DIR         Directory containing pre-fetched dependency archives
```

Build flow:
1. Checks `autoconf` availability (needed by jemalloc)
2. Validates CMake version (>= 3.16.0)
3. Creates BUILD_DIR if not exists
4. Runs `cmake <source_dir> <options>` in BUILD_DIR
5. Builds targets: `kvrocks`, `kvrocks2redis` (+ `unittest` if --unittest)

### fetch-deps

```
x.py fetch-deps [DEP_DIR]

Positional:
  DEP_DIR   Directory to store downloaded archives (default: build-deps)
```

Creates a temporary build dir, runs CMake configure with `DEPS_FETCH_DIR` set, which causes `FetchContent_DeclareGitHubWithMirror` to save archives to DEP_DIR instead of downloading each time.

### test cpp

```
x.py test cpp [BUILD_DIR] [-- extra_args...]

Positional:
  BUILD_DIR   Directory containing built unittest binary (default: build)

Extra args are forwarded to the unittest binary (GoogleTest).
Examples:
  ./x.py test cpp -- --gtest_filter="StringTest.*"
  ./x.py test cpp -- --gtest_repeat=3
  ./x.py test cpp -- --gtest_list_tests
```

### test go

```
x.py test go [BUILD_DIR] [options] [-- extra_args...]

Positional:
  BUILD_DIR   Directory containing built kvrocks binary (default: build)

Options:
  --cli-path PATH   Path to redis-cli (default: redis-cli)

Extra args are forwarded to `go test`.
Examples:
  ./x.py test go -- -run TestString
  ./x.py test go tests/gocase/unit/type/string/...
```

### format / check

```
x.py format                          # Fix format in-place
x.py check format                    # Dry-run check (fails on issues)
x.py check tidy [BUILD_DIR] [-j N] [--fix]
x.py check golangci-lint
```

## CMake Dependency Details

### FetchContent_DeclareGitHubWithMirror

Defined in `cmake/utils.cmake`. Downloads from GitHub with optional proxy and caching:

- `DEPS_FETCH_PROXY`: URL prefix prepended to download URL (for mirror/proxy)
- `DEPS_FETCH_DIR`: Local directory to cache downloaded archives

Download URL pattern: `{DEPS_FETCH_PROXY}https://github.com/{repo}/archive/{tag}.zip`

### FetchContent_MakeAvailableWithArgs

Wrapper around `FetchContent_MakeAvailable` that temporarily sets CMake variables before `add_subdirectory` and restores them after. This allows passing build options to dependencies without polluting the parent scope.

### RocksDB Configuration

Built with these options by default:
- `WITH_SNAPPY=ON`, `WITH_LZ4=ON`, `WITH_ZLIB=ON`, `WITH_ZSTD=ON` (compression)
- `WITH_TBB=ON` (threading)
- `USE_RTTI=ON`
- `ROCKSDB_BUILD_SHARED=OFF` (static only)
- `WITH_JEMALLOC` (auto-detected from `DISABLE_JEMALLOC`)
- `PORTABLE` (controls arch-specific optimizations)

### Source Structure

```
CMakeLists.txt          # Root build file
cmake/
  utils.cmake           # FetchContent helpers
  rocksdb.cmake         # RocksDB dependency
  libevent.cmake        # libevent dependency
  jemalloc.cmake        # jemalloc dependency
  [other deps].cmake    # Other dependencies
  modules/              # Find*.cmake modules
  checks/               # Compile-time feature checks
src/
  cli/main.cc           # kvrocks entry point
  version.h.in          # Version template (configured by CMake)
  VERSION.txt           # Current version string
utils/
  kvrocks2redis/        # Sync tool source
tests/
  cppunit/              # GoogleTest unit tests
  gocase/               # Go integration tests
```

### Build Artifacts

| File | Location | Description |
|------|----------|-------------|
| `kvrocks` | `build/kvrocks` | Main server binary |
| `kvrocks2redis` | `build/kvrocks2redis` | Sync tool |
| `unittest` | `build/unittest` | Unit test binary |
| `compile_commands.json` | `build/compile_commands.json` | Compilation database (for clang-tidy/clangd) |
| `version.h` | `build/version.h` | Generated version header with git SHA |

### Compiler Flags

Default flags applied to `kvrocks_objs`:
- `-Wall -Wpedantic -Wsign-compare -Wreturn-type -fno-omit-frame-pointer`
- `-Werror=unused-parameter -Werror=unused-result -Werror=unused-variable`
- `-static-libgcc` (Linux only, GCC/Clang)
- `-static-libstdc++` (when `ENABLE_STATIC_LIBSTDCXX=ON`)

### Sanitizer Configurations

**ASan**: `-fsanitize=address` + optionally `-fsanitize=leak`
**TSan**: `-fsanitize=thread`
**UBSan**: `-fsanitize=undefined -fno-sanitize=alignment,vptr,function,float-divide-by-zero`

All sanitizers require `DISABLE_JEMALLOC=ON`. ASan and TSan are mutually exclusive.

### Cross-Compilation

Toolchain file example (`cmake/riscv64.cmake`):
```bash
./x.py build --toolchain cmake/riscv64.cmake
```

For other architectures, create a CMake toolchain file specifying:
- `CMAKE_SYSTEM_NAME`
- `CMAKE_C_COMPILER` / `CMAKE_CXX_COMPILER`
- `CMAKE_FIND_ROOT_PATH`

### macOS Notes

- jemalloc is auto-disabled on macOS
- OpenSSL must be found via `find_package(OpenSSL)`
- macOS SDK path is auto-detected via `xcrun`
