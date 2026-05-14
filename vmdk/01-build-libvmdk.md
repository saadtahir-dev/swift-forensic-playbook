# 01 — Build libvmdk as Universal Static Library

## Prerequisites

- Xcode Command Line Tools
- Homebrew: `autoconf`, `automake`, `libtool`, `pkg-config`
- macFUSE 5.x installed (required for `vmdkmount`)

```bash
brew install autoconf automake libtool pkg-config
brew install --cask macfuse
```

---

## Directory Layout

```
(your BUILD_ROOT)/LIB-VMDK-SPM/
  src/                ← git clones of all libyal libs
  build/
    arm64/prefix/     ← arm64 install prefix
    x86_64/prefix/    ← x86_64 install prefix
  universal/          ← lipo'd universal .a files and tools
```

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

mkdir -p "${BUILD_ROOT}/LIB-VMDK-SPM"/{src,build/arm64/prefix,build/x86_64/prefix,universal}
```

---

## Build Order

libyal libs must be built in dependency order:

```
libcerror → libcthreads → libcdata → libclocale → libcnotify
→ libcpath → libcsplit → libbfio → libfcache → libfdata → libvmdk
```

---

## Step 1 — Clone All Repos

Do NOT use GitHub archive tarballs. Full `git clone` is required — tarballs are missing template files needed by autotools. `--depth=1` also causes missing files; use a full clone.

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

cat > "${BUILD_ROOT}/LIB-VMDK-SPM/clone.sh" << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

cd "${BUILD_ROOT}/LIB-VMDK-SPM/src"

for LIB in libcerror libcthreads libcdata libclocale libcnotify libcpath libcsplit libbfio libfcache libfdata libvmdk; do
  echo "=== Cloning ${LIB} ==="
  rm -rf "${LIB}"
  git clone https://github.com/libyal/${LIB}.git
done

echo "=== Verifying self-named subdirs ==="
for LIB in libcerror libcthreads libcdata libclocale libcnotify libcpath libcsplit libbfio libfcache libfdata libvmdk; do
  if [ -d "${LIB}/${LIB}" ]; then
    echo "OK: ${LIB}/${LIB}/"
  else
    echo "MISSING: ${LIB}/${LIB}/"
  fi
done
SCRIPT

bash "${BUILD_ROOT}/LIB-VMDK-SPM/clone.sh"
```

All 11 should report `OK`. If any report `MISSING`, delete that repo and reclone.

---

## Step 2 — Apply macFUSE 5.x Patch to vmdkmount

macFUSE 5.x uses Darwin-specific FUSE types that are not ABI-compatible with the standard POSIX types that `vmdktools/mount_fuse.c` was written against. The same boundary layer approach used for libewf applies here.

### The Problem

Build fails with:
```
incompatible function pointer types: fuse_fill_dir_t vs fuse_darwin_fill_dir_t
incompatible pointer types: struct stat * vs struct fuse_darwin_attr *
```

### Field Mapping

`fuse_darwin_attr` (macFUSE 5.x) has different field names from `struct stat`:

| POSIX `struct stat` | `fuse_darwin_attr` |
|---|---|
| `st_size` | `size` |
| `st_mode` | `mode` |
| `st_nlink` | `nlink` |
| `st_uid` | `uid` |
| `st_gid` | `gid` |
| `st_atimespec` | `atimespec` |
| `st_mtimespec` | `mtimespec` |
| `st_ctimespec` | `ctimespec` |

### Apply the Patch

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

cat > "${BUILD_ROOT}/LIB-VMDK-SPM/patch_fuse.sh" << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go
export SRC="${BUILD_ROOT}/LIB-VMDK-SPM/src/libvmdk"

echo "=== Step 1: Fix fuse_fill_dir_t in signatures ==="
sed -i '' 's/fuse_fill_dir_t filler/fuse_darwin_fill_dir_t filler/g' \
    "${SRC}/vmdktools/mount_fuse.c"
sed -i '' 's/fuse_fill_dir_t filler/fuse_darwin_fill_dir_t filler/g' \
    "${SRC}/vmdktools/mount_fuse.h"

echo "=== Step 2: Fix struct stat * in function signatures ==="
sed -i '' 's/^     struct stat \*stat_info,$/     struct fuse_darwin_attr *stat_info,/' \
    "${SRC}/vmdktools/mount_fuse.h"
sed -i '' 's/^     struct stat \*stat_info );$/     struct fuse_darwin_attr *stat_info );/' \
    "${SRC}/vmdktools/mount_fuse.h"
