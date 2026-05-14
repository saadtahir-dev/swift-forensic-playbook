# VMDK — VMware Virtual Machine Disk

> Formats: `.vmdk`

---

## Overview

VMware's virtual disk format, used by VMware Fusion, Workstation, and ESXi. Read by **libvmdk**, part of the libyal project.

---

## Library

| Library | Source |
|---|---|
| libvmdk | https://github.com/libyal/libvmdk |

libvmdk shares the same libyal common dependencies as libewf (`libbfio`, `libcdata`, `libcerror`, etc.) plus zlib.

---

## Guides

| Guide | Description |
|---|---|
| [01 — Build libvmdk](./01-build-libvmdk.md) | Build libvmdk and all dependencies as universal static libraries, including vmdkmount with macFUSE support |
| [02 — SPM Package](./02-spm-basic.md) | Wrap libvmdk in a Swift Package with bundled tools and Swift wrapper |
| [03 — Bundled Tool Locator](./03-spm-fuse.md) | How vmdkmount is bundled and resolved at runtime via VMDKToolLocator |

---

## Quick Reference

**Build pattern:** libyal autoconf (`./synclibs.sh` + `./autogen.sh` + `./configure`)

**Key configure flags (static libs):**
```bash
--enable-static
--with-pic
--with-libcerror=<prefix>
CC="clang -arch arm64"
--host=aarch64-apple-darwin
```

**Key configure flags (vmdkmount with FUSE):**
```bash
--with-libfuse
CFLAGS="-I/usr/local/include"
LDFLAGS="-L/usr/local/lib"
PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:..."
```

**System deps (no build needed):**
- zlib → `/usr/lib/libz.dylib`
- pthread → system
- macFUSE → user installs via `brew install --cask macfuse`

---

## Notes

- libyal repos must be cloned with full `git clone` — GitHub archive tarballs are missing required template files
- Each lib needs `./synclibs.sh` run before `./autogen.sh` — this clones embedded dependencies
- Use `./autogen.sh` not `autoreconf -fi` directly
- Only pass `--with-libcerror` to configure — let all other deps use embedded local sources
- All 11 libs must be built in dependency order before libvmdk itself
- `vmdkmount` requires macFUSE 5.x compatibility patch before building — see `01-build-libvmdk.md`
- macFUSE 5.x uses `fuse_darwin_attr` instead of `struct stat` — a boundary layer translation is required in `mount_fuse.c`