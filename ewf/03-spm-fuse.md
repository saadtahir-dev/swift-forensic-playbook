# Wrapping libewf in a Swift Package (with macFUSE)

> Adds macFUSE support to the libewf SPM package for mounting EWF images as virtual filesystems.
> macFUSE must be installed by the user — it cannot be bundled (requires a kernel extension).

---

## Prerequisites

- macFUSE installed by the user:

```bash
brew install --cask macfuse
```

Verify:

```bash
ls /Library/Filesystems/macfuse.fs
# Expected: Contents
```

Find FUSE headers:

```bash
find /usr/local -name "fuse.h" 2>/dev/null
# Expected: /usr/local/include/fuse.h
```

> **Note:** macFUSE 5.x puts headers at `/usr/local/include/fuse.h`. The library is at `/usr/local/lib/libfuse.dylib`.

---

## macFUSE 5.x Compatibility — Critical Fix

macFUSE 5.x uses Darwin-specific FUSE APIs that are **not ABI-compatible** with the standard POSIX/FUSE3 types that libewf was written against.

### The Problem

Build fails in `ewftools/` with:

```
incompatible function pointer types:
  fuse_fill_dir_t vs fuse_darwin_fill_dir_t
  struct stat * vs struct fuse_darwin_attr *
```

### Why a Simple Replacement Fails

The naive fix of globally replacing `struct stat *` with `struct fuse_darwin_attr *` introduces a new failure:

```
no member named 'st_size' in 'struct fuse_darwin_attr'
no member named 'st_mode'
no member named 'st_nlink'
```

This is because `fuse_darwin_attr` has completely different field names and memory layout from `struct stat`. A global replacement creates an ABI inconsistency:

| Component | Type | Problem |
|---|---|---|
| Function signature | `struct fuse_darwin_attr *` | Required by macFUSE 5.x |
| Internal implementation | POSIX `struct stat` logic | Fields like `st_size`, `st_mode` |
| Field access | `st_size`, `st_mode` etc. | Don't exist on `fuse_darwin_attr` |

### Correct Architecture — Boundary Layer Approach

```
Internal logic      → struct stat        (POSIX fields unchanged)
FUSE API boundary   → fuse_darwin_attr   (macFUSE 5.x callback signatures)
Bridge layer        → explicit conversion (stat → fuse_darwin_attr)
```

---

## Step 1 — Apply Patches

### 1a — Fix `fuse_fill_dir_t` (valid simple replacement)

This is only a function pointer typedef — no struct layout issue:

```bash
sed -i '' 's/fuse_fill_dir_t filler/fuse_darwin_fill_dir_t filler/g' ewftools/mount_fuse.c
sed -i '' 's/fuse_fill_dir_t filler/fuse_darwin_fill_dir_t filler/g' ewftools/mount_fuse.h
sed -i '' 's/fuse_fill_dir_t filler/fuse_darwin_fill_dir_t filler/g' ewftools/ewfmount.c
```

### 1b — Fix function signatures only (NOT internal variables)

Only FUSE-facing callback signatures use `fuse_darwin_attr`. Internal logic stays as `struct stat`:

```bash
# Fix function signatures in headers
sed -i '' 's/struct stat \*stat_info/struct fuse_darwin_attr *stat_info/g' ewftools/mount_fuse.h

# Fix function signatures in source (lines with parameter declarations)
# Do NOT replace internal variable declarations
```

### 1c — Add translation layer in `mount_fuse.c`

Inside each `getattr`-style callback, after computing values into a local `struct stat`, add an explicit conversion to `fuse_darwin_attr`:

```c
// Internal computation uses struct stat
struct stat posix_stat;
memset(&posix_stat, 0, sizeof(struct stat));

// ... fill posix_stat fields ...
posix_stat.st_size  = file_size;
posix_stat.st_mode  = S_IFREG | 0444;
posix_stat.st_nlink = 1;
posix_stat.st_uid   = getuid();
posix_stat.st_gid   = getgid();

// Translation layer: struct stat → struct fuse_darwin_attr
stat_info->fa_size       = posix_stat.st_size;
stat_info->fa_mode       = posix_stat.st_mode;
stat_info->fa_nlink      = posix_stat.st_nlink;
stat_info->fa_uid        = posix_stat.st_uid;
stat_info->fa_gid        = posix_stat.st_gid;
stat_info->fa_atime      = posix_stat.st_atimespec;
stat_info->fa_mtime      = posix_stat.st_mtimespec;
stat_info->fa_ctime      = posix_stat.st_ctimespec;
```

**Field mapping reference:**

| POSIX (`struct stat`) | Darwin (`fuse_darwin_attr`) |
|---|---|
| `st_size` | `fa_size` |
| `st_mode` | `fa_mode` |
| `st_nlink` | `fa_nlink` |
| `st_uid` | `fa_uid` |
| `st_gid` | `fa_gid` |
| `st_atimespec` | `fa_atime` |
| `st_mtimespec` | `fa_mtime` |
| `st_ctimespec` | `fa_ctime` |

---

## Step 2 — Build libewf with FUSE Support

If the repo was cloned without generated autotools files:

```bash
./autogen.sh
```

Then configure and build:

```bash
# arm64
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

./configure --enable-static --disable-shared \
  --with-libfuse \
  CC="clang -arch arm64" \
  CFLAGS="-arch arm64 -I/usr/local/include" \
  LDFLAGS="-L/usr/local/lib" \
  --host=aarch64-apple-macos \
  --prefix="${BUILD_ROOT}/LIB-EWF-SPM/arm64"

make -j$(sysctl -n hw.ncpu) && make install
```

