## Part C — Create Universal Fat Binaries

You must lipo **every** `.a` that SPM will link — libaff4 plus all its static deps.

```bash
LIBS_TO_LIPO=(
  "libaff4.a"
  "openssl/lib/libcrypto.a"
  "openssl/lib/libssl.a"
  "raptor/lib/libraptor2.a"
  "uriparser/lib/liburiparser.a"
  "pcre2/lib/libpcre2-8.a"
  "snappy/lib/libsnappy.a"
)

mkdir -p "$BUILD_ROOT/universal"

# libaff4 itself (installed to $BUILD_ROOT/{arm64,x86_64}/lib/)
lipo -create \
  "$BUILD_ROOT/arm64/lib/libaff4.a" \
  "$BUILD_ROOT/x86_64/lib/libaff4.a" \
  -output "$BUILD_ROOT/universal/libaff4.a"

# Each dep
for LIB in openssl/lib/libcrypto.a openssl/lib/libssl.a \
           raptor/lib/libraptor2.a uriparser/lib/liburiparser.a \
           pcre2/lib/libpcre2-8.a snappy/lib/libsnappy.a; do

  BASENAME=$(basename "$LIB")
  lipo -create \
    "$BUILD_ROOT/deps/arm64/$LIB" \
    "$BUILD_ROOT/deps/x86_64/$LIB" \
    -output "$BUILD_ROOT/universal/$BASENAME"
done
```

Verify all:

```bash
for f in "$BUILD_ROOT/universal/"*.a; do
  echo -n "$f: "
  lipo -info "$f"
done
# Every line should show: x86_64 arm64
```

---

## Part D — Create the Swift Package Structure

### D.1 — Directory layout

```bash
SPM_ROOT="$HOME/LibAFF4SPM"

mkdir -p "$SPM_ROOT/Sources/Claff4/include"
mkdir -p "$SPM_ROOT/Sources/LibAFF4"
mkdir -p "$SPM_ROOT/Tests/LibAFF4Tests"
```

### D.2 — Copy fat binaries

```bash
# libaff4 itself
cp "$BUILD_ROOT/universal/libaff4.a" "$SPM_ROOT/Sources/Claff4/"

# All dep static libs (SPM links them all from one directory)
cp "$BUILD_ROOT/universal/libcrypto.a"    "$SPM_ROOT/Sources/Claff4/"
cp "$BUILD_ROOT/universal/libssl.a"       "$SPM_ROOT/Sources/Claff4/"
cp "$BUILD_ROOT/universal/libraptor2.a"   "$SPM_ROOT/Sources/Claff4/"
cp "$BUILD_ROOT/universal/liburiparser.a" "$SPM_ROOT/Sources/Claff4/"
cp "$BUILD_ROOT/universal/libpcre2-8.a"   "$SPM_ROOT/Sources/Claff4/"
cp "$BUILD_ROOT/universal/libsnappy.a"    "$SPM_ROOT/Sources/Claff4/"
```

### D.3 — Copy headers

```bash
# Main libaff4 header
cp "$BUILD_ROOT/arm64/include/aff4/aff4.h" \
   "$SPM_ROOT/Sources/Claff4/include/"

# Full aff4 headers subfolder
cp -r "$BUILD_ROOT/arm64/include/aff4" \
      "$SPM_ROOT/Sources/Claff4/include/"
```

### D.4 — Create placeholder

```bash
echo "// placeholder" > "$SPM_ROOT/Sources/Claff4/placeholder.c"
```

### D.5 — Final structure

```
LibAFF4SPM/
├── Package.swift
├── Sources/
│   ├── Claff4/
│   │   ├── include/
│   │   │   ├── aff4.h              ← main header
│   │   │   └── aff4/               ← full headers subfolder
│   │   ├── libaff4.a               ← fat binary
│   │   ├── libcrypto.a             ← fat binary
│   │   ├── libssl.a                ← fat binary
│   │   ├── libraptor2.a            ← fat binary
│   │   ├── liburiparser.a          ← fat binary
│   │   ├── libpcre2-8.a            ← fat binary
│   │   ├── libsnappy.a             ← fat binary
│   │   └── placeholder.c
│   └── LibAFF4/
│       └── LibAFF4.swift
└── Tests/
    └── LibAFF4Tests/
        └── LibAFF4Tests.swift
```

---

## Part E — Package.swift

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "LibAFF4",
    platforms: [.macOS(.v13)],
    products: [
        .library(
            name: "LibAFF4",
            targets: ["LibAFF4"]
        ),
    ],
    targets: [
        .target(
            name: "Claff4",
            path: "Sources/Claff4",
            sources: ["placeholder.c"],
            publicHeadersPath: "include",
            linkerSettings: [
                // libaff4 and all its static deps
                .linkedLibrary("aff4"),
                .linkedLibrary("crypto"),
                .linkedLibrary("ssl"),
                .linkedLibrary("raptor2"),
                .linkedLibrary("uriparser"),
                .linkedLibrary("pcre2-8"),
                .linkedLibrary("snappy"),
                // macOS system libs (no bundling needed)
                .linkedLibrary("z"),
                .linkedLibrary("c++"),        // libaff4 is C++, needs stdlib
                .unsafeFlags(["-LSources/Claff4"])
            ]
        ),
        .target(
            name: "LibAFF4",
            dependencies: ["Claff4"],
            path: "Sources/LibAFF4"
        ),
        .testTarget(
            name: "LibAFF4Tests",
            dependencies: ["LibAFF4"],
            path: "Tests/LibAFF4Tests"
        ),
    ]
)
```

> **`-lc++`** is required. libaff4 is a C++ library. The C++ standard library (`libc++`) must be explicitly linked when embedding C++ in a Swift package's C interop target.

---

## Part F — Swift Wrapper

```swift
// Sources/LibAFF4/LibAFF4.swift
import Claff4

