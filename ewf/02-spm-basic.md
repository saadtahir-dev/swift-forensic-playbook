# Wrapping libewf in a Swift Package (Basic — No FUSE Runtime)

> Wraps the universal libewf forensic SDK into a Swift Package for use in:
>
> * macOS forensic applications
> * SwiftUI apps
> * command line tools
> * DFIR tooling
> * internal forensic frameworks

This version links directly against the universal static libraries produced in:

```text
${BUILD_ROOT}/LIB-EWF-SPM/universal/prefix
```

---

# Prerequisites

Complete:

* `01-build-libewf.md`

Expected final SDK layout:

```text
${BUILD_ROOT}/LIB-EWF-SPM/universal/prefix/
├── bin/
├── include/
├── lib/
└── share/
```

---

# Root Paths

```bash
# Root folders

BUILD_ROOT="${HOME}/forensic-libs"   # build artifacts/output
SPM_ROOT="${HOME}/forensic-spm"      # Swift Package projects
```

---

# Step 1 — Create Swift Package

```bash
mkdir -p "${SPM_ROOT}"

cd "${SPM_ROOT}"

swift package init --type library --name Libewf
```

---

# Step 2 — Create Package Structure

```bash
cd "${SPM_ROOT}/Libewf"

mkdir -p Sources/Clibewf/include
mkdir -p Sources/Libewf
mkdir -p Tests/LibewfTests
```

---

# Step 3 — Copy Universal SDK Files

```bash
BUILD_ROOT="${HOME}/forensic-libs"
SPM_ROOT="${HOME}/forensic-spm"

UNIVERSAL_ROOT="${BUILD_ROOT}/LIB-EWF-SPM/universal/prefix"
```

Copy universal static libraries:

```bash
cp "${UNIVERSAL_ROOT}/lib/"*.a \
   Sources/Clibewf/
```

Copy public headers:

```bash
cp -R "${UNIVERSAL_ROOT}/include/"* \
      Sources/Clibewf/include/
```

Create placeholder source file required by SwiftPM:

```bash
echo "// placeholder" > Sources/Clibewf/placeholder.c
```

---

# Step 4 — Verify Copied Files

Verify libraries:

```bash
find Sources/Clibewf -name "*.a"
```

Expected output includes:

```text
libewf.a
libbfio.a
libcerror.a
libsmraw.a
libuna.a
```

Verify headers:

```bash
find Sources/Clibewf/include -maxdepth 2 | sort
```

Expected:

```text
include/libewf.h
include/libewf/
include/libbfio.h
include/libbfio/
```

---

# Step 5 — Package.swift

Replace the generated `Package.swift` with:

```swift
// swift-tools-version: 5.9

import PackageDescription

let package = Package(
    name: "Libewf",
    platforms: [
        .macOS(.v13)
    ],
    products: [
        .library(
            name: "Libewf",
            targets: ["Libewf"]
        )
    ],
    targets: [

        // C wrapper target

        .target(
            name: "Clibewf",
            path: "Sources/Clibewf",
            sources: [
                "placeholder.c"
            ],
            publicHeadersPath: "include",
            cSettings: [
                .headerSearchPath("include")
            ],
            linkerSettings: [

                // Static forensic libraries

                .unsafeFlags([
                    "-L\(FileManager.default.currentDirectoryPath)/Sources/Clibewf"
                ]),

                // Core forensic libs

                .linkedLibrary("ewf"),
                .linkedLibrary("bfio"),
                .linkedLibrary("cdata"),
                .linkedLibrary("cdatetime"),
                .linkedLibrary("cerror"),
                .linkedLibrary("cfile"),
                .linkedLibrary("clocale"),
                .linkedLibrary("cnotify"),
                .linkedLibrary("cpath"),
                .linkedLibrary("csplit"),
                .linkedLibrary("cthreads"),
                .linkedLibrary("fcache"),
                .linkedLibrary("fdata"),
                .linkedLibrary("fdatetime"),
                .linkedLibrary("fguid"),
                .linkedLibrary("fvalue"),
                .linkedLibrary("hmac"),
                .linkedLibrary("odraw"),
                .linkedLibrary("smdev"),
                .linkedLibrary("smraw"),
                .linkedLibrary("una"),
                .linkedLibrary("caes"),

                // System libraries

                .linkedLibrary("z"),
                .linkedLibrary("bz2"),
                .linkedLibrary("iconv")
            ]
        ),

        // Swift wrapper target

        .target(
            name: "Libewf",
            dependencies: ["Clibewf"],
            path: "Sources/Libewf"
        ),

        // Tests

        .testTarget(
            name: "LibewfTests",
            dependencies: ["Libewf"],
            path: "Tests/LibewfTests"
        )
    ]
)
```

