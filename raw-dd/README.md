# Raw / DD — Direct Sector Dump Formats

> Formats: `.raw`, `.dd`, `.000`, `.001`, `.00001`

---

## Overview

Raw sector dump formats are the simplest forensic image format — a flat binary copy of a disk with no container structure. There is no library required to read them beyond standard file I/O.

On macOS, mounting raw images as virtual filesystems requires **macFUSE**.

---

## What You Need

| Component | Source | Notes |
|---|---|---|
| macFUSE | https://github.com/osxfuse/osxfuse/releases | User must install — kernel extension, cannot be bundled |
| Standard file I/O | macOS system | No library needed |

---

## macFUSE Installation

macFUSE must be installed by the user. It cannot be bundled inside an app because it requires a kernel extension.

```bash
brew install --cask macfuse
```

Verify installation:

```bash
ls /Library/Filesystems/macfuse.fs
# Expected: Contents
```

---

## Reading Raw Images

Raw images are read with standard `open()` / `read()` / `lseek()` / `close()` system calls. No third-party library is needed.

```c
#include <fcntl.h>
#include <unistd.h>

int fd = open("/path/to/image.dd", O_RDONLY);
// seek to sector offset, read sectors directly
```

---

## Mounting via macFUSE

To mount a raw image as a virtual filesystem, implement `fuse_operations` from the macFUSE headers and use standard FUSE mount calls.

macFUSE headers are located at:
```
/usr/local/include/fuse.h
/usr/local/lib/libfuse.dylib
```

---

## Multi-Part Images

Formats like `.000`, `.001`, `.00001` are simply segmented raw images. Read them sequentially — open each segment in order and treat them as a contiguous byte stream.

---

## Notes

- No SPM package needed — raw I/O is handled natively
- macFUSE is a user prerequisite, not a bundled dependency
- Verify macFUSE is installed at app launch and prompt the user if missing
