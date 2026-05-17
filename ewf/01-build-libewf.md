# Building libewf as a Universal macOS Forensic SDK

> Produces a production-grade universal forensic SDK:
>
> * arm64 + x86_64
> * universal static libraries
> * universal forensic CLI tools
> * macFUSE-enabled mounting support
> * Swift Package Manager ready layout

---

# Prerequisites

Install required tooling:

```bash
brew install autoconf automake libtool gettext openssl@3 pkgconf
brew install --cask macfuse
```

Verify macFUSE installation:

```bash
ls /Library/Filesystems/macfuse.fs
```

---

# Root Paths

```bash
# Root folders

BUILD_ROOT="${HOME}/forensic-libs"   # build artifacts/output
SPM_ROOT="${HOME}/forensic-spm"      # Swift Package projects

# Source checkout

SRC_ROOT="${BUILD_ROOT}/src"

# Final universal SDK

UNIVERSAL_ROOT="${BUILD_ROOT}/LIB-EWF-SPM/universal"
```

---

# Step 1 — Clone Source

```bash
mkdir -p "${SRC_ROOT}"

cd "${SRC_ROOT}"

git clone https://github.com/libyal/libewf.git libewf-with-fuse

cd libewf-with-fuse
```

---

# Step 2 — Synchronize Dependencies

IMPORTANT:

Run `synclibs.sh` BEFORE applying any local patches.

Re-running it later may overwrite patched source files.

```bash
bash synclibs.sh
```

This synchronizes required libyal dependencies including:

* libbfio
* libcerror
* libsmraw
* libuna
* libfdata
* libhmac
* libcaes
* etc.

---

# Step 3 — Generate Build Files

```bash
bash autogen.sh
```

Expected output includes:

```text
glibtoolize: copying file ...
```

Warnings about obsolete autoconf macros are harmless.

IMPORTANT:

If you later run:

```bash
make distclean
```

you must re-run:

```bash
bash autogen.sh
```

before configuring again.

---

# Step 4 — Apply macFUSE 5.x Compatibility Patches

macFUSE 5.x introduces Darwin-specific callback typedefs:

* `fuse_darwin_fill_dir_t`
* `struct fuse_darwin_attr`

The correct fix is:

* patch ONLY FUSE callback signatures
* keep internal filesystem/stat logic using POSIX `struct stat`

DO NOT globally replace:

* `struct stat`
* `st_size`
* `st_mode`
* internal stat handling

That approach causes ABI incompatibilities and compile failures.

---

## Patch libewf

```bash
cd "${SRC_ROOT}/libewf-with-fuse/src/libewf"
```

Patch callback typedefs:

```bash
perl -0pi -e 's/fuse_fill_dir_t filler/fuse_darwin_fill_dir_t filler/g' ewftools/mount_fuse.h
```

Patch getattr callback signature:

```bash
perl -0pi -e 's/int mount_fuse_getattr\(\n     const char \*path,\n     struct stat \*stat_info,\n     struct fuse_file_info \*file_info/int mount_fuse_getattr\(\n     const char *path,\n     struct fuse_darwin_attr *stat_info,\n     struct fuse_file_info *file_info/g' ewftools/mount_fuse.h
```

---

## Patch libsmraw

```bash
cd "${SRC_ROOT}/libewf-with-fuse/src/libsmraw"
```

Patch callback typedefs:

```bash
perl -0pi -e 's/fuse_fill_dir_t filler/fuse_darwin_fill_dir_t filler/g' smrawtools/mount_fuse.h
```

Patch getattr callback signature:

```bash
perl -0pi -e 's/int mount_fuse_getattr\(\n     const char \*path,\n     struct stat \*stat_info,\n     struct fuse_file_info \*file_info/int mount_fuse_getattr\(\n     const char *path,\n     struct fuse_darwin_attr *stat_info,\n     struct fuse_file_info *file_info/g' smrawtools/mount_fuse.h
```

Patch FUSE operation casts:

