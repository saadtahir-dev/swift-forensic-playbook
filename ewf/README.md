# EWF — Expert Witness Format

> Formats: `.E01`, `.Ex01`, `.S01`

---

## Overview

The Expert Witness Format (EWF) is used by EnCase and other forensic tools. It supports compression, hashing, and metadata alongside the raw image data.

The C library for reading and writing EWF images is **libewf**, maintained by Joachim Metz as part of the libyal project.

---

## Library

| Library | Source |
|---|---|
| libewf | https://github.com/libyal/libewf |

libewf depends on a set of common libyal libraries (`libbfio`, `libcdata`, `libcerror`, etc.) plus OpenSSL and zlib.

---

## Guides

| Guide | Description |
|---|---|
| [01 — Build libewf](./01-build-libewf.md) | Build libewf and all dependencies as universal static libraries |
| [02 — SPM (Basic)](./02-spm-basic.md) | Wrap libewf in a Swift Package without FUSE support |
| [03 — SPM (with FUSE)](./03-spm-fuse.md) | Wrap libewf with macFUSE support for mounting EWF images |

---

## Quick Reference

**Build pattern:** autoconf (`./autogen.sh` + `./configure`)

**Key configure flags:**
```bash
--enable-static --disable-shared
CC="clang -arch arm64"
--host=aarch64-apple-macos
```

**System deps (no build needed):**
- zlib → `/usr/lib/libz.dylib`
- bzip2 → `/usr/lib/libbz2.dylib`
- macFUSE → user installs via `brew install --cask macfuse`

---

## Notes

- All libyal libraries follow the same autoconf build pattern
- Build each dependency first, then libewf
- FUSE support is optional — only needed if you want to mount images as virtual filesystems
