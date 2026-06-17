# Chapter 5 — Compiling a Cross-Toolchain

> Repository: `github.com/Abeerithm/LFS-13.0-Systemd-Build-Notes`  
> Local repository path: `~/Projects/LFS-13.0-Systemd-Build-Notes`  
> Review branch: `docs-audit-lfs13`  
> Status: **Draft for review — do not commit or push yet**  
> Book source: *Linux From Scratch — Version 13.0-systemd*, Chapter 5  
> Actual LFS mount point used in this build: `/mnt/lfs`  
> Build user: `lfs`

---

## 5.1 Objective

This chapter builds the initial cross-toolchain used to isolate the future LFS system from the Debian host.

The required outcome is:

1. cross Binutils installed under `/mnt/lfs/tools`;
2. cross GCC installed under `/mnt/lfs/tools`;
3. sanitized Linux API headers installed under `/mnt/lfs/usr/include`;
4. a temporary Glibc installed under `/mnt/lfs/usr`;
5. a verified temporary C toolchain that links against the LFS filesystem rather than the Debian host;
6. a temporary Libstdc++ installed for later C++ builds.

This chapter must be completed as the unprivileged user:

```text
lfs
```

Do **not** build Chapter 5 as `root`.

---

## Documentation convention

Every package section separates:

### Official book command
The command sequence from LFS 13.0-systemd.

### Executed in this build
Commands or outputs confirmed from the actual project history.

### Environment-specific note
Adjustments required for this Debian VirtualBox build.

---

## 5.1.1 Preflight checks

Before extracting any Chapter 5 package, verify the shell environment.

Run as `lfs`:

```bash
whoami
echo "$LFS"
echo "$LFS_TGT"
echo "$PATH" | tr ':' '\n'
umask
set -o | grep hashall
```

Expected values:

```text
whoami:      lfs
LFS:         /mnt/lfs
LFS_TGT:     x86_64-lfs-linux-gnu
PATH first:  /mnt/lfs/tools/bin
umask:       0022
hashall:     off
```

Verify the required source archives:

```bash
cd "$LFS/sources"

ls -1 \
  binutils-2.46.0.tar.xz \
  gcc-15.2.0.tar.xz \
  linux-6.18.10.tar.xz \
  glibc-2.43.tar.xz \
  mpfr-4.2.2.tar.xz \
  gmp-6.3.0.tar.xz \
  mpc-1.3.1.tar.gz \
  glibc-fhs-1.patch
```

### Important safety rule

Before any install command that uses:

```bash
DESTDIR=$LFS
```

verify again:

```bash
whoami
echo "$LFS"
```

Expected:

```text
lfs
/mnt/lfs
```

A wrong `LFS` value or running as `root` could install temporary system files into the Debian host.

---

## 5.2 Binutils 2.46.0 — Pass 1

### Purpose

Binutils provides the linker, assembler, and object-file utilities required by GCC and Glibc.

It must be the first compiled package because GCC and Glibc inspect the available linker and assembler during their own configuration.

Approximate build time:

```text
1 SBU
```

Required disk space:

```text
691 MB
```

---

### 5.2.1 Extract the source

```bash
cd "$LFS/sources"

rm -rf binutils-2.46.0
tar -xf binutils-2.46.0.tar.xz
cd binutils-2.46.0
```

Create a dedicated build directory:

```bash
mkdir -v build
cd build
```

---

### 5.2.2 Configure Binutils

### Official book command

```bash
../configure --prefix=$LFS/tools \
  --with-sysroot=$LFS \
  --target=$LFS_TGT \
  --disable-nls \
  --enable-gprofng=no \
  --disable-werror \
  --enable-new-dtags \
  --enable-default-hash-style=gnu
```

### Meaning of the main options

| Option | Purpose |
|---|---|
| `--prefix=$LFS/tools` | Installs temporary cross tools under `/mnt/lfs/tools` |
| `--with-sysroot=$LFS` | Makes the linker use `/mnt/lfs` as the future system root |
| `--target=$LFS_TGT` | Builds tools for `x86_64-lfs-linux-gnu` |
| `--disable-nls` | Disables translations for temporary tools |
| `--enable-gprofng=no` | Skips a profiler not required at this stage |
| `--disable-werror` | Prevents compiler warnings from aborting the build |
| `--enable-new-dtags` | Uses `RUNPATH` rather than legacy `RPATH` |
| `--enable-default-hash-style=gnu` | Uses the GNU ELF hash style by default |