```bash
perl -0pi -e 's/smrawmount_fuse_operations.readdir    = &mount_fuse_readdir;/smrawmount_fuse_operations.readdir = (int (*)(const char *, void *, fuse_darwin_fill_dir_t, off_t, struct fuse_file_info *, enum fuse_readdir_flags)) \&mount_fuse_readdir;/g' smrawtools/smrawmount.c

perl -0pi -e 's/smrawmount_fuse_operations.getattr    = &mount_fuse_getattr;/smrawmount_fuse_operations.getattr = (int (*)(const char *, struct fuse_darwin_attr *, struct fuse_file_info *)) \&mount_fuse_getattr;/g' smrawtools/smrawmount.c
```

---

# Step 5 — Build arm64

```bash
cd "${SRC_ROOT}/libewf-with-fuse"

PREFIX="${BUILD_ROOT}/LIB-EWF-SPM/arm64/prefix"

SDK_PATH="$(xcrun --sdk macosx --show-sdk-path)"

./configure \
  --enable-static \
  --disable-shared \
  --with-libfuse \
  CC="clang -arch arm64" \
  CFLAGS="-arch arm64 -mmacosx-version-min=12.0 -isysroot ${SDK_PATH}" \
  CPPFLAGS="-I/opt/homebrew/include -I/opt/homebrew/include/fuse" \
  LDFLAGS="-L/opt/homebrew/lib" \
  FUSE_CFLAGS="-I/opt/homebrew/include/fuse -D_FILE_OFFSET_BITS=64" \
  FUSE_LIBS="-L/opt/homebrew/lib -lfuse" \
  --host=aarch64-apple-darwin \
  --prefix="${PREFIX}"

make -j$(sysctl -n hw.ncpu)

make install
```

Verify FUSE support:

```bash
grep HAVE_LIBFUSE common/config.h
```

Expected:

```text
#define HAVE_LIBFUSE3 1
```

---

# Step 6 — Build x86_64

```bash
make distclean

bash autogen.sh
```

```bash
PREFIX="${BUILD_ROOT}/LIB-EWF-SPM/x86_64/prefix"

SDK_PATH="$(xcrun --sdk macosx --show-sdk-path)"

./configure \
  --enable-static \
  --disable-shared \
  --with-libfuse \
  CC="clang -arch x86_64" \
  CFLAGS="-arch x86_64 -mmacosx-version-min=12.0 -isysroot ${SDK_PATH}" \
  CPPFLAGS="-I/usr/local/include -I/usr/local/include/fuse" \
  LDFLAGS="-L/usr/local/lib" \
  FUSE_CFLAGS="-I/usr/local/include/fuse -D_FILE_OFFSET_BITS=64" \
  FUSE_LIBS="-L/usr/local/lib -lfuse" \
  --host=x86_64-apple-darwin \
  --prefix="${PREFIX}"

make -j$(sysctl -n hw.ncpu)

make install
```

---

# Step 7 — Verify Architecture Builds

Verify arm64:

```bash
file "${BUILD_ROOT}/LIB-EWF-SPM/arm64/prefix/bin/ewfmount"
```

Expected:

```text
Mach-O 64-bit executable arm64
```

Verify x86_64:

```bash
file "${BUILD_ROOT}/LIB-EWF-SPM/x86_64/prefix/bin/ewfmount"
```

Expected:

```text
Mach-O 64-bit executable x86_64
```

Verify FUSE linkage:

```bash
otool -L "${BUILD_ROOT}/LIB-EWF-SPM/x86_64/prefix/bin/ewfmount"
```

Expected:

```text
libfuse3
```

---

# Step 8 — Create Universal SDK Layout

```bash
mkdir -p "${UNIVERSAL_ROOT}/prefix"/{bin,include,lib,share}
```

Copy shared assets:

```bash
cp -R \
  "${BUILD_ROOT}/LIB-EWF-SPM/arm64/prefix/include/"* \
  "${UNIVERSAL_ROOT}/prefix/include/"

cp -R \
  "${BUILD_ROOT}/LIB-EWF-SPM/arm64/prefix/share/"* \
  "${UNIVERSAL_ROOT}/prefix/share/"
```

