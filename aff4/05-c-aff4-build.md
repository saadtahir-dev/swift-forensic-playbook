# Building `Velocidex/c-aff4` as a Universal Static Library on Apple Silicon macOS

> Companion guide to the `aff4-cpp-lite` build. Produces a universal `libaff4.a` (arm64 + x86_64) suitable for SwiftPM integration.

---

## Overview

`Velocidex/c-aff4` is a full read+write AFF4 C++ implementation. Unlike `aff4-cpp-lite`, it includes:

- Multi-threaded imaging
- Full AFF4 image creation and manipulation
- The `aff4imager` binary
- RDF metadata via raptor2
- Logging via spdlog

**Final output:**
```
(your BUILD_ROOT)/LIB-CAFF4-SPM/deps/universal/lib/libaff4.a
```

Validated as universal (`arm64 + x86_64`), static, and dylib-clean.

---

## Environment Variables

Set these before starting. Keep them in your shell for the entire build session:

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"    # shared deps from aff4-cpp-lite build
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"      # c-aff4 specific build root

mkdir -p "$CAFF4_ROOT/deps"/{arm64,x86_64,universal/lib}
```

---

## Dependencies

### Reused from previous build (already universal)

These were built during the `aff4-cpp-lite` phase and are reused directly:

| Library | Path |
|---|---|
| zlib | `$AFF4_LITE_ROOT/deps/{arch}/zlib` |
| snappy | `$AFF4_LITE_ROOT/deps/{arch}/snappy` |
| lz4 | `$AFF4_LITE_ROOT/deps/{arch}/lz4` |
| raptor2 | `$AFF4_LITE_ROOT/deps/{arch}/raptor` |
| uriparser | `$AFF4_LITE_ROOT/deps/{arch}/uriparser` |
| openssl | `$AFF4_LITE_ROOT/deps/{arch}/openssl` |

### New deps for c-aff4

**tclap** — header-only CLI flag parsing library:
```bash
brew install tclap
```

**util-linux** — provides a proper `libuuid` with a real `.pc` file. macOS's built-in uuid stubs are not sufficient for c-aff4's configure:
```bash
brew install util-linux
```

**spdlog (vendored, header-only)** — c-aff4 was written against an older spdlog that bundled fmt internally. Modern Homebrew spdlog 1.x + fmt 12.x is ABI-incompatible with c-aff4's source. The fix is to use spdlog 0.17.0 in header-only mode, bypassing Homebrew entirely:

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"

cd "$CAFF4_ROOT"
git clone --depth=1 --branch v0.17.0 https://github.com/gabime/spdlog.git spdlog-src

# Verify bundled fmt is present
ls "$CAFF4_ROOT/spdlog-src/include/spdlog/fmt/"
# Expected: bundled  fmt.h  ostr.h
```

> **Why vendored spdlog?** Modern Homebrew ships spdlog 1.x which uses fmt as an external dependency. fmt 12.x changed the formatter API — custom types like `aff4::URN` and `aff4::AFF4_IMAGE_COMPRESSION_ENUM_t` require explicit `fmt::formatter` specializations that c-aff4 never added. Using spdlog 0.17.0 in header-only mode with its bundled fmt avoids the entire ABI mismatch.

> **Architecture independence:** spdlog 0.17.0 is consumed in header-only mode — no compiled library, no architecture-specific objects. The same `$CAFF4_ROOT/spdlog-src/include` path is used unchanged for both arm64 and x86_64 builds.

**Unlink Homebrew spdlog and fmt** to prevent linker contamination:
```bash
brew unlink spdlog
brew unlink fmt

# Verify completely removed from pkg-config
pkg-config --libs spdlog 2>/dev/null
pkg-config --libs fmt 2>/dev/null
# Both must return blank or error — NOT /opt/homebrew paths
```

---

## Clone and Bootstrap

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"

cd "$CAFF4_ROOT"
git clone https://github.com/Velocidex/c-aff4.git caff4-src
cd caff4-src

autoreconf -fi
```

Obsolete macro warnings from `autoreconf` are harmless.

---

## Fix: Exclude pmem Tools

Even with `--without-yaml`, the `tools/pmem/osxpmem` target attempts to compile yaml-cpp code and fails. Remove it from the build entirely:

```bash
sed -i '' 's|SUBDIRS = aff4 tools/pmem tests|SUBDIRS = aff4 tests|' Makefile.am