public struct AFF4Client {

    /// Opens an AFF4 image at the given path and returns a basic info string.
    /// Replace with actual libaff4 API calls for your use case.
    public static func open(path: String) throws -> String {
        // libaff4 C API entry point — adjust to actual exported symbols
        // See aff4/aff4.h for available functions
        return "AFF4 image at \(path)"
    }
}
```

> **Note:** libaff4's primary API is C++, but it exports a C-compatible interface via `extern "C"` in its headers. Use only those `extern "C"` symbols from Swift. Do not attempt to call C++ class methods directly through the C interop layer.

---

## Part G — Test

```swift
// Tests/LibAFF4Tests/LibAFF4Tests.swift
import XCTest
@testable import LibAFF4

final class LibAFF4Tests: XCTestCase {
    func testOpen() throws {
        let result = try AFF4Client.open(path: "/tmp/test.aff4")
        XCTAssertFalse(result.isEmpty)
        print("Result: \(result)")
    }
}
```

Build and test:

```bash
cd "$SPM_ROOT"
swift build
# Expected: Build complete!

swift test
```

---

## Part H — Verification Checklist

```bash
# 1. Confirm every fat binary has both slices
for f in "$SPM_ROOT/Sources/Claff4/"*.a; do
  echo -n "$(basename $f): "
  lipo -info "$f"
done

# 2. Confirm no unexpected dylib deps leaked in
otool -L "$SPM_ROOT/Sources/Claff4/libaff4.a" 2>/dev/null | grep -v "\.a:"
# Should return nothing — static .a files have no dylib deps

# 3. After swift build, check the final linked product
otool -L .build/debug/LibAFF4
# Acceptable: libSystem.B.dylib, libc++.1.dylib, libz.1.dylib
# NOT acceptable: libssl.dylib, libcrypto.dylib, libraptor2.dylib etc.
# (those must be statically linked, not dylibs)
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `Could not find a package configuration file for Raptor2` | CMake can't find raptor2 | Pass `-DRaptor2_ROOT=` explicitly pointing to your built prefix |
| `Undefined symbols for architecture arm64: std::__1::...` | Missing C++ stdlib link | Add `.linkedLibrary("c++")` to `linkerSettings` |
| `ld: can't open output file ... (No such file or directory)` | CMake output path has special chars | Ensure `$BUILD_ROOT` has no spaces or parens |
| `OpenSSL::Crypto not found` | CMake picked up Apple's system stub | Set `OPENSSL_ROOT_DIR` explicitly to your Homebrew/built prefix |
| `library not found for -laff4` | `.a` not in the `-L` path SPM is using | Double-check `unsafeFlags(["-LSources/Claff4"])` path is correct relative to package root |
| `Undefined symbols: _raptor_new_world` | raptor2 not linked | Add `.linkedLibrary("raptor2")` |
| `pcre2.h not found` during libaff4 CMake | PCRE2 headers not found | Pass `-DPCRE2_INCLUDE_DIR=` explicitly |
| `lipo: ... have the same architecture arm64` | Built both slices with same arch flag | Verify `CMAKE_OSX_ARCHITECTURES` is different per build |
| `no such module 'Claff4'` | publicHeadersPath wrong | Ensure `aff4.h` is directly inside `Sources/Claff4/include/` |
| `'aff4/aff4_base.h' file not found` | Headers subfolder not copied | Run `cp -r .../include/aff4 Sources/Claff4/include/` |

---

## Key Differences From Groups 1 & 2

| | Groups 1 & 2 (libyal) | Group 3 (libaff4) |
|---|---|---|
| **Build system** | autoconf (`./autogen.sh` + `./configure`) | CMake (`cmake -S ... -B ...`) |
| **Language** | C | C++ |
| **Cross-compile flag** | `CC="clang -arch ..."` | `CMAKE_OSX_ARCHITECTURES=arm64` |
| **Dep discovery** | `--with-libX=` configure flags | `-DFOO_ROOT=` / `-DFOO_LIBRARY=` CMake vars |
| **C++ stdlib** | Not needed | Must link `-lc++` in SPM |
| **Dep count** | Deps are also libyal (same build pattern) | Deps use 3 different build systems (OpenSSL, autoconf, CMake) |
| **Fat binary scope** | Just the one `.a` | Must lipo libaff4 **and** all dep `.a` files |

---

## Key Concepts

- **`CMAKE_OSX_ARCHITECTURES`** = CMake's equivalent of `CC="clang -arch ..."` for cross-compilation
- **`-DBUILD_SHARED_LIBS=OFF`** = CMake equivalent of `--enable-static --disable-shared`
- **`-lc++`** = required for any C++ static lib embedded in SPM — the Swift linker doesn't add it automatically
- **`extern "C"`** = the only C++ symbols safely callable from Swift's C interop layer
- **Fat dep `.a` files** = every statically linked dep must also be a fat binary, or the final link fails for one architecture
- **`otool -L`** = your final verification that no unexpected dylibs leaked into the linked product