---

# Step 9 — Merge Universal Static Libraries

```bash
for LIB in "${BUILD_ROOT}/LIB-EWF-SPM/arm64/prefix/lib/"*.a; do

  NAME="$(basename "$LIB")"

  lipo -create \
    "${BUILD_ROOT}/LIB-EWF-SPM/arm64/prefix/lib/${NAME}" \
    "${BUILD_ROOT}/LIB-EWF-SPM/x86_64/prefix/lib/${NAME}" \
    -output "${UNIVERSAL_ROOT}/prefix/lib/${NAME}"

done
```

---

# Step 10 — Merge Universal CLI Tools

```bash
for BIN in "${BUILD_ROOT}/LIB-EWF-SPM/arm64/prefix/bin/"*; do

  NAME="$(basename "$BIN")"

  lipo -create \
    "${BUILD_ROOT}/LIB-EWF-SPM/arm64/prefix/bin/${NAME}" \
    "${BUILD_ROOT}/LIB-EWF-SPM/x86_64/prefix/bin/${NAME}" \
    -output "${UNIVERSAL_ROOT}/prefix/bin/${NAME}"

  chmod +x "${UNIVERSAL_ROOT}/prefix/bin/${NAME}"

done
```

---

# Step 11 — Verify Universal SDK

Verify universal libraries:

```bash
for LIB in "${UNIVERSAL_ROOT}/prefix/lib/"*.a; do
  lipo -info "$LIB"
done
```

Expected:

```text
Architectures in the fat file: x86_64 arm64
```

Verify binaries:

```bash
file "${UNIVERSAL_ROOT}/prefix/bin/ewfmount"

file "${UNIVERSAL_ROOT}/prefix/bin/smrawmount"
```

Expected:

```text
Mach-O universal binary with 2 architectures
```

---

# Step 12 — Runtime Verification

Mount an EWF image:

```bash
mkdir -p /tmp/ewf_mount

"${UNIVERSAL_ROOT}/prefix/bin/ewfmount" \
  /path/to/image.E01 \
  /tmp/ewf_mount
```

Verify mounted stream:

```bash
ls -lah /tmp/ewf_mount
```

Expected:

```text
ewf1
```

which exposes the mounted raw forensic stream.

---

# Final SDK Layout

```text
${BUILD_ROOT}/LIB-EWF-SPM/universal/prefix/
├── bin/
├── include/
├── lib/
└── share/
```

---

# Common Errors

| Error                                     | Cause                                      | Fix                                              |
| ----------------------------------------- | ------------------------------------------ | ------------------------------------------------ |
| `No sub system to mount EWF format`       | Built without FUSE                         | Configure with `--with-libfuse`                  |
| `configure: error: missing FUSE support`  | Wrong include path                         | Verify `FUSE_CFLAGS`                             |
| `Makefile.in missing`                     | autotools files removed                    | Run `bash autogen.sh`                            |
| `Undefined symbols` for `-lz` or `-lbz2`  | Missing system libs                        | Add `-lz -lbz2`                                  |
| `conflicting types for mount_fuse_*`      | Incomplete Darwin callback patching        | Patch callback signatures only                   |
| `fa_size/fa_mode` errors                  | Incorrect full Darwin struct migration     | Keep internal POSIX `struct stat` logic          |
| `libuna.a missing` during universal merge | Dependency built for one architecture only | Build libuna separately for missing architecture |

---

# References

* [libyal/libewf GitHub](https://github.com/libyal/libewf?utm_source=chatgpt.com)
* [Apple Universal Binary Documentation](https://developer.apple.com/documentation/apple-silicon/building-a-universal-macos-binary?utm_source=chatgpt.com)
* [Homebrew libewf Formula](https://formulae.brew.sh/formula/libewf?utm_source=chatgpt.com)

([GitHub][1])

[1]: https://github.com/libyal/libewf/?utm_source=chatgpt.com "Libewf is a library to access the Expert Witness ..."
