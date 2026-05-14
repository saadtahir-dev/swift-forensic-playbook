# Wrapping libaff4 in a Swift Package (SPM)

> Group 3 of the forensic image format series. libaff4 is a **C++ CMake library** — the build process differs significantly from the autoconf-based `libyal` stack used in Groups 1 & 2.

---

## Prerequisites

Install build tools via Homebrew:

```bash
brew install cmake ninja pkg-config
brew install openssl@3 zlib pcre2 snappy
```

Install raptor2 and uriparser (not in default formulae, may need source builds — see below):

```bash
brew install raptor uriparser
```

Verify key paths:

```bash
# OpenSSL (brew installs keg-only, not on default path)
brew --prefix openssl@3
# Expected: /opt/homebrew/opt/openssl@3  (Apple Silicon)
#           /usr/local/opt/openssl@3      (Intel)

# raptor2
brew --prefix raptor
# Expected: /opt/homebrew/opt/raptor

# uriparser
brew --prefix uriparser
```

> **Why not system OpenSSL?** Apple ships a deprecated, stripped-down OpenSSL stub. Always use the Homebrew version for real SSL/crypto work.

---

## Dependency Overview

libaff4 links against these libraries at build time. All must be present **for both architectures** before building libaff4 itself:

| Library | Role | Source |
|---|---|---|
| `libssl` / `libcrypto` | Hashing, encryption | `brew install openssl@3` |
| `zlib` | Compression | macOS system (`/usr/lib/libz.dylib`) |
| `libraptor2` | RDF/Turtle metadata parsing | `brew install raptor` |
| `liburiparser` | URI parsing | `brew install uriparser` |
| `libpcre2` | Regex | `brew install pcre2` |
| `libsnappy` | Optional fast compression | `brew install snappy` |

> **Strategy:** For each dependency, you need both `arm64` and `x86_64` static `.a` files so the final libaff4 fat binary has no dylib dependencies. Homebrew only gives you the native architecture. For cross-arch deps, you build from source.

---

## Part A — Build Dependencies From Source (Both Architectures)

> Skip any dep that libaff4's CMake finds to be optional and you're comfortable omitting. Snappy in particular is often optional. But to produce a self-contained static binary, build them all.

### A.0 — Set up build roots

```bash
export BUILD_ROOT="$HOME/aff4_build"
mkdir -p "$BUILD_ROOT"/{arm64,x86_64,universal}/{lib,include}
mkdir -p "$BUILD_ROOT"/deps/{arm64,x86_64}
```

---

### A.1 — OpenSSL

OpenSSL has its own cross-compile flags. Do **not** use CMake for it.

```bash
cd "$BUILD_ROOT"
git clone --depth=1 https://github.com/openssl/openssl.git openssl-src
cd openssl-src

# arm64
./Configure darwin64-arm64-cc \
  no-shared \
  --prefix="$BUILD_ROOT/deps/arm64/openssl" \
  -mmacosx-version-min=12.0
make -j$(sysctl -n hw.ncpu) && make install_sw

# x86_64
make distclean

./Configure darwin64-x86_64-cc \
  no-shared \
  --prefix="$BUILD_ROOT/deps/x86_64/openssl" \
  -mmacosx-version-min=12.0
make -j$(sysctl -n hw.ncpu) && make install_sw
```

Verify:

```bash
file "$BUILD_ROOT/deps/arm64/openssl/lib/libssl.a"
# Expected: ... current ar archive

lipo -info "$BUILD_ROOT/deps/arm64/openssl/lib/libssl.a"
# Expected: Non-fat file ... architecture: arm64
```

---

### A.2 — zlib

Use the macOS system zlib — it is stable, ABI-compatible, and ships with both architectures baked in. **No build needed.** Reference it at `/usr/lib/libz.dylib` or `/usr/lib/libz.tbd`.

> Exception: If you want a fully static binary with no dylib deps at all, build zlib from source:

```bash
cd "$BUILD_ROOT"
git clone --depth=1 https://github.com/madler/zlib.git zlib-src

# arm64
cd zlib-src
CFLAGS="-arch arm64 -mmacosx-version-min=12.0" \
  ./configure --static --prefix="$BUILD_ROOT/deps/arm64/zlib"
make -j$(sysctl -n hw.ncpu) && make install
make distclean

# x86_64
CFLAGS="-arch x86_64 -mmacosx-version-min=12.0" \
  ./configure --static --prefix="$BUILD_ROOT/deps/x86_64/zlib"
make -j$(sysctl -n hw.ncpu) && make install
```

---

### A.3 — PCRE2