---

### 5.2.3 Build and install

```bash
make
make install
```

### Optional SBU measurement

To use Binutils Pass 1 as the SBU baseline:

```bash
time {
  ../configure --prefix=$LFS/tools \
    --with-sysroot=$LFS \
    --target=$LFS_TGT \
    --disable-nls \
    --enable-gprofng=no \
    --disable-werror \
    --enable-new-dtags \
    --enable-default-hash-style=gnu &&
  make &&
  make install
}
```

Use either the separate commands or the timed block, not both.

---

### 5.2.4 Verify Binutils Pass 1

```bash
$LFS_TGT-ld --version | head -n 1
$LFS_TGT-as --version | head -n 1

ls -l "$LFS/tools/bin/"{ld,as}

$LFS_TGT-ld --verbose | grep SEARCH
```

### Executed in this build

The retained build history confirmed:

```text
GNU ld (GNU Binutils) 2.46.0.20260210
GNU assembler (GNU Binutils) 2.46.0.20260210
```

The tools were installed under:

```text
/mnt/lfs/tools/bin
```

---

### 5.2.5 Clean the extracted tree

```bash
cd "$LFS/sources"
rm -rf binutils-2.46.0
```

---

## 5.3 GCC 15.2.0 — Pass 1

### Purpose

This is the first temporary GCC build.

It produces a cross C/C++ compiler that is not yet dependent on a target Glibc.

Approximate build time:

```text
3.8 SBU
```

Required disk space:

```text
5.4 GB
```

---

### 5.3.1 Extract GCC and its internal math dependencies

```bash
cd "$LFS/sources"

rm -rf gcc-15.2.0
tar -xf gcc-15.2.0.tar.xz
cd gcc-15.2.0
```

Extract MPFR:

```bash
tar -xf ../mpfr-4.2.2.tar.xz
mv -v mpfr-4.2.2 mpfr
```

Extract GMP:

```bash
tar -xf ../gmp-6.3.0.tar.xz
mv -v gmp-6.3.0 gmp
```

Extract MPC:

```bash
tar -xf ../mpc-1.3.1.tar.gz
mv -v mpc-1.3.1 mpc
```

### Why these libraries are embedded here

GCC requires GMP, MPFR, and MPC.

Placing them inside the GCC source tree allows the GCC build system to build and use them automatically without relying on host versions.

---

### 5.3.2 Correct the x86_64 library directory name

Run:

```bash
case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' \
        -i.orig gcc/config/i386/t-linux64
  ;;
esac
```

### Why this matters

LFS deliberately avoids:

```text
/usr/lib64
```

The x86_64 build must use:

```text
lib
```

instead of:

```text
lib64
```

Verify the change:

```bash
diff -u gcc/config/i386/t-linux64{.orig,} || true
```

---

### 5.3.3 Create a dedicated build directory

```bash
mkdir -v build
cd build
```

---

### 5.3.4 Configure GCC Pass 1

### Official book command

```bash
../configure \
  --target=$LFS_TGT \
  --prefix=$LFS/tools \
  --with-glibc-version=2.43 \
  --with-sysroot=$LFS \
  --with-newlib \
  --without-headers \
  --enable-default-pie \
  --enable-default-ssp \
  --disable-nls \
  --disable-shared \
  --disable-multilib \
  --disable-threads \
  --disable-libatomic \
  --disable-libgomp \
  --disable-libquadmath \
  --disable-libssp \
  --disable-libvtv \
  --disable-libstdcxx \
  --enable-languages=c,c++
```

### Meaning of the main options

| Option | Purpose |
|---|---|
| `--target=$LFS_TGT` | Builds a compiler for the LFS target triplet |
| `--prefix=$LFS/tools` | Installs the temporary compiler under `/mnt/lfs/tools` |
| `--with-glibc-version=2.43` | Declares the target Glibc version |
| `--with-sysroot=$LFS` | Makes the compiler search `/mnt/lfs` for target files |
| `--with-newlib` | Prevents libc-dependent code in `libgcc` before Glibc exists |
| `--without-headers` | Avoids requiring target headers during this first pass |
| `--enable-default-pie` | Enables position-independent executables by default |
| `--enable-default-ssp` | Enables stack-smashing protection by default |
| `--disable-shared` | Avoids target shared-library dependencies before Glibc exists |
| `--disable-multilib` | Builds a pure 64-bit LFS toolchain |
| `--disable-libstdcxx` | Defers Libstdc++ until after Glibc is installed |
| `--enable-languages=c,c++` | Builds only the languages required for LFS |

