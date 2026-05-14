# Wrapping Velocidex/c-aff4 in a Swift Package (SPM)

> Wraps the prebuilt universal `libaff4.a` (from [05-c-aff4-build.md](./05-c-aff4-build.md)) into a Swift Package.

---

## Prerequisites

- Universal `libaff4.a` and all dep `.a` files built per [05-c-aff4-build.md](./05-c-aff4-build.md)
- All universal dep binaries from [01-dependencies.md](./01-dependencies.md)

---

## Step 1 — Create Package Structure

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"
CAFF4_SPM="${SPM_ROOT}/libcaff4-spm"

mkdir -p "$CAFF4_SPM/Sources/Ccaff4/include/aff4"
mkdir -p "$CAFF4_SPM/Sources/Libcaff4"
mkdir -p "$CAFF4_SPM/Tests/Libcaff4Tests"
```

---

## Step 2 — Copy Fat Binaries

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"
CAFF4_SPM="${SPM_ROOT}/libcaff4-spm"

# libaff4 itself
cp "$CAFF4_ROOT/deps/universal/lib/libaff4.a"            "$CAFF4_SPM/Sources/Ccaff4/"

# Shared deps (built in 01-dependencies.md)
cp "$AFF4_LITE_ROOT/deps/universal/raptor/lib/libraptor2.a"   "$CAFF4_SPM/Sources/Ccaff4/"
cp "$AFF4_LITE_ROOT/deps/universal/lz4/lib/liblz4.a"          "$CAFF4_SPM/Sources/Ccaff4/"
cp "$AFF4_LITE_ROOT/deps/universal/snappy/lib/libsnappy.a"     "$CAFF4_SPM/Sources/Ccaff4/"
cp "$AFF4_LITE_ROOT/deps/universal/uriparser/lib/liburiparser.a" "$CAFF4_SPM/Sources/Ccaff4/"
cp "$AFF4_LITE_ROOT/deps/universal/zlib/lib/libz.a"            "$CAFF4_SPM/Sources/Ccaff4/"
cp "$AFF4_LITE_ROOT/deps/universal/openssl/lib/libcrypto.a"    "$CAFF4_SPM/Sources/Ccaff4/"
cp "$AFF4_LITE_ROOT/deps/universal/openssl/lib/libssl.a"       "$CAFF4_SPM/Sources/Ccaff4/"
```

---

## Step 3 — Copy Headers

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"
CAFF4_SPM="${SPM_ROOT}/libcaff4-spm"

# Full aff4 headers subfolder
cp -r "$CAFF4_ROOT/deps/arm64/caff4/include/aff4/" \
      "$CAFF4_SPM/Sources/Ccaff4/include/aff4/"

# Vendored spdlog headers (required at compile time)
cp -r "$CAFF4_ROOT/spdlog-src/include/spdlog" \
      "$CAFF4_SPM/Sources/Ccaff4/include/spdlog"
```

---

## Step 4 — Create module.modulemap

c-aff4 headers are C++. SPM's module system cannot handle C++ headers directly. A `module.modulemap` restricts the module to only the C-compatible interface:

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"
CAFF4_SPM="${SPM_ROOT}/libcaff4-spm"

cat > "$CAFF4_SPM/Sources/Ccaff4/include/module.modulemap" << 'EOF'
module Ccaff4 {
    header "aff4/libaff4-c.h"
    export *
}
EOF
```

> **Why `module.modulemap`?** Without it, SPM tries to import all headers including C++ ones (`aff4_base.h` includes `<iostream>`), causing `'iostream' file not found` errors. The modulemap exposes only `libaff4-c.h` — the pure `extern "C"` interface.

---

## Step 5 — Create Placeholder

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"
CAFF4_SPM="${SPM_ROOT}/libcaff4-spm"

echo "// placeholder" > "$CAFF4_SPM/Sources/Ccaff4/placeholder.c"
```

---

## Step 6 — Package.swift

```swift
// swift-tools-version: 5.9
import PackageDescription
import Foundation