```bash
cd "$BUILD_ROOT"
git clone --depth=1 https://github.com/PCRE2Project/pcre2.git pcre2-src

# arm64
cmake -S pcre2-src -B pcre2-arm64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=arm64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=12.0 \
  -DBUILD_SHARED_LIBS=OFF \
  -DPCRE2_BUILD_PCRE2_8=ON \
  -DPCRE2_SUPPORT_UTF=ON \
  -DCMAKE_INSTALL_PREFIX="$BUILD_ROOT/deps/arm64/pcre2"
cmake --build pcre2-arm64 -j$(sysctl -n hw.ncpu)
cmake --install pcre2-arm64

# x86_64
cmake -S pcre2-src -B pcre2-x86_64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=x86_64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=12.0 \
  -DBUILD_SHARED_LIBS=OFF \
  -DPCRE2_BUILD_PCRE2_8=ON \
  -DPCRE2_SUPPORT_UTF=ON \
  -DCMAKE_INSTALL_PREFIX="$BUILD_ROOT/deps/x86_64/pcre2"
cmake --build pcre2-x86_64 -j$(sysctl -n hw.ncpu)
cmake --install pcre2-x86_64
```

---

### A.4 — uriparser

```bash
cd "$BUILD_ROOT"
git clone --depth=1 https://github.com/uriparser/uriparser.git uriparser-src

# arm64
cmake -S uriparser-src -B uriparser-arm64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=arm64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=12.0 \
  -DBUILD_SHARED_LIBS=OFF \
  -DURIPARSER_BUILD_TESTS=OFF \
  -DURIPARSER_BUILD_DOCS=OFF \
  -DCMAKE_INSTALL_PREFIX="$BUILD_ROOT/deps/arm64/uriparser"
cmake --build uriparser-arm64 -j$(sysctl -n hw.ncpu)
cmake --install uriparser-arm64

# x86_64
cmake -S uriparser-src -B uriparser-x86_64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=x86_64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=12.0 \
  -DBUILD_SHARED_LIBS=OFF \
  -DURIPARSER_BUILD_TESTS=OFF \
  -DURIPARSER_BUILD_DOCS=OFF \
  -DCMAKE_INSTALL_PREFIX="$BUILD_ROOT/deps/x86_64/uriparser"
cmake --build uriparser-x86_64 -j$(sysctl -n hw.ncpu)
cmake --install uriparser-x86_64
```

---

### A.5 — Snappy

```bash
cd "$BUILD_ROOT"
git clone --depth=1 https://github.com/google/snappy.git snappy-src

# arm64
cmake -S snappy-src -B snappy-arm64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=arm64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=12.0 \
  -DBUILD_SHARED_LIBS=OFF \
  -DSNAPPY_BUILD_TESTS=OFF \
  -DSNAPPY_BUILD_BENCHMARKS=OFF \
  -DCMAKE_INSTALL_PREFIX="$BUILD_ROOT/deps/arm64/snappy"
cmake --build snappy-arm64 -j$(sysctl -n hw.ncpu)
cmake --install snappy-arm64

# x86_64
cmake -S snappy-src -B snappy-x86_64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=x86_64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=12.0 \
  -DBUILD_SHARED_LIBS=OFF \
  -DSNAPPY_BUILD_TESTS=OFF \
  -DSNAPPY_BUILD_BENCHMARKS=OFF \
  -DCMAKE_INSTALL_PREFIX="$BUILD_ROOT/deps/x86_64/snappy"
cmake --build snappy-x86_64 -j$(sysctl -n hw.ncpu)
cmake --install snappy-x86_64
```

---

### A.6 — raptor2

raptor2 uses autoconf but the upstream git snapshot has several incompatibilities with modern macOS tooling that require fixes before it will build. Follow this sequence exactly.

#### A.6.1 — Install required tooling

```bash
brew install gtk-doc bison flex
export PATH="/opt/homebrew/opt/bison/bin:/opt/homebrew/opt/flex/bin:$PATH"
```

Verify:

```bash
gtkdocize --version
bison --version
flex --version
```

> **Why all three?** `autoreconf` requires `gtkdocize` even with `--disable-gtk-doc`. `bison` and `flex` are needed to generate `turtle_lexer.c`, `turtle_parser.c`, and `parsedate.c` — the git snapshot does not include these generated files.

#### A.6.2 — Clone

```bash
cd "$BUILD_ROOT"
git clone --depth=1 https://github.com/dajobe/raptor.git raptor-src
cd raptor-src
```

#### A.6.3 — Fix broken gtk-doc integration

