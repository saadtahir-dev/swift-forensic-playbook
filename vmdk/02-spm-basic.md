# 02 — SPM Package

Wraps the universal `libvmdk.a` in a Swift Package with:
- `CLibVMDK` — C target exposing the libvmdk API
- `CLibVMDKFuse` — C target with macFUSE headers linked
- `CLibVMDKResources` — resource bundle containing `vmdkmount` and `vmdkinfo`
- `LibVMDK` — Swift wrapper with `VMDKHandle`, `VMDKToolLocator`, and typed enums

Verified with `swift build` and `swift test` on Apple Silicon macOS.

---

## Final Package Structure

```
(your SPM_ROOT)/libvmdk-spm/
  Package.swift
  Sources/
    CLibVMDK/
      placeholder.c
      include/
        module.modulemap
        vmdk_shim.h
        libvmdk.h + libvmdk/
        libbfio.h + libbfio/
        ... (all libyal headers)
      libvmdk.a + all other .a files
    CLibVMDKFuse/
      placeholder.c
      include/
        module.modulemap
        vmdk_fuse_shim.h
        libvmdk.h + libvmdk/
        ... (same headers as CLibVMDK)
      libvmdk.a + all other .a files
    CLibVMDKResources/
      CLibVMDKResourcesBundle.swift
      bin/
        vmdkmount    ← universal binary (arm64 + x86_64), links libfuse3
        vmdkinfo     ← universal binary (arm64 + x86_64)
    LibVMDK/
      VMDKToolLocator.swift
      VMDKHandle.swift (+ extensions)
      VMDKError.swift
      VMDKDiskType.swift
      VMDKExtentType.swift
      VMDKExtentDescriptor.swift
  Tests/
    LibVMDKTests/
      BasicTests.swift
    LibVMDKSwiftTests/
      VMDKHandleTests.swift
```

---

## Step 1 — Copy Universal Libraries

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

UNIV="${BUILD_ROOT}/LIB-VMDK-SPM/universal"

