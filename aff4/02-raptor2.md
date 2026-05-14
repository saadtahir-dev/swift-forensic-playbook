# raptor2 — Build Guide

> **Critical:** Use the **release tarball**, not the git snapshot. The git snapshot has broken flex/bison generated lexer files that cause null pointer dereferences (`EXC_BAD_ACCESS` in `raptor_new_uri_relative_to_base_counted`) when processing real Turtle RDF metadata from AFF4 images. The release tarball ships with correctly pre-generated lexer/parser files.

---

## Why the Git Snapshot Fails

The raptor2 git snapshot (`github.com/dajobe/raptor`) does not include pre-generated lexer/parser C files. When generated manually with modern flex/bison under `-std=gnu11`, the resulting `turtle_lexer.c` has subtle URI handling bugs — specifically a null pointer dereference in `raptor_new_uri_relative_to_base_counted` that only surfaces when parsing real Turtle RDF data from AFF4 images.

The **raptor2 2.0.16 release tarball** includes correctly pre-generated files that have been tested against real-world data.

---

## Prerequisites

```bash
brew install autoconf automake libtool
```

> Unlike the git snapshot, the release tarball does NOT require `gtk-doc`, `bison`, or `flex`.

---

## Step 1 — Download Release Tarball

```bash
cd "$BUILD_ROOT"
curl -L https://download.librdf.org/source/raptor2-2.0.16.tar.gz -o raptor2-2.0.16.tar.gz
tar xzf raptor2-2.0.16.tar.gz
cd raptor2-2.0.16
```

Verify pre-generated files are present:

```bash
ls src/turtle_lexer.c src/turtle_parser.c src/parsedate.c
# All three must exist
```

---

## Step 2 — Build for arm64

```bash
./configure --enable-static --disable-shared \
  CC="clang -arch arm64" \
  CFLAGS="-std=gnu11 -arch arm64 -mmacosx-version-min=12.0" \
  --host=aarch64-apple-macos \
  --prefix="$BUILD_ROOT/deps/arm64/raptor"

make -j$(sysctl -n hw.ncpu) && make install
```

Verify:

```bash
lipo -info "$BUILD_ROOT/deps/arm64/raptor/lib/libraptor2.a"
# Expected: architecture: arm64
```

---

## Step 3 — Build for x86_64

**Important:** Run `make distclean` between builds.

```bash
make distclean

./configure --enable-static --disable-shared \
  CC="clang -arch x86_64" \
  CFLAGS="-std=gnu11 -arch x86_64 -mmacosx-version-min=12.0" \
  --host=x86_64-apple-macos \
  --prefix="$BUILD_ROOT/deps/x86_64/raptor"

make -j$(sysctl -n hw.ncpu) && make install
```

Verify:

```bash
lipo -info "$BUILD_ROOT/deps/x86_64/raptor/lib/libraptor2.a"
# Expected: architecture: x86_64
```

---

## Step 4 — Create Universal Binary

```bash
mkdir -p "$BUILD_ROOT/deps/universal/raptor/lib"

lipo -create \
  "$BUILD_ROOT/deps/arm64/raptor/lib/libraptor2.a" \
  "$BUILD_ROOT/deps/x86_64/raptor/lib/libraptor2.a" \
  -output "$BUILD_ROOT/deps/universal/raptor/lib/libraptor2.a"

lipo -info "$BUILD_ROOT/deps/universal/raptor/lib/libraptor2.a"
# Expected: x86_64 arm64
```

---

## Why `make distclean` Between Builds

The arm64 build produces `.libs/librdfa.a` as a fat file. When x86_64 tries to extract objects from it using `ar`, it fails:

```
ar: librdfa.a is a fat file (use libtool(1) or lipo(1) and ar(1) on it)
ar: Inappropriate file type or format
```

`make distclean` wipes these intermediate files before the x86_64 build.

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `EXC_BAD_ACCESS` in `raptor_new_uri_relative_to_base_counted` | Git snapshot lexer bug with real AFF4 data | Use release tarball 2.0.16 |
| `AFF4_open` returns nil, no crash | raptor URI base set to `"."` instead of absolute URI | Patch `data_store.cc` line 277: replace `"."` with `"file:///"` |
| `ar: librdfa.a is a fat file` | Intermediate files from previous arch build | Run `make distclean` between builds |
| `Undefined symbols: _xmlInitParser` | libxml2 not linked in SPM | Add `.linkedLibrary("xml2")` to Package.swift |
| `lipo: same architecture` | Both slices built with same arch | Verify `CC="clang -arch ..."` differs per build |
