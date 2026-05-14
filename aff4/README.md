# AFF4 — Advanced Forensics File Format 4

> The open standard for storing digital evidence. Used by tools like Velociraptor, FTK, and others.

---

## Overview

AFF4 is a container format built on ZIP with RDF metadata (Turtle format). It supports compression, hashing, sparse streams, and multiple evidence streams in a single container.

The original C++ reference implementation (`aff4/aff4`) is no longer available. As of 2025 there are two viable C/C++ libraries:

| Library | Repo | Type | Use case |
|---|---|---|---|
| **aff4-cpp-lite** | `aff4/aff4-cpp-lite` | Lightweight C/C++ reader | Read-only mounting and enumeration |
| **c-aff4** | `Velocidex/c-aff4` | Full C++ read+write | Full forensic imaging workflow |

Both are covered in this section.

---

## Guides

| Guide | Description |
|---|---|
| [01 — Dependencies](./01-dependencies.md) | Build all shared deps: OpenSSL, zlib, PCRE2, uriparser, snappy, lz4 |
| [02 — raptor2](./02-raptor2.md) | Build raptor2 (RDF parsing) — real-world build with all fixes documented |
| [03 — aff4-cpp-lite Build](./03-aff4-cpp-lite-build.md) | Build aff4-cpp-lite as universal static library |
| [04 — aff4-cpp-lite SPM](./04-aff4-cpp-lite-spm.md) | Wrap aff4-cpp-lite in a Swift Package |
| [05 — c-aff4 Build](./05-c-aff4-build.md) | Build Velocidex/c-aff4 as universal static library |
| [06 — c-aff4 SPM](./06-c-aff4-spm.md) | Wrap c-aff4 in a Swift Package |

---

## Which Library Should I Use?

**Use `aff4-cpp-lite` if you need:**
- Read-only access to AFF4 images
- Lighter dependency tree
- Simpler build process

**Use `Velocidex/c-aff4` if you need:**
- Full read+write AFF4 support
- Image creation and manipulation
- The `aff4imager` tool functionality

Both guides produce a universal fat binary and a working Swift Package.

---

## Shared Dependencies

Both libraries share the same core dependencies. Build them once from [01-dependencies.md](./01-dependencies.md) and reuse for both:

- OpenSSL (libcrypto + libssl)
- zlib
- PCRE2
- uriparser
- snappy
- lz4
- raptor2

---

## Key Challenges

**Apple Silicon cross-arch builds** require pkg-config isolation. Homebrew on Apple Silicon is arm64-only — without isolating `PKG_CONFIG_PATH`, x86_64 builds accidentally resolve arm64 libraries. Covered in detail in each build guide.

**raptor2** has significant build system issues with modern macOS tooling. The full real-world fix sequence is documented in [02-raptor2.md](./02-raptor2.md).

**c-aff4 spdlog/fmt incompatibility** — the library was written against an older spdlog that bundled fmt internally. Modern Homebrew spdlog/fmt 12.x is ABI-incompatible. Covered in [05-c-aff4-build.md](./05-c-aff4-build.md).
