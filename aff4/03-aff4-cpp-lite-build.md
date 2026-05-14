## Part B — Build libaff4

### B.0 — Library selection

The original `https://github.com/aff4/aff4.git` repository no longer exists. As of 2025, the `aff4` GitHub organisation has two viable C++ implementations:

| Repo | Type | Build system | Use case |
|---|---|---|---|
| `aff4/aff4-cpp-lite` | Lightweight C/C++ reader | autoconf | Read-only mounting and enumeration |
| `Velocidex/c-aff4` | Full read+write implementation | autoconf | Full forensic imaging workflow |

For read-only AFF4 mounting and enumeration, **`aff4/aff4-cpp-lite`** is the correct choice. It is actively maintained (last updated Feb 2026), has a clean C API suitable for Swift interop, and its dep tree is a subset of what you have already built.

> **`aff4-cpp-lite` dep tree vs what is already built:**
> - zlib ✅, snappy ✅, raptor2 ✅ — already done in Part A
> - lz4 ❌ — one new dep, build first (see B.1)
> - openssl — used by tests/examples only, not the core library

#### B.0.1 — Build lz4 (new dep)

```bash
cd "$BUILD_ROOT"
git clone --depth=1 https://github.com/lz4/lz4.git lz4-src

# arm64
cmake -S lz4-src/build/cmake -B lz4-arm64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=arm64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=12.0 \
  -DBUILD_SHARED_LIBS=OFF \
  -DLZ4_BUILD_CLI=OFF \
  -DLZ4_BUILD_LEGACY_LZ4C=OFF \
  -DCMAKE_INSTALL_PREFIX="$BUILD_ROOT/deps/arm64/lz4"
cmake --build lz4-arm64 -j$(sysctl -n hw.ncpu)
cmake --install lz4-arm64

# x86_64
cmake -S lz4-src/build/cmake -B lz4-x86_64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=x86_64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=12.0 \
  -DBUILD_SHARED_LIBS=OFF \
  -DLZ4_BUILD_CLI=OFF \
  -DLZ4_BUILD_LEGACY_LZ4C=OFF \
  -DCMAKE_INSTALL_PREFIX="$BUILD_ROOT/deps/x86_64/lz4"
cmake --build lz4-x86_64 -j$(sysctl -n hw.ncpu)
cmake --install lz4-x86_64
```

Verify:

```bash
lipo -info "$BUILD_ROOT/deps/arm64/lz4/lib/liblz4.a"
lipo -info "$BUILD_ROOT/deps/x86_64/lz4/lib/liblz4.a"
# Expected: arm64 and x86_64 respectively
```

---

### B.1 — Clone and bootstrap

```bash
cd "$BUILD_ROOT"
git clone --depth=1 https://github.com/aff4/aff4-cpp-lite.git aff4-src
cd aff4-src
autoreconf -fi
```

The `autoreconf` output will show obsolete macro warnings — these are harmless.

---

### B.2 — Build for arm64

The arm64 build goes through cleanly with explicit dep paths:

```bash
./configure --enable-static --disable-shared \
  CC="clang -arch arm64" \
  CXX="clang++ -arch arm64" \
  CFLAGS="-std=gnu11 -arch arm64 -mmacosx-version-min=12.0" \
  CXXFLAGS="-std=c++11 -stdlib=libc++ -arch arm64 -mmacosx-version-min=12.0" \
  LDFLAGS="-stdlib=libc++ \
    -L$BUILD_ROOT/deps/arm64/zlib/lib \
    -L$BUILD_ROOT/deps/arm64/snappy/lib \
    -L$BUILD_ROOT/deps/arm64/lz4/lib \
    -L$BUILD_ROOT/deps/arm64/raptor/lib" \
  CPPFLAGS="-I$BUILD_ROOT/deps/arm64/zlib/include \
    -I$BUILD_ROOT/deps/arm64/snappy/include \
    -I$BUILD_ROOT/deps/arm64/lz4/include \
    -I$BUILD_ROOT/deps/arm64/raptor/include" \
  --host=aarch64-apple-macos \
  --prefix="$BUILD_ROOT/deps/arm64/aff4"

make -j1 V=1 && make install
```

Verify:

```bash
lipo -info "$BUILD_ROOT/deps/arm64/aff4/lib/libaff4.a"
# Expected: architecture: arm64
```

Also verify the static archive has no embedded dylib dependencies:

```bash
otool -L "$BUILD_ROOT/deps/arm64/aff4/lib/libaff4.a"
# Expected: only archive object member names (libaff4_la-*.o), no dylib paths
```