sed -i '' 's/^     struct stat \*stat_info,$/     struct fuse_darwin_attr *stat_info,/' \
    "${SRC}/vmdktools/mount_fuse.c"
sed -i '' 's/^     struct stat \*stat_info )$/     struct fuse_darwin_attr *stat_info )/' \
    "${SRC}/vmdktools/mount_fuse.c"

echo "=== Step 3: Apply boundary layer in mount_fuse_set_stat_info ==="
python3 << 'PYEOF'
import os

path = os.path.join(os.environ["SRC"], "vmdktools", "mount_fuse.c")
with open(path, "r") as f:
    content = f.read()

old_body = "\tstat_info->st_size  = (off_t) size;\n\tstat_info->st_mode  = file_mode;\n\n\tif( ( file_mode & 0x4000 ) != 0 )\n\t{\n\t\tstat_info->st_nlink = 2;\n\t}\n\telse\n\t{\n\t\tstat_info->st_nlink = 1;\n\t}\n#if defined( HAVE_GETEUID )\n\tstat_info->st_uid = geteuid();\n#endif\n#if defined( HAVE_GETEGID )\n\tstat_info->st_gid = getegid();\n#endif\n\n\tstat_info->st_atime = access_time / 1000000000;\n\tstat_info->st_ctime = inode_change_time / 1000000000;\n\tstat_info->st_mtime = modification_time / 1000000000;\n\n#if defined( STAT_HAVE_NSEC )\n\tstat_info->st_atime_nsec = access_time % 1000000000;\n\tstat_info->st_ctime_nsec = inode_change_time % 1000000000;\n\tstat_info->st_mtime_nsec = modification_time % 1000000000;\n#endif"

new_body = "\tstruct stat posix_stat;\n\tmemset( &posix_stat, 0, sizeof( struct stat ) );\n\n\tposix_stat.st_size  = (off_t) size;\n\tposix_stat.st_mode  = file_mode;\n\n\tif( ( file_mode & 0x4000 ) != 0 )\n\t{\n\t\tposix_stat.st_nlink = 2;\n\t}\n\telse\n\t{\n\t\tposix_stat.st_nlink = 1;\n\t}\n#if defined( HAVE_GETEUID )\n\tposix_stat.st_uid = geteuid();\n#endif\n#if defined( HAVE_GETEGID )\n\tposix_stat.st_gid = getegid();\n#endif\n\n\tposix_stat.st_atime = access_time / 1000000000;\n\tposix_stat.st_ctime = inode_change_time / 1000000000;\n\tposix_stat.st_mtime = modification_time / 1000000000;\n\n\t/* Boundary layer: struct stat -> fuse_darwin_attr */\n\tstat_info->size      = posix_stat.st_size;\n\tstat_info->mode      = posix_stat.st_mode;\n\tstat_info->nlink     = posix_stat.st_nlink;\n\tstat_info->uid       = posix_stat.st_uid;\n\tstat_info->gid       = posix_stat.st_gid;\n\tstat_info->atimespec = posix_stat.st_atimespec;\n\tstat_info->mtimespec = posix_stat.st_mtimespec;\n\tstat_info->ctimespec = posix_stat.st_ctimespec;"

if old_body in content:
    content = content.replace(old_body, new_body)
    print("Boundary layer applied")
else:
    print("ERROR: old body not found")
    exit(1)

with open(path, "w") as f:
    f.write(content)
PYEOF

echo "=== Step 4: Fix readdir local variable and allocation ==="
python3 << 'PYEOF'
import os

path = os.path.join(os.environ["SRC"], "vmdktools", "mount_fuse.c")
with open(path, "r") as f:
    content = f.read()

content = content.replace(
    "struct stat *stat_info                = NULL;",
    "struct fuse_darwin_attr *stat_info    = NULL;"
)
content = content.replace(
    "\tstat_info = memory_allocate_structure(\n\t             struct stat );",
    "\tstat_info = memory_allocate_structure(\n\t             struct fuse_darwin_attr );"
)
content = content.replace(
    "    sizeof( struct stat ) ) == NULL )",
    "    sizeof( struct fuse_darwin_attr ) ) == NULL )"
)

with open(path, "w") as f:
    f.write(content)
print("readdir fixes applied")
PYEOF

echo "=== Patch complete ==="
SCRIPT

