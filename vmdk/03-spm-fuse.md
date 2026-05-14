# 03 — Bundled Tool Locator (VMDKToolLocator)

`vmdkmount` is bundled inside the SPM package as a resource in `CLibVMDKResources`. At runtime, `VMDKToolLocator` resolves the path to the binary from the resource bundle so consumers don't need to install anything separately — the only external dependency is macFUSE itself (a kernel extension the user must install).

---

## Architecture

```
CLibVMDKResources (SPM target)
  └── bin/
        ├── vmdkmount   ← universal binary, links libfuse3.4.dylib
        └── vmdkinfo    ← universal binary

CLibVMDKResourcesBundle.swift
  └── exposes Bundle.module for the CLibVMDKResources target

VMDKToolLocator.swift (in LibVMDK)
  └── import CLibVMDKResources
  └── resolves tools via CLibVMDKResourcesBundle.bundle
  └── caches resolved paths (thread-safe, NSLock)
  └── sets executable permissions on first access
```

---

## Why a Dedicated Resources Target

SPM C targets (`CLibVMDK`) don't generate a reliable `Bundle.module` accessor in all contexts (especially Xcode app bundles). Putting resources in a separate Swift target (`CLibVMDKResources`) solves this — Swift targets always generate `Bundle.module` correctly and the bundle is copied into the app bundle automatically.

This is the same pattern used by `libewf-spm` (`ClibewfResources` + `EWFToolLocator`).

---

## CLibVMDKResourcesBundle.swift

```swift
import Foundation

public enum CLibVMDKResourcesBundle {
    public static let bundle: Bundle = {
        Bundle.module
    }()
}
```

---

## VMDKToolLocator.swift

```swift
import Foundation
import CLibVMDKResources

public enum VMDKToolLocator {

    private static let lock = NSLock()
    private static var cache: [String: String] = [:]

    public static func path(for tool: String) -> String? {
        lock.lock()
        if let cached = cache[tool] {
            lock.unlock()
            return cached
        }
        lock.unlock()

        if let url = CLibVMDKResourcesBundle.bundle.url(
            forResource: tool,
            withExtension: nil,
            subdirectory: "bin"
        ) {
            let path = url.path

            try? FileManager.default.setAttributes(
                [.posixPermissions: 0o755],
                ofItemAtPath: path
            )

            if FileManager.default.isExecutableFile(atPath: path) {
                lock.lock()
                cache[tool] = path
                lock.unlock()

                print("[VMDKToolLocator] Cached \(tool) -> \(path)")
                return path
            }
        }

        print("[VMDKToolLocator] Tool not found in CLibVMDKResources bundle: \(tool)")
        return nil
    }

    public static var vmdkmount: String? { path(for: "vmdkmount") }
    public static var vmdkinfo: String?  { path(for: "vmdkinfo") }
}
```

---

## Runtime Bundle Path

When used from an Xcode app bundle, the resolved path looks like:

```
<App>.app/Contents/Resources/libvmdk-spm_CLibVMDKResources.bundle/Contents/Resources/bin/vmdkmount
```

When used via `swift build` / `swift run`:

```
.build/debug/<product>.bundle/bin/vmdkmount
```

`Bundle.module` handles both cases automatically.

---

## macFUSE Requirement

`vmdkmount` links against `/usr/local/lib/libfuse3.4.dylib` (macFUSE 5.x). This dylib is installed by macFUSE and must be present at runtime.

The user must install macFUSE:
```bash
brew install --cask macfuse
```

After installation, the kernel extension may need to be approved in System Settings → Privacy & Security.

Verify the kext is loaded:
```bash
kextstat | grep -i fuse
# Expected: io.macfuse.filesystems.macfuse.25 ...
```

---

## Usage in your app (for example ImageMounter)

`VMDKMountManager` in a host app such as ImageMounter uses `VMDKToolLocator` to resolve `vmdkmount` before calling `ProcessExecutor`:

```swift
import LibVMDK

let toolPath = VMDKToolLocator.vmdkmount ?? SystemToolLocator.path(for: "vmdkmount")

let mountResult = try await executor.run(
    executable: toolPath,
    arguments: [url.path, mountPoint.path],
    throwOnNonZeroExit: true,
    log: log
)
```

`ProcessExecutor` receives an already-resolved absolute path — it does no further tool lookup.

---

## macFUSE 5.x Compatibility Notes

`vmdkmount` was patched before building to support macFUSE 5.x. The key changes in `vmdktools/mount_fuse.c`:

| Change | Detail |
|---|---|
| `fuse_fill_dir_t` → `fuse_darwin_fill_dir_t` | Callback typedef changed in macFUSE 5.x |
| `struct stat *` → `struct fuse_darwin_attr *` in signatures | macFUSE 5.x callback signatures use Darwin attr type |
| Internal logic kept as `struct stat` | Fields `st_size`, `st_mode` etc. work as before |
| Boundary layer translation added | Explicit copy from `struct stat` to `fuse_darwin_attr` at end of `mount_fuse_set_stat_info` |
| `readdir` local var updated | `struct fuse_darwin_attr *stat_info` + updated `memory_allocate_structure` |

Without this patch, `vmdkmount` exits 1 with: `No sub system to mount VMware Virtual Disk (VMDK) format.`
