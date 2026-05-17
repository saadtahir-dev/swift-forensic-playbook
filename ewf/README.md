# EWF — Expert Witness Format

> Formats: `.E01`, `.Ex01`, `.S01`

---

# Overview

The Expert Witness Format (EWF) is a forensic disk image container format widely used by:

* EnCase
* FTK
* X-Ways
* forensic acquisition/imaging tools

EWF stores:

* raw disk sectors
* compression
* checksums
* metadata
* acquisition information

The primary open-source implementation is:

* [libewf](https://github.com/libyal/libewf?utm_source=chatgpt.com)

This repository documents how to build a **production-grade universal macOS forensic SDK** around:

* `libewf`
* `libsmraw`
* libyal dependencies
* macFUSE integration
* Swift Package Manager integration

The final output includes:

* Universal static libraries (`arm64 + x86_64`)
* Universal CLI binaries
* Universal headers/includes
* macFUSE 5.x compatibility
* Swift Package integration
* Verified mounting support on Apple Silicon and Intel macOS

libewf upstream supports:

* `.E01`
* `.Ex01`
* `.S01`
* `.L01`
* split RAW images

and includes tools such as:

* `ewfmount`
* `ewfinfo`
* `ewfverify`
* `ewfacquire`

as documented by the upstream project. ([GitHub][1])

---

# Upstream Libraries

| Library                                                               | Source                        |
| --------------------------------------------------------------------- | ----------------------------- |
| [libewf](https://github.com/libyal/libewf?utm_source=chatgpt.com)     | Expert Witness Format library |
| [libsmraw](https://github.com/libyal/libsmraw?utm_source=chatgpt.com) | Split RAW image support       |
| [macFUSE](https://macfuse.github.io?utm_source=chatgpt.com)           | macOS FUSE runtime            |

---

# Build Features

| Feature                    | Status |
| -------------------------- | ------ |
| arm64 build                | ✅      |
| x86_64 build               | ✅      |
| Universal static libraries | ✅      |
| Universal CLI tools        | ✅      |
| macFUSE 5.x support        | ✅      |
| EWF mounting verified      | ✅      |
| Swift Package integration  | ✅      |
| Universal SDK layout       | ✅      |

---

# Final Universal SDK Layout

```text
build/universal/prefix/
├── bin/
├── include/
├── lib/
└── share/
```

Includes:

* universal forensic libraries
* universal ewf tools
* universal smraw tools
* universal headers
* SwiftPM-ready integration

---

# Verified Universal Components

## Universal Static Libraries

```text
libewf.a
libsmraw.a
libbfio.a
libcdata.a
libcerror.a
libfdata.a
libhmac.a
libuna.a
...
```

All verified with:

```bash
lipo -info
```

Expected:

```text
Architectures in the fat file: x86_64 arm64
```

---

## Universal CLI Tools

```text
ewfmount
ewfinfo
ewfverify
ewfacquire
smrawmount
smrawverify
```

All verified as:

```text
Mach-O universal binary
```

---

# Guides

| Guide                                                      | Description                                   |
| ---------------------------------------------------------- | --------------------------------------------- |
| [01-build-libewf.md](./01-build-libewf.md)                 | Build universal libewf SDK                    |
| [02-spm-basic.md](./02-spm-basic.md)                       | Basic Swift Package integration               |
| [03-spm-fuse.md](./03-spm-fuse.md)                         | macFUSE-enabled build and SwiftPM integration |
| [04-universal-sdk-layout.md](./04-universal-sdk-layout.md) | Merge and validate universal SDK              |

---

# Important Notes

---

## synclibs.sh Overwrites Patches

Running:

```bash
bash synclibs.sh
```

re-syncs upstream libyal dependencies and may overwrite patched source files.

Always:

1. run `synclibs.sh`
2. THEN apply macFUSE compatibility patches

Never patch before syncing dependencies.

---

## macFUSE 5.x Compatibility

Modern macFUSE versions use Darwin-specific callback APIs:

```c
fuse_darwin_fill_dir_t
struct fuse_darwin_attr
```

Older libyal code expects standard POSIX/FUSE types:

```c
fuse_fill_dir_t
struct stat
```

This causes build failures on modern macOS/macFUSE versions.

Typical errors:

```text
incompatible function pointer types
```

```text
struct stat * vs struct fuse_darwin_attr *
```

macFUSE Darwin extensions are documented by the macFUSE project. ([GitHub][2])

---

# Correct Compatibility Strategy

The correct fix is:

| Layer             | Type                      |
| ----------------- | ------------------------- |
| Internal logic    | `struct stat`             |
| FUSE boundary     | `struct fuse_darwin_attr` |
| Callback typedefs | `fuse_darwin_fill_dir_t`  |
| Translation layer | explicit conversion       |

Do NOT globally replace:

```c
struct stat
```

with:

```c
struct fuse_darwin_attr
```

That breaks internal ABI assumptions and causes failures such as:

```text
no member named 'st_size'
```

---

# Runtime Verification

Verified working mount:

```bash
ewfmount image.E01 /tmp/ewf_mount
```

Expected output:

```text
/tmp/ewf_mount/ewf1
```

Verified result:

```text
-r--r--r--  17G  ewf1
```

which exposes the mounted raw forensic stream.

---

# Swift Package Integration

The SDK supports:

* static linking
* SwiftPM integration
* bundled CLI tools
* notarized macOS applications

Supported via:

* universal `.a` libraries
* universal binaries
* universal headers

---

# macFUSE Runtime Requirement

Users must install macFUSE separately:

```bash
brew install --cask macfuse
```

because macFUSE requires:

* privileged installation
* system extensions
* kernel/runtime integration

Official runtime:

* [macFUSE Official Site](https://macfuse.github.io?utm_source=chatgpt.com)

---

# Supported macOS Architectures

| Architecture           | Status |
| ---------------------- | ------ |
| Apple Silicon (arm64)  | ✅      |
| Intel (x86_64)         | ✅      |
| Universal/Fat binaries | ✅      |

---

# Verified Build Outputs

Verified successfully:

* arm64 static libraries
* x86_64 static libraries
* universal merged libraries
* arm64 ewfmount
* x86_64 ewfmount
* universal ewfmount
* arm64 smrawmount
* x86_64 smrawmount
* universal smrawmount

---

# References

* [libewf GitHub](https://github.com/libyal/libewf?utm_source=chatgpt.com)
* [libsmraw GitHub](https://github.com/libyal/libsmraw?utm_source=chatgpt.com)
* [macFUSE Official Site](https://macfuse.github.io?utm_source=chatgpt.com)
* [Homebrew libewf Formula](https://formulae.brew.sh/formula/libewf?utm_source=chatgpt.com)
* [Kali libewf Tools Reference](https://www.kali.org/tools/libewf/?utm_source=chatgpt.com)

libewf project details, supported formats, and ewftools capabilities are documented upstream. ([GitHub][1])

[1]: https://github.com/libyal/libewf?utm_source=chatgpt.com "Libewf is a library to access the Expert Witness ..."
[2]: https://github.com/libyal/libewf/?utm_source=chatgpt.com "Libewf is a library to access the Expert Witness ..."