bash "${BUILD_ROOT}/LIB-VMDK-SPM/patch_fuse.sh"
```

---

## Step 3 — Build arm64 (with FUSE)

> The build includes `-I/usr/local/include` and `-L/usr/local/lib` so `pkg-config` can find `fuse3` and the FUSE headers are found during compilation.

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

cat > "${BUILD_ROOT}/LIB-VMDK-SPM/build_arm64.sh" << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

PREFIX="${BUILD_ROOT}/LIB-VMDK-SPM/build/arm64/prefix"
SRC="${BUILD_ROOT}/LIB-VMDK-SPM/src"

CC="$(xcrun -f clang)"
CFLAGS="-arch arm64 -mmacosx-version-min=12.0 -isysroot $(xcrun --sdk macosx --show-sdk-path) -I/usr/local/include"
LDFLAGS="-arch arm64 -L/usr/local/lib"

export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:${PREFIX}/lib/pkgconfig"
export PKG_CONFIG_LIBDIR="/usr/local/lib/pkgconfig:${PREFIX}/lib/pkgconfig"

for LIB in libcerror libcthreads libcdata libclocale libcnotify libcpath libcsplit libbfio libfcache libfdata libvmdk; do
  echo ""
  echo "=== Building ${LIB} (arm64) ==="
  cd "${SRC}/${LIB}"

  make distclean 2>/dev/null || true
  rm -f config.cache
  rm -rf autom4te.cache

  bash synclibs.sh
  bash autogen.sh 2>&1 | tail -2
  test -f configure || { echo "ERROR: configure missing for ${LIB}"; exit 1; }

  if [ "$LIB" = "libvmdk" ]; then
    ./configure \
      --host="aarch64-apple-darwin" \
      --prefix="${PREFIX}" \
      --enable-static \
      --with-pic \
      --with-libcerror="${PREFIX}" \
      --with-libfuse \
      CC="${CC}" \
      CFLAGS="${CFLAGS}" \
      LDFLAGS="${LDFLAGS}"
  else
    ./configure \
      --host="aarch64-apple-darwin" \
      --prefix="${PREFIX}" \
      --enable-static \
      --with-pic \
      --with-libcerror="${PREFIX}" \
      CC="${CC}" \
      CFLAGS="${CFLAGS}" \
      LDFLAGS="${LDFLAGS}"
  fi

  make -j"$(sysctl -n hw.logicalcpu)"
  make install
  cd -
done

echo ""
echo "=== arm64 build complete ==="
echo "=== Checking vmdkmount FUSE linkage ==="
otool -L "${PREFIX}/bin/vmdkmount" | grep -i fuse
SCRIPT

bash "${BUILD_ROOT}/LIB-VMDK-SPM/build_arm64.sh" 2>&1 | tee "${BUILD_ROOT}/LIB-VMDK-SPM/build_arm64.log"
```

Expected: `vmdkmount` links against `libfuse3.4.dylib`.

---

## Step 4 — Build x86_64 (with FUSE)

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

cat > "${BUILD_ROOT}/LIB-VMDK-SPM/build_x86_64.sh" << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

PREFIX="${BUILD_ROOT}/LIB-VMDK-SPM/build/x86_64/prefix"
SRC="${BUILD_ROOT}/LIB-VMDK-SPM/src"

CC="$(xcrun -f clang)"
CFLAGS="-arch x86_64 -mmacosx-version-min=12.0 -isysroot $(xcrun --sdk macosx --show-sdk-path) -I/usr/local/include"
LDFLAGS="-arch x86_64 -L/usr/local/lib"

unset PKG_CONFIG_PATH
unset PKG_CONFIG_LIBDIR
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:${PREFIX}/lib/pkgconfig"
export PKG_CONFIG_LIBDIR="/usr/local/lib/pkgconfig:${PREFIX}/lib/pkgconfig"

for LIB in libcerror libcthreads libcdata libclocale libcnotify libcpath libcsplit libbfio libfcache libfdata libvmdk; do
  echo ""
  echo "=== Building ${LIB} (x86_64) ==="
  cd "${SRC}/${LIB}"

  make distclean 2>/dev/null || true
  rm -f config.cache
  rm -rf autom4te.cache

  bash synclibs.sh
  bash autogen.sh 2>&1 | tail -2
  test -f configure || { echo "ERROR: configure missing for ${LIB}"; exit 1; }

  if [ "$LIB" = "libvmdk" ]; then
    ./configure \
      --host="x86_64-apple-darwin" \
      --prefix="${PREFIX}" \
      --enable-static \
      --with-pic \
      --with-libcerror="${PREFIX}" \
      --with-libfuse \
      CC="${CC}" \
      CFLAGS="${CFLAGS}" \
      LDFLAGS="${LDFLAGS}"
  else
    ./configure \
      --host="x86_64-apple-darwin" \
      --prefix="${PREFIX}" \
      --enable-static \
      --with-pic \
      --with-libcerror="${PREFIX}" \
      CC="${CC}" \
      CFLAGS="${CFLAGS}" \
      LDFLAGS="${LDFLAGS}"
  fi

  make -j"$(sysctl -n hw.logicalcpu)"
  make install
  cd -
