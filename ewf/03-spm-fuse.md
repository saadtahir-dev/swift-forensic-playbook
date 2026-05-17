# Wrapping libewf in a Swift Package (with macFUSE)

> Adds macFUSE support to the libewf Swift Package for mounting:
>
> * `.E01`
> * `.Ex01`
> * split RAW images
>
> as virtual filesystems on macOS.

This guide produces:

```text
universal/prefix/
├── bin/
├── include/
├── lib/
└── share/
```

with:

* arm64 support
* x86_64 support
* universal static libraries
* universal CLI binaries

---

# Important

macFUSE cannot be bundled inside your app.

Users must install macFUSE separately because it requires:

* system extensions
* kernel integration
* privileged installation

Official site:

[macFUSE](https://macfuse.github.io/?utm_source=chatgpt.com)

---

# Root Paths

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # build artifacts/output
SPM_ROOT="${HOME}/forensic-spm"      # Swift Package projects
```

---

# Prerequisites

Install build tools:

```bash
brew install autoconf automake libtool gettext
```

Install macFUSE:

```bash
brew install --cask macfuse
```

macFUSE installation may require approval in:

```text
System Settings
→ Privacy & Security
→ Allow System Extension
```

macFUSE installer/runtime details: ([macfuse.github.io][1])

---

# Step 1 — Clone Repository

```bash
mkdir -p "${BUILD_ROOT}"

cd "${BUILD_ROOT}"

git clone https://github.com/libyal/libewf.git LIB-EWF-SPM-V2

cd LIB-EWF-SPM-V2
```

---

# Step 2 — Generate Build Files

```bash
bash autogen.sh
```

If `make distclean` is run later:

```bash
bash autogen.sh
```

must be executed again before configuring.

---

# Step 3 — Verify macFUSE Installation

Verify headers:

```bash
find /usr/local/include -name "fuse.h"
```

Expected:

```text
/usr/local/include/fuse.h
```

Verify libraries:

```bash
ls /usr/local/lib/libfuse*
```

Expected:

```text
libfuse.dylib
libfuse3.dylib
```

---

# Step 4 — Critical macFUSE 5.x Compatibility Fix

---

## Root Cause

macFUSE 5.x uses Darwin-specific callback types:

```c
fuse_darwin_fill_dir_t
struct fuse_darwin_attr
```

libewf/libsmraw upstream code expects:

```c
fuse_fill_dir_t
struct stat
```

This creates ABI mismatches during compilation.

Common errors:

```text
incompatible function pointer types
fuse_fill_dir_t vs fuse_darwin_fill_dir_t
```

```text
struct stat * vs struct fuse_darwin_attr *
```

macFUSE Darwin API extensions are documented in modern macFUSE/libfuse3 releases. ([macfuse.github.io][1])

---

# Correct Fix Strategy

DO NOT globally replace:

```c
struct stat
```

with:

```c
struct fuse_darwin_attr
```

That breaks internal POSIX logic.

Correct architecture:

| Layer                | Type                      |
| -------------------- | ------------------------- |
| Internal logic       | `struct stat`             |
| FUSE API boundary    | `struct fuse_darwin_attr` |
| Translation boundary | explicit conversion       |

---

# Step 5 — Apply Darwin FUSE Patches

---

## 5a — Fix `fuse_fill_dir_t`

Inside:

```text
ewftools/mount_fuse.h
ewftools/mount_fuse.c
smrawtools/mount_fuse.h
smrawtools/mount_fuse.c
```

replace:

```c
fuse_fill_dir_t
```

with:

```c
fuse_darwin_fill_dir_t
```

Example:

```c
int mount_fuse_filldir(
     void *buffer,
     fuse_darwin_fill_dir_t filler,
```

---

## 5b — Fix FUSE Callback Signatures Only

Only FUSE-facing callback signatures should use:

```c
struct fuse_darwin_attr *
```

Example:

```c
int mount_fuse_getattr(
     const char *path,
     struct fuse_darwin_attr *stat_info,
     struct fuse_file_info *file_info )
```

DO NOT change:

* internal structs
* internal local variables
* POSIX stat handling

---

## 5c — Keep Internal Logic as `struct stat`

This remains unchanged:

```c
struct stat st;
struct stat *stat_info = &st;
```

Internal field access must remain:

```c
st_size
st_mode
st_uid
st_gid
```

NOT:

```c
fa_size
fa_mode
```

---

## 5d — Add Darwin Translation Layer

Inside `mount_fuse_filldir()`:

```c
#if defined( __APPLE__ ) && defined( FUSE_DARWIN_ENABLE_EXTENSIONS )

struct fuse_darwin_attr fa;
struct fuse_darwin_attr *darwin_stat_info = &fa;

#else

struct stat *stat_info = &st;

#endif
```

Before calling `filler()`:

```c
filler(
    buffer,
    name,
    darwin_stat_info,
```

---

## 5e — Preserve Internal `mount_fuse_set_stat_info()`

This function must continue using:

```c
struct stat *
```

NOT:

```c
struct fuse_darwin_attr *
```

Correct:

```c
int mount_fuse_set_stat_info(
     struct stat *stat_info,
```

---

# Step 6 — Build arm64

```bash
BUILD_ROOT="${HOME}/forensic-libs"
SPM_ROOT="${HOME}/forensic-spm"

PREFIX="${BUILD_ROOT}/LIB-EWF-SPM-V2/build/arm64/prefix"

mkdir -p "${PREFIX}"
```

Configure:

```bash
./configure \
  --enable-static \
  --disable-shared \
  --with-libfuse \
  CC="clang -arch arm64" \
  CFLAGS="-arch arm64 -I/usr/local/include/fuse3 -I/usr/local/include" \
  LDFLAGS="-L/usr/local/lib" \
  --host=aarch64-apple-macos \
  --prefix="${PREFIX}"
```

Build:

```bash
make -j$(sysctl -n hw.ncpu)
make install
```

---

# Step 7 — Verify arm64 Build

Verify FUSE detection:

```bash
grep HAVE_LIBFUSE common/config.h
```

Expected:

```text
#define HAVE_LIBFUSE3 1
```

Verify ewfmount:

```bash
file "${PREFIX}/bin/ewfmount"
```

Expected:

```text
Mach-O 64-bit executable arm64
```

Verify linkage:

```bash
otool -L "${PREFIX}/bin/ewfmount"
```

Expected:

```text
/usr/local/lib/libfuse3.dylib
```

---

# Step 8 — Build x86_64

Clean previous build:

```bash
make distclean
bash autogen.sh
```

Set prefix:

```bash
PREFIX="${BUILD_ROOT}/LIB-EWF-SPM-V2/build/x86_64/prefix"

mkdir -p "${PREFIX}"
```

Configure:

```bash
./configure \
  --enable-static \
  --disable-shared \
  --with-libfuse \
  CC="clang -arch x86_64" \
  CFLAGS="-arch x86_64 -I/usr/local/include/fuse3 -I/usr/local/include" \
  LDFLAGS="-L/usr/local/lib" \
  --host=x86_64-apple-macos \
  --prefix="${PREFIX}"
```

Build:

```bash
make -j$(sysctl -n hw.ncpu)
make install
```

---

# Step 9 — Build Missing libuna (x86_64)

Sometimes `libuna.a` is missing from x86 builds.

Build separately:

```bash
cd libuna

bash autogen.sh

./configure \
  --enable-static \
  --disable-shared \
  CC="clang -arch x86_64" \
  CFLAGS="-arch x86_64" \
  --host=x86_64-apple-macos \
  --prefix="${BUILD_ROOT}/LIB-EWF-SPM-V2/build/x86_64/prefix"

make -j$(sysctl -n hw.ncpu)
make install
```

Verify:

```bash
ls "${BUILD_ROOT}/LIB-EWF-SPM-V2/build/x86_64/prefix/lib/libuna.a"
```

---

# Step 10 — Create Universal SDK

Create folders:

```bash
mkdir -p build/universal/prefix/{bin,include,lib,share}
```

---

## Universal Libraries

```bash
for LIB in build/arm64/prefix/lib/*.a; do

  NAME="$(basename "$LIB")"

  lipo -create \
    "build/arm64/prefix/lib/${NAME}" \
    "build/x86_64/prefix/lib/${NAME}" \
    -output "build/universal/prefix/lib/${NAME}"

done
```

---

## Universal Binaries

```bash
for BIN in build/arm64/prefix/bin/*; do

  NAME="$(basename "$BIN")"

  lipo -create \
    "build/arm64/prefix/bin/${NAME}" \
    "build/x86_64/prefix/bin/${NAME}" \
    -output "build/universal/prefix/bin/${NAME}"

done
```

---

## Copy Headers + Share

```bash
cp -R build/arm64/prefix/include/* \
      build/universal/prefix/include/

cp -R build/arm64/prefix/share/* \
      build/universal/prefix/share/
```

---

# Step 11 — Verify Universal SDK

Verify libraries:

```bash
for LIB in build/universal/prefix/lib/*.a; do
  lipo -info "$LIB"
done
```

Expected:

```text
Architectures in the fat file: x86_64 arm64
```

Verify binaries:

```bash
file build/universal/prefix/bin/ewfmount
```

Expected:

```text
Mach-O universal binary
```

---

# Step 12 — Real Mount Verification

Create mount point:

```bash
mkdir -p /tmp/ewf_mount
```

Mount sample image:

```bash
build/universal/prefix/bin/ewfmount \
  "/path/to/image.E01" \
  /tmp/ewf_mount
```

Expected:

```text
/tmp/ewf_mount/ewf1
```

Example verified result from this build:

```text
-r--r--r--  17G  ewf1
```

---

# Step 13 — SwiftPM Integration

In `Package.swift` add:

```swift
.linkedLibrary("fuse")
```

Example:

```swift
.linkerSettings([
    .linkedLibrary("fuse"),
    .linkedLibrary("z"),
    .linkedLibrary("bz2"),
    .linkedLibrary("iconv")
])
```

---

# Step 14 — Runtime Requirements

Users must install:

```bash
brew install --cask macfuse
```

before mounting works.

---

# Hardened Runtime + Notarization

If bundling CLI tools:

* `ewfmount`
* `smrawmount`
* `ewfinfo`

inside a macOS app:

they must be re-signed inside the final `.app` bundle.

Example:

```bash
codesign \
  --force \
  --deep \
  --options runtime \
  --timestamp \
  --sign "Developer ID Application: YOUR_TEAM" \
  "/path/to/bundle/bin/ewfmount"
```

---

# Common Errors

| Error                                       | Cause                                | Fix                            |
| ------------------------------------------- | ------------------------------------ | ------------------------------ |
| `fuse_fill_dir_t vs fuse_darwin_fill_dir_t` | Darwin callback mismatch             | Replace typedefs               |
| `struct stat vs fuse_darwin_attr`           | Wrong callback signature             | Change FUSE boundary only      |
| `no member named st_size`                   | Global replacement broke POSIX logic | Keep internal `struct stat`    |
| `No sub system to mount EWF format`         | Built without FUSE                   | Rebuild with `--with-libfuse`  |
| `FUSE support: no`                          | Headers not found                    | Add `/usr/local/include/fuse3` |
| `library not found for -lfuse`              | macFUSE missing                      | Install macFUSE                |
| `libuna.a missing`                          | x86 build omission                   | Build libuna separately        |
| `Undefined symbols for iconv`               | Missing system lib                   | Add `-liconv`                  |
| `Undefined symbols for compress`            | Missing zlib                         | Add `-lz`                      |
| `Undefined symbols for BZ2_*`               | Missing bzip2                        | Add `-lbz2`                    |

---

# Final SDK Layout

```text
build/universal/prefix/
├── bin/
├── include/
├── lib/
└── share/
```

Includes:

* universal forensic libraries
* universal ewf tools
* universal smraw tools
* SwiftPM-ready integration
* macFUSE runtime support

---

# References

* [macFUSE Official Site](https://macfuse.github.io/?utm_source=chatgpt.com)
* [macFUSE Releases](https://github.com/macfuse/macfuse/releases?utm_source=chatgpt.com)
* [libewf GitHub](https://github.com/libyal/libewf/issues/123?utm_source=chatgpt.com)
* [Homebrew macFUSE Formula](https://formulae.brew.sh/cask/macfuse?utm_source=chatgpt.com)

macFUSE Darwin API behavior and libfuse3 compatibility details are documented in recent macFUSE release notes. ([macfuse.github.io][2])

[1]: https://macfuse.github.io/?utm_source=chatgpt.com "macFUSE: Home"
[2]: https://macfuse.github.io/2026/04/09/macfuse-5.2.0.html?utm_source=chatgpt.com "Announcing macFUSE 5.2.0"
