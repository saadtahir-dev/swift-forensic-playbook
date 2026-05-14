# DMG / SparseBundle / SparseImage — Apple Disk Images

> Formats: `.dmg`, `.sparsebundle`, `.sparseimage`

---

## Overview

Apple disk image formats are proprietary macOS formats handled natively by the operating system. No third-party library is needed or available — `hdiutil` and `DiskArbitration.framework` do everything.

---

## Approach

All three formats are mounted the same way via `hdiutil attach`:

```bash
hdiutil attach image.dmg -noverify -nobrowse -plist
hdiutil attach image.sparseimage -noverify -nobrowse -plist
hdiutil attach image.sparsebundle -noverify -nobrowse -plist
```

The `-plist` flag returns structured XML with device identifiers and mount points — parse this to get the mounted volume path.

Unmount with:
```bash
hdiutil detach /dev/diskN
```

---

## ImageMounter Integration

`DMGImageMounter` in `ImageMounter` handles all three formats:

- Detects type by file extension (`.dmg`, `.sparseimage`, `.sparsebundle`)
- Calls `hdiutil attach` with `-plist` output
- Parses the plist via `HDIUtilPlistParser` to extract device identifiers and mount points
- Unmounts via `hdiutil detach` on all device identifiers

No FUSE, no library build, no bundled tools required.

---

## Notes

- `.sparsebundle` is a directory bundle (not a flat file) — `hdiutil` handles it transparently
- APFS containers create a synthesized disk (e.g. `disk9`) separate from the physical disk (`disk6`) — the mount point is on the synthesized disk's volume slice
- `hdiutil info -plist` can be used to check already-attached images before mounting
- macFUSE is not involved — this is pure macOS native API