# Regenerate after the Makefile.am change
autoreconf -fi
```

> This removes the pmem acquisition tooling while keeping the core `libaff4.a` library and tests intact.

---

## Build for arm64

### Configure

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"

# Clear any contaminating environment variables first
unset PKG_CONFIG_PATH
unset PKG_CONFIG_LIBDIR
unset LIBS

# Add tclap and uuid to pkg-config path
export PKG_CONFIG_PATH="\
/opt/homebrew/opt/tclap/lib/pkgconfig:\
$(brew --prefix util-linux)/lib/pkgconfig:\
$AFF4_LITE_ROOT/deps/arm64/zlib/lib/pkgconfig:\
$AFF4_LITE_ROOT/deps/arm64/raptor/lib/pkgconfig:\
$AFF4_LITE_ROOT/deps/arm64/lz4/lib/pkgconfig:\
$AFF4_LITE_ROOT/deps/arm64/uriparser/lib/pkgconfig:\
/opt/homebrew/Library/Homebrew/os/mac/pkgconfig/15"

export PKG_CONFIG_LIBDIR="$PKG_CONFIG_PATH"

./configure --enable-static --disable-shared \
  CC="clang -arch arm64" \
  CXX="clang++ -arch arm64" \
  CFLAGS="-std=gnu11 -arch arm64 -mmacosx-version-min=12.0" \
  CXXFLAGS="-std=c++14 -stdlib=libc++ -arch arm64 -mmacosx-version-min=12.0" \
  LDFLAGS="-stdlib=libc++ \
    -L$AFF4_LITE_ROOT/deps/arm64/zlib/lib \
    -L$AFF4_LITE_ROOT/deps/arm64/snappy/lib \
    -L$AFF4_LITE_ROOT/deps/arm64/lz4/lib \
    -L$AFF4_LITE_ROOT/deps/arm64/raptor/lib \
    -L$AFF4_LITE_ROOT/deps/arm64/uriparser/lib" \
  CPPFLAGS="-I$AFF4_LITE_ROOT/deps/arm64/zlib/include \
    -I$AFF4_LITE_ROOT/deps/arm64/snappy/include \
    -I$AFF4_LITE_ROOT/deps/arm64/lz4/include \
    -I$AFF4_LITE_ROOT/deps/arm64/raptor/include \
    -I$AFF4_LITE_ROOT/deps/arm64/uriparser/include \
    -I/opt/homebrew/opt/tclap/include \
    -I$CAFF4_ROOT/spdlog-src/include" \
  --host=aarch64-apple-darwin \
  --prefix="$CAFF4_ROOT/deps/arm64/caff4" \
  --without-yaml \
  --disable-strip
```

### Build and install

```bash
make -j1 V=1 && make install
```

> **`-j1`** is intentional — parallel builds can cause race conditions with autotools-generated dependencies.

### Verify

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"

lipo -info "$CAFF4_ROOT/deps/arm64/caff4/lib/libaff4.a"
# Expected: Non-fat file ... is architecture: arm64
```

---

## Build for x86_64

### pkg-config isolation (critical on Apple Silicon)

Homebrew on Apple Silicon installs arm64-only packages under `/opt/homebrew`. Without isolation, pkg-config resolves arm64 libraries during x86_64 builds, causing architecture mismatch linker failures. Isolate pkg-config to the custom x86_64 dep tree:

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"

unset PKG_CONFIG_PATH
unset PKG_CONFIG_LIBDIR

export PKG_CONFIG=/opt/homebrew/bin/pkg-config

export PKG_CONFIG_PATH="\
$AFF4_LITE_ROOT/deps/x86_64/zlib/lib/pkgconfig:\
$AFF4_LITE_ROOT/deps/x86_64/lz4/lib/pkgconfig:\
$AFF4_LITE_ROOT/deps/x86_64/raptor/lib/pkgconfig:\
$AFF4_LITE_ROOT/deps/x86_64/uriparser/lib/pkgconfig:\
/opt/homebrew/opt/tclap/lib/pkgconfig:\
$(brew --prefix util-linux)/lib/pkgconfig:\
/opt/homebrew/Library/Homebrew/os/mac/pkgconfig/15"

export PKG_CONFIG_LIBDIR="$PKG_CONFIG_PATH"
```

Verify all outputs reference only `deps/x86_64` or system paths — never `/opt/homebrew/Cellar`:

```bash
pkg-config --libs zlib
pkg-config --libs raptor2
pkg-config --cflags tclap
pkg-config --libs uuid
```

### Clean previous build artifacts

```bash
make distclean
autoreconf -fi

# Re-apply the pmem exclusion after distclean
sed -i '' 's|SUBDIRS = aff4 tools/pmem tests|SUBDIRS = aff4 tests|' Makefile.am
autoreconf -fi
```

### Configure

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"

./configure --enable-static --disable-shared \
  CC="clang -arch x86_64" \
  CXX="clang++ -arch x86_64" \
  CFLAGS="-std=gnu11 -arch x86_64 -mmacosx-version-min=12.0" \
  CXXFLAGS="-std=c++14 -stdlib=libc++ -arch x86_64 -mmacosx-version-min=12.0" \
  LDFLAGS="-stdlib=libc++ \
    -L$AFF4_LITE_ROOT/deps/x86_64/zlib/lib \
    -L$AFF4_LITE_ROOT/deps/x86_64/snappy/lib \
    -L$AFF4_LITE_ROOT/deps/x86_64/lz4/lib \
    -L$AFF4_LITE_ROOT/deps/x86_64/raptor/lib \
    -L$AFF4_LITE_ROOT/deps/x86_64/uriparser/lib" \
  CPPFLAGS="-I$AFF4_LITE_ROOT/deps/x86_64/zlib/include \
    -I$AFF4_LITE_ROOT/deps/x86_64/snappy/include \
    -I$AFF4_LITE_ROOT/deps/x86_64/lz4/include \
    -I$AFF4_LITE_ROOT/deps/x86_64/raptor/include \
    -I$AFF4_LITE_ROOT/deps/x86_64/uriparser/include \
    -I/opt/homebrew/opt/tclap/include \
    -I$CAFF4_ROOT/spdlog-src/include" \
  --host=x86_64-apple-darwin \
  --prefix="$CAFF4_ROOT/deps/x86_64/caff4" \
  --without-yaml \
  --disable-strip
