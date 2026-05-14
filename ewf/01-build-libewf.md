# Building libewf as a Universal Static Library

> Produces a universal `libewf.a` (arm64 + x86_64) for use in a Swift Package.

---

## Prerequisites

```bash
brew install autoconf automake libtool gettext
```

---

## Step 1 — Clone and Generate Build Files

```bash
git clone --depth=1 https://github.com/libyal/libewf.git
cd libewf
./autogen.sh
```

> If `autogen.sh` succeeds you'll see output from `glibtoolize` copying `.m4` macros. Warnings about obsolete macros are harmless.

> **Important:** If `make distclean` is run between builds, re-run `./autogen.sh` before configuring again — it wipes the generated files.

---

## Step 2 — Build for arm64

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

./configure --enable-static --disable-shared \
  CC="clang -arch arm64" \
  CFLAGS="-arch arm64" \
  --host=aarch64-apple-macos \
  --prefix="${BUILD_ROOT}/LIB-EWF-SPM/arm64"

make -j$(sysctl -n hw.ncpu)
make install
```

Output: `${BUILD_ROOT}/LIB-EWF-SPM/arm64/lib/libewf.a`

---

## Step 3 — Build for x86_64

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

./autogen.sh

./configure --enable-static --disable-shared \
  CC="clang -arch x86_64" \
  CFLAGS="-arch x86_64" \
  --host=x86_64-apple-macos \
  --prefix="${BUILD_ROOT}/LIB-EWF-SPM/x86_64"

make -j$(sysctl -n hw.ncpu)
make install
```

Output: `${BUILD_ROOT}/LIB-EWF-SPM/x86_64/lib/libewf.a`

---

## Step 4 — Create Universal Fat Binary

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

lipo -create \
  "${BUILD_ROOT}/LIB-EWF-SPM/arm64/lib/libewf.a" \
  "${BUILD_ROOT}/LIB-EWF-SPM/x86_64/lib/libewf.a" \
  -output "${BUILD_ROOT}/LIB-EWF-SPM/libewf.a"

lipo -info "${BUILD_ROOT}/LIB-EWF-SPM/libewf.a"
# Expected: Architectures in the fat file: libewf.a are: x86_64 arm64
```

---

## Step 5 — Test the Binaries

```c
// test.c
#include <stdio.h>
#include "libewf.h"

int main() {
    const char *version = libewf_get_version();
    printf("libewf version: %s\n", version);
    return 0;
}
```

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

# arm64
clang -arch arm64 \
  -I"${BUILD_ROOT}/LIB-EWF-SPM/arm64/include" \
  -L"${BUILD_ROOT}/LIB-EWF-SPM/arm64/lib" \
  -lewf -lz -lbz2 \
  -o test_arm64 test.c && ./test_arm64

# x86_64
clang -arch x86_64 \
  -I"${BUILD_ROOT}/LIB-EWF-SPM/x86_64/include" \
  -L"${BUILD_ROOT}/LIB-EWF-SPM/x86_64/lib" \
  -lewf -lz -lbz2 \
  -o test_x86 test.c && ./test_x86
```

> Running the x86_64 binary on Apple Silicon works via Rosetta 2 — valid for verification.

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `cannot find input file: 'Makefile.in'` | Build files not generated or wiped by `make distclean` | Run `./autogen.sh` again |
| `Undefined symbols` for `-lz` or `-lbz2` | Missing system lib flags | Add `-lz -lbz2` to link command |
| `ld: warning: built for newer macOS version` | SDK version mismatch | Harmless, safe to ignore |