---

### 5.3.5 Build and install GCC Pass 1

```bash
make
make install
```

---

### 5.3.6 Create the complete internal `limits.h`

Return to the GCC source root:

```bash
cd ..
```

Create the full internal limits header:

```bash
cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/include/limits.h
```

### Why this step is required

During GCC Pass 1, `$LFS/usr/include/limits.h` does not exist yet.

GCC therefore installs an incomplete internal header.

This command creates the complete internal version required by later steps.

---

### 5.3.7 Verify GCC Pass 1

```bash
$LFS_TGT-gcc --version | head -n 1

$LFS_TGT-gcc -print-prog-name=ld

$LFS_TGT-gcc -print-libgcc-file-name
```

Expected linker path pattern:

```text
/mnt/lfs/tools/...
```

Verify the compiler exists:

```bash
ls -l "$LFS/tools/bin/"$LFS_TGT-gcc
```

---

### 5.3.8 Clean the extracted tree

```bash
cd "$LFS/sources"
rm -rf gcc-15.2.0
```

---

## 5.4 Linux 6.18.10 API Headers

### Purpose

The Linux API headers expose kernel interfaces required by Glibc.

These are sanitized user-space headers, not a compiled kernel.

Approximate build time:

```text
less than 0.1 SBU
```

Required disk space:

```text
1.7 GB
```

---

### 5.4.1 Extract the Linux source

```bash
cd "$LFS/sources"

rm -rf linux-6.18.10
tar -xf linux-6.18.10.tar.xz
cd linux-6.18.10
```

---

### 5.4.2 Clean stale files

```bash
make mrproper
```

### Why this matters

This ensures that no stale generated files remain inside the source tree.

---

### 5.4.3 Generate sanitized headers

```bash
make headers
```

Remove files that are not C header files:

```bash
find usr/include -type f ! -name '*.h' -delete
```

Copy the sanitized headers into LFS:

```bash
cp -rv usr/include "$LFS/usr"
```

---

### 5.4.4 Verify Linux API headers

```bash
test -f "$LFS/usr/include/linux/version.h" \
  && echo "OK: Linux API headers installed"
```

Inspect a sample:

```bash
find "$LFS/usr/include" -type f | head
```

Expected directories include:

```text
/mnt/lfs/usr/include/asm
/mnt/lfs/usr/include/asm-generic
/mnt/lfs/usr/include/linux
/mnt/lfs/usr/include/misc
/mnt/lfs/usr/include/scsi
/mnt/lfs/usr/include/sound
```

---

### 5.4.5 Clean the extracted tree

```bash
cd "$LFS/sources"
rm -rf linux-6.18.10
```

---

## 5.5 Glibc 2.43 — Temporary Cross Build

### Purpose

Glibc is the main C library.

This is the first package in Chapter 5 that is truly cross-compiled for the future LFS system.

Approximate build time:

```text
1.4 SBU
```

Required disk space:

```text
890 MB
```

---

### 5.5.1 Extract the source

```bash
cd "$LFS/sources"

rm -rf glibc-2.43
tar -xf glibc-2.43.tar.xz
cd glibc-2.43
```

---

### 5.5.2 Create dynamic-loader compatibility symlinks

```bash
case $(uname -m) in
  i?86)
    ln -sfv ld-linux.so.2 "$LFS/lib/ld-lsb.so.3"
  ;;
  x86_64)
    ln -sfv ../lib/ld-linux-x86-64.so.2 "$LFS/lib64"
    ln -sfv ../lib/ld-linux-x86-64.so.2 \
      "$LFS/lib64/ld-lsb-x86-64.so.3"
  ;;
esac
```

### Verify

```bash
ls -l "$LFS/lib64"
```

Expected on x86_64:

```text
ld-linux-x86-64.so.2 -> ../lib/ld-linux-x86-64.so.2
ld-lsb-x86-64.so.3   -> ../lib/ld-linux-x86-64.so.2
```

---

### 5.5.3 Apply the FHS patch

### Official book intent

Apply:

```text
glibc-fhs-1.patch
```

so Glibc programs use FHS-compliant runtime paths instead of `/var/db`.

### Executed in this build