// Xcode resolves SPM package paths differently from `swift build`.
// Using #filePath (absolute path to this Package.swift at build time)
// ensures the -L flag resolves correctly in both contexts.
// Do NOT replace with a relative path like "-LSources/Ccaff4" — it breaks Xcode builds.
let packageRoot = URL(fileURLWithPath: #filePath)
    .deletingLastPathComponent()
    .path

let package = Package(
    name: "Libcaff4",
    platforms: [.macOS(.v13)],
    products: [
        .library(
            name: "Libcaff4",
            targets: ["Libcaff4"]
        ),
    ],
    targets: [
        .target(
            name: "Ccaff4",
            path: "Sources/Ccaff4",
            sources: ["placeholder.c"],
            publicHeadersPath: "include",
            cxxSettings: [
                .define("SPDLOG_HEADER_ONLY"),
                .headerSearchPath("include"),
                .headerSearchPath("include/spdlog"),
            ],
            linkerSettings: [
                .linkedLibrary("aff4"),
                .linkedLibrary("raptor2"),
                .linkedLibrary("lz4"),
                .linkedLibrary("snappy"),
                .linkedLibrary("uriparser"),
                .linkedLibrary("z"),
                .linkedLibrary("crypto"),
                .linkedLibrary("ssl"),
                .linkedLibrary("c++"),
                .linkedLibrary("pthread"),
                .linkedLibrary("xml2"),
                .linkedLibrary("xslt"),
                .linkedLibrary("iconv"),
                .linkedLibrary("curl"),
                .unsafeFlags(["-L\(packageRoot)/Sources/Ccaff4"])
            ]
        ),
        .target(
            name: "Libcaff4",
            dependencies: ["Ccaff4"],
            path: "Sources/Libcaff4"
        ),
        .testTarget(
            name: "Libcaff4Tests",
            dependencies: ["Libcaff4"],
            path: "Tests/Libcaff4Tests"
        ),
    ]
)
```

> **`#filePath` vs relative `-L` path** — `swift build` resolves `-LSources/Ccaff4` relative to the package root correctly, but Xcode resolves it relative to the build intermediates directory, causing `ld: library 'aff4' not found`. Using `#filePath` gives an absolute path to `Package.swift` at build time, which works in both contexts. Always use `#filePath` when the package will be integrated into an Xcode project.

> **`-lxml2 -lxslt`** — raptor2 was built with libxml2 and libxslt support. Both ship with macOS and must be linked explicitly.

> **`-lc++`** — c-aff4 is C++. The C++ stdlib must be explicitly linked in SPM C interop targets.

---

## Step 7 — Swift Wrapper

```swift
// Sources/Libcaff4/Libcaff4.swift
import Ccaff4

public struct CAFF4Client {

    /// Returns the libaff4 version string.
    public static func version() -> String {
        let ptr = AFF4_version()
        return ptr.map { String(cString: $0) } ?? ""
    }

    /// Sets the log verbosity level.
    public static func setVerbosity(_ level: AFF4_LOG_LEVEL) {
        AFF4_set_verbosity(level)
    }

    /// Opens an AFF4 image and returns a handle.
    /// Caller is responsible for closing the handle with `close()`.
    public static func open(path: String) -> OpaquePointer? {
        var msg: UnsafeMutablePointer<AFF4_Message>? = nil
        let handle = AFF4_open(path, &msg)
        if let m = msg { AFF4_free_messages(m) }
        return handle
    }

    /// Returns the size of the opened AFF4 object in bytes.
    public static func size(handle: OpaquePointer) -> UInt64 {
        var msg: UnsafeMutablePointer<AFF4_Message>? = nil
        let size = AFF4_object_size(handle, &msg)
        if let m = msg { AFF4_free_messages(m) }
        return size
    }

    /// Closes an open AFF4 handle.
    public static func close(handle: OpaquePointer) {
        var msg: UnsafeMutablePointer<AFF4_Message>? = nil
        AFF4_close(handle, &msg)
        if let m = msg { AFF4_free_messages(m) }
    }
}
```

---

## Step 8 — Tests

```swift
// Tests/Libcaff4Tests/Libcaff4Tests.swift
import XCTest
import Ccaff4

final class Libcaff4Tests: XCTestCase {

    func testVersion() {
        let versionPtr = AFF4_version()
        XCTAssertNotNil(versionPtr)
        let version = String(cString: versionPtr!)
        XCTAssertFalse(version.isEmpty)
        print("libaff4 version: \(version)")
    }

    func testSetVerbosity() {
        AFF4_set_verbosity(AFF4_LOG_LEVEL_ERROR)
        print("Verbosity set to ERROR")
    }

    func testOpenImage() throws {
        let path = "/tmp/test.aff4"
        guard FileManager.default.fileExists(atPath: path) else {
            throw XCTSkip("No test image at \(path) — skipping")
        }
        var msg: UnsafeMutablePointer<AFF4_Message>? = nil
        let handle = AFF4_open(path, &msg)
        if let m = msg { AFF4_free_messages(m) }
        XCTAssertNotNil(handle)
        if let h = handle {
            var msg2: UnsafeMutablePointer<AFF4_Message>? = nil
            let size = AFF4_object_size(h, &msg2)
            if let m = msg2 { AFF4_free_messages(m) }
            print("Image size: \(size) bytes")
            XCTAssertGreaterThan(size, 0)
            var msg3: UnsafeMutablePointer<AFF4_Message>? = nil
            AFF4_close(h, &msg3)
            if let m = msg3 { AFF4_free_messages(m) }
        }
    }
}
```

---

## Step 9 — Build and Test

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"
CAFF4_SPM="${SPM_ROOT}/libcaff4-spm"

cd "$CAFF4_SPM"
swift build   # Expected: Build complete!
swift test    # Expected: Test Suite 'All tests' passed
```

---

## Step 10 — Verify No dylib Leaks

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export AFF4_LITE_ROOT="${BUILD_ROOT}/LIB-AFF4-SPM"
export CAFF4_ROOT="${BUILD_ROOT}/LIB-CAFF4-SPM"
CAFF4_SPM="${SPM_ROOT}/libcaff4-spm"

cd "$CAFF4_SPM"
find .build -name "Libcaff4PackageTests" -not -name "*.dSYM" -type f | head -1 | \
  xargs otool -L | grep -v "/usr/lib\|/System\|Testing\|XCTest"
# Expected: blank — no Homebrew or third-party dylib paths
```

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `'iostream' file not found` | SPM importing C++ headers | Add `module.modulemap` exposing only `libaff4-c.h` |
| `Undefined symbols: _xml*` | raptor2 needs libxml2 | Add `.linkedLibrary("xml2")` |
| `Undefined symbols: _xslt*` | raptor2 needs libxslt | Add `.linkedLibrary("xslt")` |
| `Undefined symbols: std::__1::*` | C++ stdlib not linked | Add `.linkedLibrary("c++")` |
| `library not found for -laff4` | `.a` not found | Check `unsafeFlags` `-L` path |

---

## Package Structure

```
(your SPM_ROOT)/libcaff4-spm/
├── Package.swift
├── Sources/
│   ├── Ccaff4/
│   │   ├── include/
│   │   │   ├── module.modulemap     ← exposes only libaff4-c.h
│   │   │   ├── aff4/                ← full aff4 headers
│   │   │   └── spdlog/              ← vendored spdlog 0.17.0 headers
│   │   ├── libaff4.a
│   │   ├── libraptor2.a
│   │   ├── liblz4.a
│   │   ├── libsnappy.a
│   │   ├── liburiparser.a
│   │   ├── libz.a
│   │   ├── libcrypto.a
│   │   ├── libssl.a
│   │   └── placeholder.c
│   └── Libcaff4/
│       └── Libcaff4.swift
└── Tests/
    └── Libcaff4Tests/
        └── Libcaff4Tests.swift
```