---

# Step 6 — Create Swift Wrapper

Create:

```text
Sources/Libewf/Libewf.swift
```

Contents:

```swift
import Clibewf

public struct LibewfClient {

    public init() {}

    public func version() -> String {

        guard let version = libewf_get_version() else {
            return "Unknown"
        }

        return String(cString: version)
    }
}
```

---

# Step 7 — Create Test

Create:

```text
Tests/LibewfTests/LibewfTests.swift
```

Contents:

```swift
import XCTest
@testable import Libewf

final class LibewfTests: XCTestCase {

    func testVersion() {

        let client = LibewfClient()

        let version = client.version()

        XCTAssertFalse(version.isEmpty)

        print("libewf version: \(version)")
    }
}
```

---

# Step 8 — Build

```bash
cd "${SPM_ROOT}/Libewf"

swift build
```

Expected:

```text
Build complete!
```

---

# Step 9 — Run Tests

```bash
swift test
```

Expected:

```text
Test Suite 'All tests' passed
```

---

# Step 10 — Verify Universal Linking

Verify linked architectures:

```bash
find .build -name "*.dylib" -o -name "*.a"
```

Verify package resolves universal libraries correctly.

---

# Final Package Layout

```text
Libewf/
├── Package.swift
├── Sources/
│   ├── Clibewf/
│   │   ├── include/
│   │   ├── *.a
│   │   └── placeholder.c
│   │
│   └── Libewf/
│       └── Libewf.swift
│
└── Tests/
    └── LibewfTests/
        └── LibewfTests.swift
```

---

# Notes

This package:

* statically links all forensic dependencies
* does not require Homebrew at runtime
* does not require system-installed libewf
* supports both Apple Silicon and Intel Macs
* is suitable for notarized macOS applications

FUSE runtime mounting support is covered separately in:

```text
03-spm-fuse.md
```

---

# Common Errors

| Error                                       | Cause                        | Fix                                          |
| ------------------------------------------- | ---------------------------- | -------------------------------------------- |
| `library not found for -lewf`               | Static libs not copied       | Verify `.a` files exist in `Sources/Clibewf` |
| `no such module 'Clibewf'`                  | Wrong target structure       | Verify `publicHeadersPath`                   |
| `'libewf/types.h' file not found`           | Include folders not copied   | Copy full include directory                  |
| `Undefined symbols`                         | Missing dependency libraries | Link all libyal `.a` libraries               |
| `Undefined symbols for iconv`               | Missing system lib           | Add `-liconv`                                |
| `Undefined symbols for compress/uncompress` | Missing zlib                 | Add `-lz`                                    |
| `Undefined symbols for BZ2_*`               | Missing bzip2                | Add `-lbz2`                                  |
| SwiftPM ignores `.a` files                  | Missing source file          | Ensure `placeholder.c` exists                |
| Intel Macs fail to load                     | Non-universal libs copied    | Verify with `lipo -info`                     |

---

# Verification Commands

Verify universal libraries:

```bash
for LIB in Sources/Clibewf/*.a; do
  lipo -info "$LIB"
done
```

Expected:

```text
Architectures in the fat file: x86_64 arm64
```

---

# References

* [libyal/libewf GitHub](https://github.com/libyal/libewf?utm_source=chatgpt.com)
* [Swift Package Manager Documentation](https://www.swift.org/documentation/package-manager/?utm_source=chatgpt.com)
* [Apple Universal Binary Guide](https://developer.apple.com/documentation/apple-silicon/building-a-universal-macos-binary?utm_source=chatgpt.com)

Based on upstream libewf architecture and SwiftPM native C target integration practices. ([GitHub][1])

[1]: https://github.com/libyal/libewf/?utm_source=chatgpt.com "Libewf is a library to access the Expert Witness ..."
