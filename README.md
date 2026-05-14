# swift-forensic-playbook

> Battle-tested guides for wrapping forensic disk image libraries as universal Swift Packages (SPM) on macOS.

Building forensic imaging tools on macOS with Swift means wrestling with C/C++ libraries, autoconf build systems, cross-architecture compilation, and SPM interop — none of which have good documentation in one place. This playbook documents the complete real-world process, including every error encountered and how it was resolved.

---

## What This Is

A collection of step-by-step build and packaging guides for the major forensic disk image formats, each producing:

- A universal static library (arm64 + x86_64 fat binary)
- A Swift Package (SPM) that wraps the C interface
- Verified with `swift build` + `swift test`

All guides are written from actual builds on Apple Silicon macOS. Every failure, workaround, and fix is documented.

---

## Formats Covered

| Format | Library | Status |
|---|---|---|
| [Raw / DD](./raw-dd/README.md) | Direct I/O + macFUSE | ✅ |
| [E01 / Ex01 / S01 (EWF)](./ewf/README.md) | libewf | ✅ |
| [AFF4](./aff4/README.md) | aff4-cpp-lite + Velocidex/c-aff4 | ✅ |
| [DMG / SparseBundle](./dmg/README.md) | macOS native (DiskArbitration) | ✅ |
| [VMDK](./vmdk/README.md) | libvmdk | ✅ |
| [VHD / VHDX](./vhd/README.md) | libvhdi | 🔜 |

---

## Who This Is For

- Swift/macOS developers building forensic imaging tools
- Security researchers integrating forensic format support into macOS apps
- Anyone who has spent hours trying to get `libewf` or `libaff4` to build statically on Apple Silicon

---

## Environment

All guides assume:

- Apple Silicon Mac (M-series)
- macOS 12.0+ deployment target
- Xcode 15+
- Homebrew installed

Cross-architecture builds (arm64 + x86_64) are covered throughout, including the pkg-config isolation techniques required on Apple Silicon.

**Path conventions:** Bash snippets that compile or copy artifacts start with `BUILD_ROOT="${HOME}/forensic-libs"` (object files, prefixes, fat libraries) and `SPM_ROOT="${HOME}/forensic-spm"` (Swift package checkouts). Override those variables to match your machine before running commands verbatim.

---

## Structure

```
(your SPM_ROOT)/swift-forensic-playbook/
├── raw-dd/          ← Raw/DD sector dump formats
├── ewf/             ← Expert Witness Format (E01/Ex01/S01)
├── aff4/            ← Advanced Forensics File Format 4
├── dmg/             ← Apple DMG/SparseBundle (native)
├── vmdk/            ← VMware VMDK
└── vhd/             ← Microsoft VHD/VHDX
```

---

## Key Concepts Across All Guides

**Universal fat binary** — a single `.a` file containing both arm64 and x86_64 slices, created with `lipo -create`. Required for SPM packages targeting both architectures.

**pkg-config isolation** — on Apple Silicon, Homebrew installs arm64-only packages. Cross-arch builds require isolating `PKG_CONFIG_PATH` to custom-built x86_64 dep trees. Covered in detail in the AFF4 guides.

**Static linking** — all libraries are built with `--enable-static` (autoconf) or `-DBUILD_SHARED_LIBS=OFF` (CMake) to produce self-contained SPM packages with no runtime dylib dependencies.

**module.modulemap** — C++ libraries expose a C interface (`extern "C"`) that Swift can call. A `module.modulemap` restricts SPM's module system to only the C-compatible headers, hiding C++ internals.

**libyal build pattern** — libyal libraries (libewf, libvmdk, libvhdi, etc.) require `git clone` (not tarballs), `./synclibs.sh` to populate embedded dependencies, and `./autogen.sh` to generate the build system. Direct `autoreconf -fi` does not work.

---

## Contributing

If you've built a forensic format library not covered here, PRs are welcome. Follow the existing guide structure — include every error and fix, not just the happy path.

---

## License

MIT