```

### Build and install

```bash
make -j1 V=1 && make install
```

### Verify

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"

lipo -info "$CAFF4_ROOT/deps/x86_64/caff4/lib/libaff4.a"
# Expected: Non-fat file ... is architecture: x86_64
```

### Unset pkg-config isolation after build

```bash
unset PKG_CONFIG_PATH
unset PKG_CONFIG_LIBDIR
```

---

## Create Universal Binary

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"

mkdir -p "$CAFF4_ROOT/deps/universal/lib"

lipo -create \
  "$CAFF4_ROOT/deps/arm64/caff4/lib/libaff4.a" \
  "$CAFF4_ROOT/deps/x86_64/caff4/lib/libaff4.a" \
  -output "$CAFF4_ROOT/deps/universal/lib/libaff4.a"

lipo -info "$CAFF4_ROOT/deps/universal/lib/libaff4.a"
# Expected: Architectures in the fat file: ... are: x86_64 arm64
```

---

## Verification

### Architecture check

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"

lipo -info "$CAFF4_ROOT/deps/universal/lib/libaff4.a"
# Expected: Architectures in the fat file: ... are: x86_64 arm64
```

### dylib dependency check

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"

otool -L "$CAFF4_ROOT/deps/universal/lib/libaff4.a"
```

Clean output shows only archive object members — no dylib paths:

```
Archive : libaff4.a (architecture x86_64)
libaff4.a(aff4_image.o):
libaff4.a(libaff4_la-aff4.o):
...
Archive : libaff4.a (architecture arm64)
libaff4.a(aff4_image.o):
libaff4.a(libaff4_la-aff4.o):
...
```

The archive must NOT show:
- Homebrew dylib paths
- `/opt/homebrew/...` references
- `libfmt.dylib`, `libspdlog.dylib`, or `libssl.dylib`

---

## Issues Encountered and Resolutions

| Issue | Cause | Fix |
|---|---|---|
| `fmt::v12::` unresolved symbols | Homebrew fmt 12.x leaked into build via pkg-config | `brew unlink spdlog fmt`, use vendored spdlog 0.17.0 headers only |
| `type_is_unformattable_for` compile errors | fmt 12 API requires `fmt::formatter` specializations c-aff4 never added | Vendored spdlog 0.17.0 with bundled old fmt |
| Stale fmt v12 `.o` files after unlinking | Previous compile artifacts referenced fmt v12 | `make distclean` + full rebuild from scratch |
| arm64 libs resolved during x86_64 build | Homebrew pkg-config resolves arm64-only paths | Isolated `PKG_CONFIG_PATH` + `PKG_CONFIG_LIBDIR` to x86_64 dep tree |
| `configure: error: uuid library not found` | macOS uuid stubs have no proper `.pc` file | `brew install util-linux` provides real `libuuid` |
| `configure: error: OS macos is not supported` | `--host` flag used `macos` instead of `darwin` | Use `--host=aarch64-apple-darwin` / `--host=x86_64-apple-darwin` |
| yaml-cpp / osxpmem build failure | `tools/pmem` compiled yaml-cpp even with `--without-yaml` | Remove `tools/pmem` from `SUBDIRS` in `Makefile.am` |
| tclap headers not found | CPPFLAGS missing tclap include path | Add `-I/opt/homebrew/opt/tclap/include` to `CPPFLAGS` |

---

## Key Lessons

| Lesson | Detail |
|---|---|
| **spdlog/fmt versioning is fragile** | c-aff4 was written against old bundled fmt. Never mix modern external fmt with this codebase. Use vendored spdlog 0.17.0 header-only. |
| **`make distclean` before switching fmt strategy** | Stale `.o` files compiled against fmt v12 will re-link against v12 even after Homebrew unlinking. Full rebuild required. |
| **`brew unlink` is not enough alone** | Object files already compiled carry fmt v12 symbol references. Must distclean + rebuild. |
| **pkg-config isolation is mandatory for x86_64 on Apple Silicon** | Same pattern as `aff4-cpp-lite`. Always isolate `PKG_CONFIG_PATH` + `PKG_CONFIG_LIBDIR` for cross-arch builds. |
| **util-linux over macOS uuid stubs** | `brew install util-linux` gives a complete `libuuid` with a proper `.pc` file. macOS's built-in uuid stubs are insufficient for autotools-based configure scripts. |
| **`--host` must use `darwin` not `macos`** | c-aff4's configure.ac only recognises `darwin*` in the host OS pattern. `macos` causes `OS not supported` error. |
| **spdlog-src path is architecture-independent** | Header-only consumption means the same include path works for both arm64 and x86_64. No per-arch spdlog build needed. |