Verify FUSE was detected in configure output:

```
FUSE support:    libfuse3    ← must say this, not 'no'
```

```bash
# x86_64
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

make distclean && ./autogen.sh && \
./configure --enable-static --disable-shared \
  --with-libfuse \
  CC="clang -arch x86_64" \
  CFLAGS="-arch x86_64 -I/usr/local/include" \
  LDFLAGS="-L/usr/local/lib" \
  --host=x86_64-apple-macos \
  --prefix="${BUILD_ROOT}/LIB-EWF-SPM/x86_64"

make -j$(sysctl -n hw.ncpu) && make install
```

Create universal binary:

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

lipo -create \
  "${BUILD_ROOT}/LIB-EWF-SPM/arm64/lib/libewf.a" \
  "${BUILD_ROOT}/LIB-EWF-SPM/x86_64/lib/libewf.a" \
  -output "${BUILD_ROOT}/LIB-EWF-SPM/libewf.a"

lipo -info "${BUILD_ROOT}/LIB-EWF-SPM/libewf.a"
# Expected: Architectures in the fat file: ... are: x86_64 arm64
```

---

## Step 3 — Package.swift

The only difference from the basic guide is adding `-lfuse`:

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "Libewf",
    platforms: [.macOS(.v13)],
    products: [
        .library(name: "Libewf", targets: ["Libewf"]),
    ],
    targets: [
        .target(
            name: "Clibewf",
            path: "Sources/Clibewf",
            sources: ["placeholder.c"],
            publicHeadersPath: "include",
            linkerSettings: [
                .linkedLibrary("ewf"),
                .linkedLibrary("z"),
                .linkedLibrary("bz2"),
                .linkedLibrary("iconv"),
                .linkedLibrary("fuse"),      // ← FUSE
                .unsafeFlags(["-LSources/Clibewf"])
            ]
        ),
        .target(
            name: "Libewf",
            dependencies: ["Clibewf"],
            path: "Sources/Libewf"
        ),
        .testTarget(
            name: "LibewfTests",
            dependencies: ["Libewf"],
            path: "Tests/LibewfTests"
        ),
    ]
)
```

---

## Step 4 — Build and Test

```bash
swift build   # Expected: Build complete!
swift test    # Expected: Test Suite 'All tests' passed
```

---

## Bundling ewf* Binaries and Hardened Runtime

If you bundle the `ewf*` CLI tools (`ewfmount`, `ewfinfo`, etc.) as resources in a macOS app for distribution, Apple requires them to be signed with Hardened Runtime support.

**Key lessons:**

| Lesson | Detail |
|---|---|
| Sign in the bundle, not the source | Xcode strips signatures on copy — always re-sign post-copy inside the built bundle |
| `TARGET_BUILD_DIR` vs `BUILT_PRODUCTS_DIR` | During Archive, only `TARGET_BUILD_DIR` points to the real app location |
| SPM resource bundles are nested | Binaries land inside `.bundle` containers, not flat in `Resources` |
| User Script Sandboxing blocks `find` | Set `ENABLE_USER_SCRIPT_SANDBOXING=NO` for scripts that read inside the app bundle |
| `--timestamp` is required | Notarization rejects signatures without a trusted timestamp |

**Sign the binaries after build:**

```bash
codesign --force --deep --options runtime \
  --sign "Developer ID Application: YOUR_TEAM (TEAMID)" \
  --timestamp \
  "/path/to/your.app/Contents/Resources/libewf_ClibewfResources.bundle/Contents/Resources/bin/"*
```

**In your Xcode Run Script phase (Archive):**

```bash
BUNDLE_LIBEWF="${TARGET_BUILD_DIR}/${WRAPPER_NAME}/Contents/Resources/libewf_ClibewfResources.bundle/Contents/Resources/bin"

find "$BUNDLE_LIBEWF" -type f | while read bin; do
  codesign --force --options runtime \
    --sign "Developer ID Application: YOUR_TEAM (TEAMID)" \
    --timestamp \
    "$bin"
done
```

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `incompatible function pointer types: fuse_fill_dir_t` | macFUSE 5.x darwin type mismatch in signatures | Replace with `fuse_darwin_fill_dir_t` in signatures only |
| `no member named 'st_size' in fuse_darwin_attr` | Global struct replacement broke internal logic | Keep internal variables as `struct stat`, add translation layer |
| `FUSE support: no` in configure | FUSE headers not found | Add `-I/usr/local/include` to `CFLAGS` |
| `library not found for -lfuse` | macFUSE not installed | `brew install --cask macfuse` |
| `No sub system to mount EWF format` | Library built without FUSE | Rebuild with `--with-libfuse` |
| `cannot find input file: 'Makefile.in'` | Git clone missing generated autotools files | Run `./autogen.sh` first |
| Notarization rejected — Hardened Runtime | ewf* binaries not signed correctly | Re-sign inside bundle with `--options runtime --timestamp` |
| `Sandbox: find deny(1) file-read-data` | User Script Sandboxing enabled | Set `ENABLE_USER_SCRIPT_SANDBOXING=NO` in Build Settings |
| `syntax error near unexpected token '('` | Parentheses in `--prefix` path | Use a path with no special characters |