The patch file was stored beside the extracted source tree in:

```text
/mnt/lfs/sources
```

The command that succeeded was:

```bash
patch -Np1 -i ../glibc-fhs-1.patch
```

### Environment-specific note

Use the relative parent path:

```text
../glibc-fhs-1.patch
```

when the current directory is:

```text
/mnt/lfs/sources/glibc-2.43
```

Verify that the patch applied without a fatal error.

---

### 5.5.4 Create a dedicated build directory

```bash
mkdir -v build
cd build
```

Ensure `ldconfig` and `sln` install under `/usr/sbin`:

```bash
echo "rootsbindir=/usr/sbin" > configparms
```

Verify:

```bash
cat configparms
```

Expected:

```text
rootsbindir=/usr/sbin
```

---

### 5.5.5 Configure temporary Glibc

```bash
../configure \
  --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(../scripts/config.guess) \
  --disable-nscd \
  libc_cv_slibdir=/usr/lib \
  --enable-kernel=5.4
```

### Meaning of the main options

| Option | Purpose |
|---|---|
| `--prefix=/usr` | Installs Glibc under the future LFS `/usr` |
| `--host=$LFS_TGT` | Uses target-prefixed cross tools |
| `--build=$(../scripts/config.guess)` | Enables cross-compilation mode |
| `--disable-nscd` | Skips the obsolete name-service cache daemon |
| `libc_cv_slibdir=/usr/lib` | Prevents use of `/lib64` or `/usr/lib64` |
| `--enable-kernel=5.4` | Targets Linux kernel 5.4 or newer |

### Possible harmless warning

A warning about missing or incompatible:

```text
msgfmt
```

may appear.

This is generally harmless at this temporary stage.

---

### 5.5.6 Build and install temporary Glibc

Compile:

```bash
make
```

If a parallel build fails, rerun:

```bash
make -j1
```

Before installation, verify:

```bash
whoami
echo "$LFS"
```

Expected:

```text
lfs
/mnt/lfs
```

Install:

```bash
make DESTDIR=$LFS install
```

### Critical safety warning

Do not run the install command if:

```text
whoami != lfs
```

or:

```text
LFS != /mnt/lfs
```

A wrong value can damage the Debian host.

---

### 5.5.7 Fix the hard-coded loader path in `ldd`

```bash
sed '/RTLDLIST=/s@/usr@@g' -i "$LFS/usr/bin/ldd"
```

Verify:

```bash
grep '^RTLDLIST=' "$LFS/usr/bin/ldd"
```

The loader paths should not begin with:

```text
/usr
```

---

### 5.5.8 Run the temporary-toolchain sanity checks

Create a test executable and capture the verbose linker output:

```bash
echo 'int main(){}' | \
  $LFS_TGT-gcc -x c - -v -Wl,--verbose &> dummy.log
```

Check the requested program interpreter:

```bash
readelf -l a.out | grep ': /lib'
```

Expected on x86_64:

```text
[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

### Executed in this build

The retained build history confirmed the expected interpreter:

```text
[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

Check the C runtime start files:

```bash
grep -E -o "$LFS/lib.*/S?crt[1in].*succeeded" dummy.log
```

Expected:

```text
/mnt/lfs/lib/../lib/Scrt1.o succeeded
/mnt/lfs/lib/../lib/crti.o succeeded
/mnt/lfs/lib/../lib/crtn.o succeeded
```

Check the compiler header search order:

```bash
grep -B3 "^ $LFS/usr/include" dummy.log
```

Expected:

```text
#include <...> search starts here:
 /mnt/lfs/tools/lib/gcc/x86_64-lfs-linux-gnu/15.2.0/include
 /mnt/lfs/tools/lib/gcc/x86_64-lfs-linux-gnu/15.2.0/include-fixed
 /mnt/lfs/usr/include
```

Check linker search paths:

```bash
grep 'SEARCH.*/usr/lib' dummy.log | sed 's|; |\n|g'
```

The important requirement is that sysroot-sensitive paths begin with:

```text
=
```

Check the selected C library:

```bash
grep "/lib.*/libc.so.6 " dummy.log
```

Expected:

```text
attempt to open /mnt/lfs/usr/lib/libc.so.6 succeeded
```

Check the dynamic linker:

```bash
grep found dummy.log
```

Expected:

```text
found ld-linux-x86-64.so.2 at /mnt/lfs/usr/lib/ld-linux-x86-64.so.2
```

### Required stopping rule

Do not continue if the sanity-check output differs materially from the expected paths.

Investigate and correct the toolchain first.

---

### 5.5.9 Clean the sanity-check files and source tree

```bash
rm -v a.out dummy.log

cd "$LFS/sources"
rm -rf glibc-2.43
```

---

## 5.6 Libstdc++ from GCC 15.2.0

### Purpose

Libstdc++ is the standard C++ library.

It could not be built during GCC Pass 1 because target Glibc did not exist yet.

Approximate build time:

```text
0.2 SBU
```

Required disk space:

```text
1.3 GB
```

---

### 5.6.1 Extract GCC again

```bash
cd "$LFS/sources"

rm -rf gcc-15.2.0
tar -xf gcc-15.2.0.tar.xz
cd gcc-15.2.0
```

Create a dedicated build directory:

```bash
mkdir -v build
cd build
```

---

### 5.6.2 Configure Libstdc++

```bash
../libstdc++-v3/configure \
  --host=$LFS_TGT \
  --build=$(../config.guess) \
  --prefix=/usr \
  --disable-multilib \
  --disable-nls \
  --disable-libstdcxx-pch \
  --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/15.2.0
```

### Meaning of the main options

| Option | Purpose |
|---|---|
| `--host=$LFS_TGT` | Uses the cross compiler rather than Debian GCC |
| `--prefix=/usr` | Installs into the future LFS `/usr` |
| `--disable-multilib` | Keeps the toolchain pure 64-bit |
| `--disable-nls` | Skips translations for this temporary build |
| `--disable-libstdcxx-pch` | Skips unnecessary precompiled headers |
| `--with-gxx-include-dir=...` | Places C++ headers where the temporary GCC expects them |

---

### 5.6.3 Build and install Libstdc++

```bash
make
make DESTDIR=$LFS install
```

Remove libtool archive files because they are harmful during cross-compilation:

```bash
rm -v "$LFS/usr/lib/"lib{stdc++{,exp,fs},supc++}.la
```

---

### 5.6.4 Verify Libstdc++

```bash
ls -l "$LFS/usr/lib/"libstdc++.so*
```

Verify that the main libtool archive was removed:

```bash
test ! -e "$LFS/usr/lib/libstdc++.la" \
  && echo "OK: libstdc++.la removed"
```

Verify C++ headers:

```bash
test -d "$LFS/tools/$LFS_TGT/include/c++/15.2.0" \
  && echo "OK: temporary C++ headers installed"
```

---

### 5.6.5 Clean the extracted tree

```bash
cd "$LFS/sources"
rm -rf gcc-15.2.0
```

---

## 5.7 End-of-chapter verification

Run as `lfs`:

```bash
whoami
echo "$LFS"
echo "$LFS_TGT"
```

Expected:

```text
lfs
/mnt/lfs
x86_64-lfs-linux-gnu
```

Verify cross Binutils:

```bash
$LFS_TGT-ld --version | head -n 1
$LFS_TGT-as --version | head -n 1
```

Verify cross GCC:

```bash
$LFS_TGT-gcc --version | head -n 1
```

Verify Linux API headers:

```bash
test -f "$LFS/usr/include/linux/version.h" \
  && echo "OK: Linux API headers"
```

Verify Glibc:

```bash
test -e "$LFS/usr/lib/libc.so.6" \
  && echo "OK: temporary Glibc"
```

Verify Libstdc++:

```bash
test -e "$LFS/usr/lib/libstdc++.so" \
  && echo "OK: temporary Libstdc++"
```

Verify that LFS did not create `/usr/lib64`:

```bash
test ! -e "$LFS/usr/lib64" \
  && echo "OK: no /usr/lib64"
```

---

## 5.8 Full clean-rebuild sequence

Run this chapter as:

```text
lfs
```

not as `root`.

```bash
whoami
echo "$LFS"
echo "$LFS_TGT"
umask
set -o | grep hashall

cd "$LFS/sources"

# Binutils Pass 1
rm -rf binutils-2.46.0
tar -xf binutils-2.46.0.tar.xz
cd binutils-2.46.0
mkdir -v build
cd build

../configure --prefix=$LFS/tools \
  --with-sysroot=$LFS \
  --target=$LFS_TGT \
  --disable-nls \
  --enable-gprofng=no \
  --disable-werror \
  --enable-new-dtags \
  --enable-default-hash-style=gnu

make
make install

$LFS_TGT-ld --version | head -n 1
$LFS_TGT-as --version | head -n 1

cd "$LFS/sources"
rm -rf binutils-2.46.0

# GCC Pass 1
rm -rf gcc-15.2.0
tar -xf gcc-15.2.0.tar.xz
cd gcc-15.2.0

tar -xf ../mpfr-4.2.2.tar.xz
mv -v mpfr-4.2.2 mpfr

tar -xf ../gmp-6.3.0.tar.xz
mv -v gmp-6.3.0 gmp

tar -xf ../mpc-1.3.1.tar.gz
mv -v mpc-1.3.1 mpc

case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' \
        -i.orig gcc/config/i386/t-linux64
  ;;
esac

mkdir -v build
cd build

../configure \
  --target=$LFS_TGT \
  --prefix=$LFS/tools \
  --with-glibc-version=2.43 \
  --with-sysroot=$LFS \
  --with-newlib \
  --without-headers \
  --enable-default-pie \
  --enable-default-ssp \
  --disable-nls \
  --disable-shared \
  --disable-multilib \
  --disable-threads \
  --disable-libatomic \
  --disable-libgomp \
  --disable-libquadmath \
  --disable-libssp \
  --disable-libvtv \
  --disable-libstdcxx \
  --enable-languages=c,c++

make
make install

cd ..

cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/include/limits.h

$LFS_TGT-gcc --version | head -n 1
$LFS_TGT-gcc -print-prog-name=ld

cd "$LFS/sources"
rm -rf gcc-15.2.0

# Linux API Headers
rm -rf linux-6.18.10
tar -xf linux-6.18.10.tar.xz
cd linux-6.18.10

make mrproper
make headers
find usr/include -type f ! -name '*.h' -delete
cp -rv usr/include "$LFS/usr"

test -f "$LFS/usr/include/linux/version.h" \
  && echo "OK: Linux API headers installed"

cd "$LFS/sources"
rm -rf linux-6.18.10

# Temporary Glibc
rm -rf glibc-2.43
tar -xf glibc-2.43.tar.xz
cd glibc-2.43

case $(uname -m) in
  i?86)
    ln -sfv ld-linux.so.2 "$LFS/lib/ld-lsb.so.3"
  ;;
  x86_64)
    ln -sfv ../lib/ld-linux-x86-64.so.2 "$LFS/lib64"
    ln -sfv ../lib/ld-linux-x86-64.so.2 \
      "$LFS/lib64/ld-lsb-x86-64.so.3"
  ;;
esac

patch -Np1 -i ../glibc-fhs-1.patch

mkdir -v build
cd build

echo "rootsbindir=/usr/sbin" > configparms

../configure \
  --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(../scripts/config.guess) \
  --disable-nscd \
  libc_cv_slibdir=/usr/lib \
  --enable-kernel=5.4

make

whoami
echo "$LFS"

make DESTDIR=$LFS install

sed '/RTLDLIST=/s@/usr@@g' -i "$LFS/usr/bin/ldd"

echo 'int main(){}' | \
  $LFS_TGT-gcc -x c - -v -Wl,--verbose &> dummy.log

readelf -l a.out | grep ': /lib'

grep -E -o "$LFS/lib.*/S?crt[1in].*succeeded" dummy.log

grep -B3 "^ $LFS/usr/include" dummy.log

grep 'SEARCH.*/usr/lib' dummy.log | sed 's|; |\n|g'

grep "/lib.*/libc.so.6 " dummy.log

grep found dummy.log

rm -v a.out dummy.log

cd "$LFS/sources"
rm -rf glibc-2.43

# Temporary Libstdc++
rm -rf gcc-15.2.0
tar -xf gcc-15.2.0.tar.xz
cd gcc-15.2.0

mkdir -v build
cd build

../libstdc++-v3/configure \
  --host=$LFS_TGT \
  --build=$(../config.guess) \
  --prefix=/usr \
  --disable-multilib \
  --disable-nls \
  --disable-libstdcxx-pch \
  --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/15.2.0

make
make DESTDIR=$LFS install

rm -v "$LFS/usr/lib/"lib{stdc++{,exp,fs},supc++}.la

ls -l "$LFS/usr/lib/"libstdc++.so*

cd "$LFS/sources"
rm -rf gcc-15.2.0
```

---

## 5.9 Errors and lessons learned

### 1. Glibc Patch path must match the extracted-tree layout

**Observed layout:**

```text
/mnt/lfs/sources/glibc-2.43
/mnt/lfs/sources/glibc-fhs-1.patch
```

**Command that succeeded:**

```bash
patch -Np1 -i ../glibc-fhs-1.patch
```

**Repeat-build note:**

Do not blindly paste a patch path without checking the current directory.

---

### 2. Never build or install temporary Glibc as `root`

Before:

```bash
make DESTDIR=$LFS install
```

run:

```bash
whoami
echo "$LFS"
```

Expected:

```text
lfs
/mnt/lfs
```

---

### 3. If temporary Glibc fails during parallel compilation

Retry:

```bash
make -j1
```

This is a package-specific fallback, not a reason to globally remove the bounded `MAKEFLAGS` setting.

---

### 4. Do not create `/usr/lib64`

Verify:

```bash
test ! -e "$LFS/usr/lib64" \
  && echo "OK: no /usr/lib64"
```

---

### 5. Remove extracted package directories after each successful install

Keep the archives:

```text
*.tar.xz
*.tar.gz
```

Delete only extracted working trees.

This prevents stale files from affecting later passes.

---

## 5.10 GitHub evidence capture

Create a log directory:

```bash
REPO_ROOT=~/Projects/LFS-13.0-Systemd-Build-Notes

mkdir -pv "$REPO_ROOT/logs/chapter-05"
```

Capture environment checks:

```bash
{
  echo "whoami=$(whoami)"
  echo "LFS=$LFS"
  echo "LFS_TGT=$LFS_TGT"
  echo "umask=$(umask)"
  echo "PATH=$PATH"
  set -o | grep hashall
} | tee "$REPO_ROOT/logs/chapter-05/environment.txt"
```

Capture tool versions:

```bash
{
  $LFS_TGT-ld --version | head -n 1
  $LFS_TGT-as --version | head -n 1
  $LFS_TGT-gcc --version | head -n 1
} | tee "$REPO_ROOT/logs/chapter-05/tool-versions.txt"
```

Capture installed-file checks:

```bash
{
  test -f "$LFS/usr/include/linux/version.h" \
    && echo "OK: Linux API headers"

  test -e "$LFS/usr/lib/libc.so.6" \
    && echo "OK: temporary Glibc"

  test -e "$LFS/usr/lib/libstdc++.so" \
    && echo "OK: temporary Libstdc++"

  test ! -e "$LFS/usr/lib64" \
    && echo "OK: no /usr/lib64"
} | tee "$REPO_ROOT/logs/chapter-05/installed-file-checks.txt"
```

### Audit note

The original complete Chapter 5 terminal logs are not fully preserved in the retained conversation history.

The draft includes:

1. all official commands required for a clean rebuild;
2. the verified actual project path `/mnt/lfs`;
3. the confirmed Binutils version outputs;
4. the confirmed Glibc program-interpreter sanity-check output;
5. the actual Glibc patch path that succeeded in this environment.

---

## Chapter 5 approval checklist

Before moving to Chapter 6 documentation:

```text
[ ] Review all commands in this file
[ ] Confirm all Chapter 5 work runs as lfs, not root
[ ] Confirm Binutils Pass 1 versions
[ ] Confirm GCC Pass 1 exists under /mnt/lfs/tools
[ ] Confirm Linux API headers under /mnt/lfs/usr/include
[ ] Confirm Glibc patch path ../glibc-fhs-1.patch
[ ] Confirm Glibc sanity-check interpreter output
[ ] Confirm Glibc start files, header paths, libc, and linker outputs
[ ] Confirm temporary Libstdc++ libraries exist
[ ] Confirm /mnt/lfs/usr/lib64 does not exist
[ ] Capture logs/chapter-05 evidence
[ ] Do not commit or push until explicit approval
```

---

## Suggested repository placement

Place this reviewed file at:

```text
docs/chapter-05-compiling-cross-toolchain.md
```

Suggested companion evidence files:

```text
logs/chapter-05/environment.txt
logs/chapter-05/tool-versions.txt
logs/chapter-05/installed-file-checks.txt
```

---

## Current approval state

```text
DRAFT — CHAPTER 5 ONLY
DO NOT COMMIT
DO NOT PUSH
WAITING FOR REVIEW AND EVIDENCE CAPTURE
```