> **Note on example binaries:** `otool -L` on the example executables (e.g. `aff4-info`) may show Homebrew OpenSSL dylibs. This is expected and harmless — the executables are not what you are packaging. Only the `.a` archive matters, and it is clean.

---

### B.3 — Build for x86_64

> **Critical: pkg-config isolation required on Apple Silicon**
>
> On Apple Silicon, Homebrew installs packages under `/opt/homebrew` which are arm64-only. Without isolation, autotools/pkg-config resolves OpenSSL from `/opt/homebrew` during x86_64 builds, causing:
> ```
> ld: warning: ignoring file '...libssl.dylib': found architecture 'arm64', required architecture 'x86_64'
> Undefined symbols for architecture x86_64: _SHA1_Init _SHA1_Update _SHA1_Final
> ```
>
> The fix is **pkg-config isolation** — point pkg-config exclusively at the custom x86_64 dep tree. Do NOT disable pkg-config entirely; AFF4's autotools requires it and configure will fail with `configure: error: pkg-config not found`.

#### Step 1 — Isolate pkg-config to x86_64 deps

```bash
export PKG_CONFIG=/opt/homebrew/bin/pkg-config

export PKG_CONFIG_PATH="\
$BUILD_ROOT/deps/x86_64/zlib/lib/pkgconfig:\
$BUILD_ROOT/deps/x86_64/openssl/lib/pkgconfig:\
$BUILD_ROOT/deps/x86_64/lz4/lib/pkgconfig:\
$BUILD_ROOT/deps/x86_64/raptor/lib/pkgconfig:\
$BUILD_ROOT/deps/x86_64/pcre2/lib/pkgconfig:\
$BUILD_ROOT/deps/x86_64/uriparser/lib/pkgconfig"

export PKG_CONFIG_LIBDIR="$PKG_CONFIG_PATH"
```

Verify pkg-config is resolving from the correct paths:

```bash
pkg-config --libs zlib
pkg-config --libs openssl
pkg-config --libs raptor2
# All outputs must reference $BUILD_ROOT/deps/x86_64/... NOT /opt/homebrew
```

#### Step 2 — Configure and build

```bash
make distclean
autoreconf -fi

./configure --enable-static --disable-shared \
  CC="clang -arch x86_64" \
  CXX="clang++ -arch x86_64" \
  CFLAGS="-std=gnu11 -arch x86_64 -mmacosx-version-min=12.0" \
  CXXFLAGS="-std=c++11 -stdlib=libc++ -arch x86_64 -mmacosx-version-min=12.0" \
  LDFLAGS="-stdlib=libc++ \
    -L$BUILD_ROOT/deps/x86_64/zlib/lib \
    -L$BUILD_ROOT/deps/x86_64/snappy/lib \
    -L$BUILD_ROOT/deps/x86_64/lz4/lib \
    -L$BUILD_ROOT/deps/x86_64/raptor/lib" \
  CPPFLAGS="-I$BUILD_ROOT/deps/x86_64/zlib/include \
    -I$BUILD_ROOT/deps/x86_64/snappy/include \
    -I$BUILD_ROOT/deps/x86_64/lz4/include \
    -I$BUILD_ROOT/deps/x86_64/raptor/include" \
  --host=x86_64-apple-macos \
  --prefix="$BUILD_ROOT/deps/x86_64/aff4"

make -j1 V=1 && make install
```

Verify:

```bash
lipo -info "$BUILD_ROOT/deps/x86_64/aff4/lib/libaff4.a"
# Expected: architecture: x86_64
```

Verify no Homebrew contamination in the example binary:

```bash
otool -L "$BUILD_ROOT/deps/x86_64/aff4/bin/aff4-info"
# Expected: only /usr/lib/... system paths, NO /opt/homebrew paths
```

#### Step 3 — Unset pkg-config isolation after build

```bash
unset PKG_CONFIG_PATH
unset PKG_CONFIG_LIBDIR
```

> This prevents the x86_64 isolation from affecting subsequent builds.

---

### B.4 — Key lessons: Apple Silicon cross-arch builds

| Lesson | Detail |
|---|---|
| pkg-config isolation, not removal | AFF4 requires pkg-config. Disabling it breaks configure. Isolate it instead. |
| Never trust `/opt/homebrew` for x86_64 | Homebrew on Apple Silicon is arm64-only. Always override with `PKG_CONFIG_PATH`. |
| Example executables are not authoritative | `aff4-info` may link Homebrew OpenSSL dynamically. The `.a` archive is what matters — verify it separately with `otool -L`. |
| Static archive verification | `otool -L libfoo.a` shows archive object members, not dylib deps. Clean output = no embedded dylib paths. |

---