cp "${UNIV}"/*.a "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDK/"
cp "${UNIV}"/*.a "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDKFuse/"
```

---

## Step 2 — Copy Headers

Copy the full include tree — `libvmdk.h` references sub-headers in `libvmdk/` and sibling lib headers like `libbfio.h`.

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

HDRS="${BUILD_ROOT}/LIB-VMDK-SPM/build/arm64/prefix/include"

cp -r "${HDRS}"/*.h "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDK/include/"
cp -r "${HDRS}"/lib* "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDK/include/"

cp -r "${HDRS}"/*.h "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDKFuse/include/"
cp -r "${HDRS}"/lib* "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDKFuse/include/"
```

---

## Step 3 — Copy Bundled Tools

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

UNIV="${BUILD_ROOT}/LIB-VMDK-SPM/universal"

mkdir -p "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDKResources/bin"

cp "${UNIV}/vmdkmount" "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDKResources/bin/"
cp "${UNIV}/vmdkinfo"  "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDKResources/bin/"
chmod +x "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDKResources/bin/vmdkmount"
chmod +x "${SPM_ROOT}/libvmdk-spm/Sources/CLibVMDKResources/bin/vmdkinfo"
```

---

## Step 4 — placeholder.c files

Both C targets need a compile unit so Xcode doesn't fail with "missing object file":

```c
// Sources/CLibVMDK/placeholder.c  (and CLibVMDKFuse/placeholder.c)
// placeholder compile unit — required by SPM/Xcode for C targets with no sources
```

---

## Step 5 — Shim Headers

```c
// Sources/CLibVMDK/include/vmdk_shim.h
#ifndef VMDK_SHIM_H
#define VMDK_SHIM_H
#include <stdint.h>
#include <stddef.h>
#include "libvmdk.h"
#endif
```

```c
// Sources/CLibVMDKFuse/include/vmdk_fuse_shim.h
#ifndef VMDK_FUSE_SHIM_H
#define VMDK_FUSE_SHIM_H
#include <stdint.h>
#include <stddef.h>
#include "libvmdk.h"
#define _DARWIN_USE_64_BIT_INODE 1
#define FUSE_USE_VERSION 26
#include <fuse/fuse.h>
#endif
```

---

## Step 6 — module.modulemap files

```
// Sources/CLibVMDK/include/module.modulemap
module CLibVMDK {
    header "vmdk_shim.h"
    export *
}
```

```
// Sources/CLibVMDKFuse/include/module.modulemap
module CLibVMDKFuse {
    header "vmdk_fuse_shim.h"
    export *
}
```

---

## Step 7 — CLibVMDKResourcesBundle.swift

```swift
// Sources/CLibVMDKResources/CLibVMDKResourcesBundle.swift
import Foundation

public enum CLibVMDKResourcesBundle {
    public static let bundle: Bundle = {
        Bundle.module
    }()
}
```

---

## Step 8 — Package.swift

```swift
// swift-tools-version:5.9
import PackageDescription
import Foundation

let packageDirectory = URL(fileURLWithPath: #filePath).deletingLastPathComponent().path

let package = Package(
    name: "libvmdk-spm",
    platforms: [.macOS(.v12)],
    products: [
        .library(name: "CLibVMDK",     targets: ["CLibVMDK"]),
        .library(name: "CLibVMDKFuse", targets: ["CLibVMDKFuse"]),
        .library(name: "LibVMDK",      targets: ["LibVMDK"]),
    ],
    targets: [
        .target(
            name: "CLibVMDK",
            path: "Sources/CLibVMDK",
            sources: ["placeholder.c"],
            publicHeadersPath: "include",
            linkerSettings: [
                .unsafeFlags([
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libvmdk.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libbfio.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libcdata.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libcerror.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libclocale.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libcnotify.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libcpath.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libcsplit.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libcthreads.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libfcache.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDK/libfdata.a",
                ]),
                .linkedLibrary("z"),
                .linkedLibrary("pthread"),
            ]
        ),
        .target(
            name: "CLibVMDKFuse",
            path: "Sources/CLibVMDKFuse",
            sources: ["placeholder.c"],
            publicHeadersPath: "include",
            linkerSettings: [
                .unsafeFlags([
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libvmdk.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libbfio.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libcdata.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libcerror.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libclocale.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libcnotify.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libcpath.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libcsplit.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libcthreads.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libfcache.a",
                    "-Xlinker", "\(packageDirectory)/Sources/CLibVMDKFuse/libfdata.a",
                    "-L/usr/local/lib",
                    "-lfuse",
                ]),
                .linkedLibrary("z"),
                .linkedLibrary("pthread"),
            ]
        ),
        .target(
            name: "CLibVMDKResources",
            dependencies: ["CLibVMDK"],
            path: "Sources/CLibVMDKResources",
            resources: [
                .copy("bin")
            ]
        ),
        .target(
            name: "LibVMDK",
            dependencies: ["CLibVMDK", "CLibVMDKResources"],
            path: "Sources/LibVMDK"
        ),
        .testTarget(
            name: "LibVMDKTests",
            dependencies: ["CLibVMDK"],
            path: "Tests/LibVMDKTests"
        ),
        .testTarget(
            name: "LibVMDKSwiftTests",
            dependencies: ["LibVMDK"],
            path: "Tests/LibVMDKSwiftTests"
        ),
    ]
)
```

> **Why `-Xlinker` per-file instead of `-L` + `-l`?** The per-file style guarantees the exact `.a` from the package source directory is linked, regardless of what other libraries are on the system. This avoids accidental dynamic linking against a system or Homebrew version.

---

## Step 9 — Build and Test

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

cd "${SPM_ROOT}/libvmdk-spm"
rm -rf .build
swift build
swift test
```

Expected:
```
Build complete!
libvmdk version: 20251220
Test Suite 'All tests' passed
Executed 7 tests, with 0 failures
```