done

echo ""
echo "=== x86_64 build complete ==="
echo "=== Checking vmdkmount FUSE linkage ==="
otool -L "${PREFIX}/bin/vmdkmount" | grep -i fuse
SCRIPT

bash "${BUILD_ROOT}/LIB-VMDK-SPM/build_x86_64.sh" 2>&1 | tee "${BUILD_ROOT}/LIB-VMDK-SPM/build_x86_64.log"
```

---

## Step 5 — lipo Universal Binaries

Lipo both static libraries and tools:

```bash
BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

cat > "${BUILD_ROOT}/LIB-VMDK-SPM/lipo.sh" << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

BUILD_ROOT="${HOME}/forensic-libs"   # where build artifacts go
SPM_ROOT="${HOME}/forensic-spm"      # where SPM packages go

ARM64="${BUILD_ROOT}/LIB-VMDK-SPM/build/arm64/prefix/lib"
X86="${BUILD_ROOT}/LIB-VMDK-SPM/build/x86_64/prefix/lib"
ARM64BIN="${BUILD_ROOT}/LIB-VMDK-SPM/build/arm64/prefix/bin"
X86BIN="${BUILD_ROOT}/LIB-VMDK-SPM/build/x86_64/prefix/bin"
UNIV="${BUILD_ROOT}/LIB-VMDK-SPM/universal"

mkdir -p "$UNIV"

for A in "${ARM64}"/*.a; do
  LIB=$(basename "$A")
  echo "lipo → ${LIB}"
  lipo -create "${ARM64}/${LIB}" "${X86}/${LIB}" -output "${UNIV}/${LIB}"
done

for TOOL in vmdkmount vmdkinfo; do
  echo "lipo → ${TOOL}"
  lipo -create "${ARM64BIN}/${TOOL}" "${X86BIN}/${TOOL}" -output "${UNIV}/${TOOL}"
done

echo ""
echo "=== Universal libs ==="
for A in "${UNIV}"/*.a; do
  echo -n "$(basename $A): "
  lipo -info "$A"
done
echo ""
echo "=== Universal tools ==="
for T in "${UNIV}"/vmdk*; do
  echo -n "$(basename $T): "
  lipo -info "$T"
  otool -L "$T" | grep fuse
done
SCRIPT

bash "${BUILD_ROOT}/LIB-VMDK-SPM/lipo.sh"
```

Expected: all `.a` files and both tools show `x86_64 arm64`, tools link against `libfuse3.4.dylib`.

---

## Key Lessons Learned

| Issue | Fix |
|---|---|
| GitHub archive tarballs missing `configure` | Use `git clone`, not tarballs |
| `--depth=1` clones also missing template files | Use full `git clone` without `--depth` |
| `autoreconf -fi` fails with missing template files | Use `./autogen.sh` not `autoreconf -fi` directly |
| `synclibs.sh` must run before `autogen.sh` | It clones embedded dependency source into subdirs |
| `--with-libX` flags cause configure to fail for most deps | Only pass `--with-libcerror`; let everything else use embedded local sources |
| `config.cache` persists stale results across runs | Always `rm -f config.cache && rm -rf autom4te.cache` before reconfiguring |
| bash 3.2 on macOS rejects associative arrays | Write loops as functions in script files, not inline in zsh |
| arm64 pkgconfig bleeding into x86_64 build | `unset PKG_CONFIG_PATH` before setting x86_64 prefix |
| `vmdkmount` exits 1: "No sub system to mount" | Binary was built without FUSE — rebuild with `--with-libfuse` |
| `pkg-config` can't find `fuse3` | Add `/usr/local/lib/pkgconfig` to `PKG_CONFIG_PATH` |
| macFUSE 5.x `fuse_darwin_attr` field name mismatch | Field names are `size`, `mode`, `nlink` etc. — NOT `fa_size`, `fa_mode` as in older docs |
| `fuse_fill_dir_t` type mismatch | Replace with `fuse_darwin_fill_dir_t` in signatures only |
| `struct stat *` in readdir local var | Change to `struct fuse_darwin_attr *` and update `memory_allocate_structure` |
