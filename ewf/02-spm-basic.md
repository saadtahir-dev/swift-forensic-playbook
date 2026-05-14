# Wrapping libewf in a Swift Package (Basic — No FUSE)

> Wraps the prebuilt universal `libewf.a` into a Swift Package for use in macOS apps.

---

## Prerequisites

- Universal `libewf.a` built per [01-build-libewf.md](./01-build-libewf.md)

---

## Step 1 — Create Package Structure

```bash
mkdir -p SwiftPackage/Sources/Clibewf/include
mkdir -p SwiftPackage/Sources/Libewf
mkdir -p SwiftPackage/Tests/LibewfTests
```

---

## Step 2 — Copy Files

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

# Universal binary
cp "${BUILD_ROOT}/LIB-EWF-SPM/libewf.a" \
   SwiftPackage/Sources/Clibewf/

# Main header
cp "${BUILD_ROOT}/LIB-EWF-SPM/arm64/include/libewf.h" \
   SwiftPackage/Sources/Clibewf/include/

# Supporting headers subfolder
cp -r "${BUILD_ROOT}/LIB-EWF-SPM/arm64/include/libewf" \
      SwiftPackage/Sources/Clibewf/include/

# Placeholder (required by SPM)
echo "// placeholder" > SwiftPackage/Sources/Clibewf/placeholder.c
```

> Headers are architecture-independent — copy from either arm64 or x86_64 build.

---

## Step 3 — Package.swift

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

## Step 4 — Swift Wrapper

```swift
// Sources/Libewf/Libewf.swift
import Clibewf

public struct LibewfClient {
    public static func getVersion() -> String {
        let version = libewf_get_version()
        return String(cString: version!)
    }
}
```

---

## Step 5 — Test

```swift
// Tests/LibewfTests/LibewfTests.swift
import XCTest
@testable import Libewf

final class LibewfTests: XCTestCase {
    func testVersion() {
        let version = LibewfClient.getVersion()
        XCTAssertFalse(version.isEmpty)
        print("libewf version: \(version)")
    }
}
```

---

## Step 6 — Build and Test

```bash
cd SwiftPackage
swift build   # Expected: Build complete!
swift test    # Expected: Test Suite 'All tests' passed
```

---

## Package Structure

```
SwiftPackage/
├── Package.swift
├── Sources/
│   ├── Clibewf/
│   │   ├── include/
│   │   │   ├── libewf.h
│   │   │   └── libewf/
│   │   ├── libewf.a
│   │   └── placeholder.c
│   └── Libewf/
│       └── Libewf.swift
└── Tests/
    └── LibewfTests/
        └── LibewfTests.swift
```

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `library not found for -lewf` | `.a` not found by linker | Check `unsafeFlags` `-L` path |
| `no such module 'Clibewf'` | Header path wrong | Check `publicHeadersPath` |
| `'libewf/types.h' file not found` | Headers subfolder not copied | Run `cp -r .../include/libewf ...` |
| `Undefined symbols` | Missing system libs | Add `-lz -lbz2 -liconv` |