The repo has inconsistent gtk-doc references that break `autoreconf`. Remove them:

```bash
sed -i.bak '/gtk-doc\.make/d' Makefile.am
sed -i.bak '/gtk-doc\.make/d' docs/Makefile.am
```

Verify only harmless references remain:

```bash
grep -R "gtk-doc.make" . --exclude="*.bak"
# Only .gitignore and autogen.sh references are fine
```

#### A.6.4 — Fix modern automake compatibility

Modern automake rejects `+=` before a variable is initialized. Patch `docs/Makefile.am`:

```bash
sed -i.bak '/EXTRA_DIST+=/i\
EXTRA_DIST =
' docs/Makefile.am

sed -i.bak '/DISTCLEANFILES+=/i\
DISTCLEANFILES =
' docs/Makefile.am
```

#### A.6.5 — Initialize libfsp submodule

The shallow clone does not populate submodules. raptor2 requires `libfsp`:

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

#### A.6.6 — Regenerate build system

Use `autoreconf` directly instead of `autogen.sh` — it is more robust with modern tooling:

```bash
autoreconf -fi
```

Verify critical generated files exist:

```bash
ls configure
ls docs/Makefile.in
ls scripts/Makefile.in
# All three must be present
```

#### A.6.7 — Generate missing lexer/parser sources

The git snapshot does not include generated C sources. Generate them manually:

```bash
cd src
flex -o turtle_lexer.c turtle_lexer.l
bison -o turtle_parser.c turtle_parser.y
bison -o parsedate.c parsedate.y
cd ..
```

#### A.6.8 — Patch generated lexer for gnu11 compatibility

The flex-generated lexer has a `setjmp.h` include ordering issue under `gnu23`. Two fixes needed:

First, insert the missing include at the top of the generated lexer:

```bash
sed -i.bak '1i\
#include <setjmp.h>
' src/turtle_lexer.c
```

Verify:

```bash
head -5 src/turtle_lexer.c
# Expected: #include <setjmp.h> as first line
```

Second, use `-std=gnu11` instead of the default `gnu23` in the configure flags (see below).

#### A.6.9 — Build arm64

```bash
./configure --enable-static --disable-shared \
  CC="clang -arch arm64" \
  CFLAGS="-std=gnu11 -arch arm64 -mmacosx-version-min=12.0" \
  --host=aarch64-apple-macos \
  --prefix="$BUILD_ROOT/deps/arm64/raptor" \
  --without-www

make -j1 V=1 && make install
```

> **`-std=gnu11`** is required. The default `gnu23` exposes flex-generated code incompatibilities that cause `jmp_buf`, `setjmp`, and `longjmp` errors.

> **`-j1`** is intentional. Parallel builds can trigger race conditions in the raptor autotools setup. Single-threaded is more reliable here.

> **`--without-www`** disables libcurl. raptor uses it for URI fetching which you don't need for forensic imaging.

Verify:

```bash
lipo -info "$BUILD_ROOT/deps/arm64/raptor/lib/libraptor2.a"
# Expected: Non-fat file ... is architecture: arm64
```

#### A.6.10 — Build x86_64

```bash
make distclean

# Re-generate lexer/parser for clean state
cd src
flex -o turtle_lexer.c turtle_lexer.l
bison -o turtle_parser.c turtle_parser.y
bison -o parsedate.c parsedate.y
sed -i.bak '1i\
#include <setjmp.h>
' turtle_lexer.c
cd ..

./configure --enable-static --disable-shared \
  CC="clang -arch x86_64" \
  CFLAGS="-std=gnu11 -arch x86_64 -mmacosx-version-min=12.0" \
  --host=x86_64-apple-macos \
  --prefix="$BUILD_ROOT/deps/x86_64/raptor" \
  --without-www

make -j1 V=1 && make install
```

Verify:

```bash
lipo -info "$BUILD_ROOT/deps/x86_64/raptor/lib/libraptor2.a"
# Expected: Non-fat file ... is architecture: x86_64
```

#### A.6.11 — Create universal binary

```bash
mkdir -p "$BUILD_ROOT/deps/universal/raptor/lib"

lipo -create \
  "$BUILD_ROOT/deps/arm64/raptor/lib/libraptor2.a" \
  "$BUILD_ROOT/deps/x86_64/raptor/lib/libraptor2.a" \
  -output "$BUILD_ROOT/deps/universal/raptor/lib/libraptor2.a"

lipo -info "$BUILD_ROOT/deps/universal/raptor/lib/libraptor2.a"
# Expected: Architectures in the fat file: ... are: x86_64 arm64
```

