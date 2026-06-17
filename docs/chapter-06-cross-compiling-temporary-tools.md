# Chapter 6 — Cross Compiling Temporary Tools

> Repository: `github.com/Abeerithm/LFS-13.0-Systemd-Build-Notes`  
> Local repository path: `~/Projects/LFS-13.0-Systemd-Build-Notes`  
> Review branch: `docs-audit-lfs13`  
> Status: **Draft for review — do not commit or push yet**  
> Book source: *Linux From Scratch — Version 13.0-systemd*, Chapter 6  
> Actual LFS mount point used in this build: `/mnt/lfs`  
> Build user: `lfs`

---

## 6.1 Objective

This chapter cross-compiles the basic temporary utilities required before entering `chroot`.

The key idea is:

```text
The tools are installed into their final locations under /mnt/lfs,
but they are not used yet.
```

During Chapter 6, basic tasks still rely on the Debian host tools.

The installed temporary tools become usable in Chapter 7 after entering:

```text
chroot
```

All commands in this chapter must be run as the unprivileged user:

```text
lfs
```

Do **not** run Chapter 6 as `root`.

---

## Documentation convention

Every package section separates:

### Official book command
The command sequence from LFS 13.0-systemd.

### Executed in this build
Commands or outputs confirmed from the actual project history.

### Environment-specific note
Adjustments or operational lessons specific to this Debian VirtualBox build.

---

## 6.1.1 Preflight checks

Before extracting the first package, verify the environment:

```bash
whoami
echo "$LFS"
echo "$LFS_TGT"
echo "$PATH" | tr ':' '\n'
umask
set -o | grep hashall
```

Expected:

```text
whoami:      lfs
LFS:         /mnt/lfs
LFS_TGT:     x86_64-lfs-linux-gnu
PATH first:  /mnt/lfs/tools/bin
umask:       0022
hashall:     off
```

Verify the required archives:

```bash
cd "$LFS/sources"

ls -1 \
  m4-1.4.21.tar.xz \
  ncurses-6.6.tar.gz \
  bash-5.3.tar.gz \
  coreutils-9.10.tar.xz \
  diffutils-3.12.tar.xz \
  file-5.46.tar.gz \
  findutils-4.10.0.tar.xz \
  gawk-5.3.2.tar.xz \
  grep-3.12.tar.xz \
  gzip-1.14.tar.xz \
  make-4.4.1.tar.gz \
  patch-2.8.tar.xz \
  sed-4.9.tar.xz \
  tar-1.35.tar.xz \
  xz-5.8.2.tar.xz \
  binutils-2.46.0.tar.xz \
  gcc-15.2.0.tar.xz \
  mpfr-4.2.2.tar.xz \
  gmp-6.3.0.tar.xz \
  mpc-1.3.1.tar.gz
```

### Important safety rule

Every install command in this chapter uses:

```bash
DESTDIR=$LFS
```

Before each install, verify:

```bash
whoami
echo "$LFS"
```

Expected:

```text
lfs
/mnt/lfs
```

A wrong value for `LFS`, especially while running as `root`, can damage the Debian host.

---

## 6.2 M4 1.4.21

### Purpose

M4 is a macro processor used by many build systems.

Approximate build time:

```text
0.1 SBU
```

Required disk space:

```text
39 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf m4-1.4.21
tar -xf m4-1.4.21.tar.xz
cd m4-1.4.21
```

### Configure

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(build-aux/config.guess)
```

### Build and install

```bash
make
make DESTDIR=$LFS install
```

### Verify installed file

```bash
ls -l "$LFS/usr/bin/m4"
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf m4-1.4.21
```

---

## 6.3 Ncurses 6.6

### Purpose

Ncurses provides terminal-independent handling of character screens.

Approximate build time:

```text
0.4 SBU
```

Required disk space:

```text
54 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf ncurses-6.6
tar -xf ncurses-6.6.tar.gz
cd ncurses-6.6
```

### Build the host `tic` utility first

The cross-compiled Ncurses installation needs a working `tic` utility on the Debian host.

Create a host build:

```bash
mkdir build
pushd build
  ../configure --prefix=$LFS/tools AWK=gawk
  make -C include
  make -C progs tic
  install progs/tic $LFS/tools/bin
popd
```

### Configure Ncurses for the LFS target

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(./config.guess) \
  --mandir=/usr/share/man \
  --with-manpage-format=normal \
  --with-shared \
  --without-normal \
  --with-cxx-shared \
  --without-debug \
  --without-ada \
  --disable-stripping \
  AWK=gawk
```

### Meaning of important options

| Option | Purpose |
|---|---|
| `--with-manpage-format=normal` | Prevents installation of compressed manual pages |
| `--with-shared` | Builds shared C libraries |
| `--without-normal` | Skips static C libraries |
| `--with-cxx-shared` | Builds shared C++ bindings and skips static C++ bindings |
| `--without-debug` | Skips debug libraries |
| `--without-ada` | Avoids Ada support not needed inside `chroot` |
| `--disable-stripping` | Prevents use of the host `strip` on cross-compiled binaries |
| `AWK=gawk` | Prevents use of host `mawk`, which may break the build |

### Build and install

```bash
make
make DESTDIR=$LFS install
```

### Create the compatibility link and force wide-character structures

```bash
ln -sv libncursesw.so "$LFS/usr/lib/libncurses.so"

sed -e 's/^#if.*XOPEN.*$/#if 1/' \
  -i "$LFS/usr/include/curses.h"
```

### Verify

```bash
ls -l "$LFS/usr/lib/libncurses.so"
ls -l "$LFS/usr/lib/"libncursesw.so*
grep -n '^#if 1' "$LFS/usr/include/curses.h" | head
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf ncurses-6.6
```

---

## 6.4 Bash 5.3

### Purpose

Bash is the Bourne-Again Shell.

Approximate build time:

```text
0.2 SBU
```

Required disk space:

```text
72 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf bash-5.3
tar -xf bash-5.3.tar.gz
cd bash-5.3
```

### Configure

```bash
./configure --prefix=/usr \
  --build=$(sh support/config.guess) \
  --host=$LFS_TGT \
  --without-bash-malloc
```

### Why `--without-bash-malloc` is required

Bash's own memory allocator is known to cause segmentation faults in some cases.

This option makes Bash use Glibc's allocator instead.

### Build and install

```bash
make
make DESTDIR=$LFS install
```

### Create `/bin/sh`

```bash
ln -sv bash "$LFS/bin/sh"
```

### Verify

```bash
ls -l "$LFS/usr/bin/bash"
ls -l "$LFS/bin/sh"
```

Expected:

```text
/mnt/lfs/bin/sh -> bash
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf bash-5.3
```

---

## 6.5 Coreutils 9.10

### Purpose

Coreutils provides the basic command-line utilities required by every Unix-like operating system.

Approximate build time:

```text
0.3 SBU
```

Required disk space:

```text
185 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf coreutils-9.10
tar -xf coreutils-9.10.tar.xz
cd coreutils-9.10
```

### Configure

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(build-aux/config.guess) \
  --enable-install-program=hostname \
  --enable-no-install-program=kill,uptime
```

### Why `hostname` is enabled

The Perl test suite later requires:

```text
hostname
```

### Build and install

```bash
make
make DESTDIR=$LFS install
```

### Move `chroot` and its manual page to the final expected locations

```bash
mv -v "$LFS/usr/bin/chroot" "$LFS/usr/sbin"

mkdir -pv "$LFS/usr/share/man/man8"

mv -v "$LFS/usr/share/man/man1/chroot.1" \
      "$LFS/usr/share/man/man8/chroot.8"

sed -i 's/"1"/"8"/' "$LFS/usr/share/man/man8/chroot.8"
```

### Verify

```bash
ls -l "$LFS/usr/sbin/chroot"
ls -l "$LFS/usr/share/man/man8/chroot.8"
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf coreutils-9.10
```

---

## 6.6 Diffutils 3.12

### Purpose

Diffutils compares files and directories.

Approximate build time:

```text
0.1 SBU
```

Required disk space:

```text
35 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf diffutils-3.12
tar -xf diffutils-3.12.tar.xz
cd diffutils-3.12
```

### Configure

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  gl_cv_func_strcasecmp_works=y \
  --build=$(./build-aux/config.guess)
```

### Why `gl_cv_func_strcasecmp_works=y` is required

The normal configure test needs to run a compiled target program.

That is not possible during cross-compilation.

The value is specified explicitly because Glibc 2.43 provides a working `strcasecmp`.

### Build and install

```bash
make
make DESTDIR=$LFS install
```

### Verify

```bash
ls -l "$LFS/usr/bin/"{cmp,diff,diff3,sdiff}
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf diffutils-3.12
```

---

## 6.7 File 5.46

### Purpose

The `file` utility identifies file types by inspecting file contents.

Approximate build time:

```text
0.1 SBU
```

Required disk space:

```text
43 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf file-5.46
tar -xf file-5.46.tar.gz
cd file-5.46
```

### Build a host-native temporary `file` utility

The signature database must be generated by the same version of `file` being built.

```bash
mkdir build
pushd build
  ../configure --disable-bzlib \
    --disable-libseccomp \
    --disable-xzlib \
    --disable-zlib
  make
popd
```

### Why the host-native copy disables optional libraries

The configure script may try to use host libraries when the corresponding headers are missing.

The disabled features are unnecessary for this temporary copy.

### Configure the target build

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(./config.guess)
```

### Build and install

```bash
make FILE_COMPILE=$(pwd)/build/src/file
make DESTDIR=$LFS install
```

### Remove the harmful libtool archive

```bash
rm -v "$LFS/usr/lib/libmagic.la"
```

### Verify

```bash
ls -l "$LFS/usr/bin/file"
test ! -e "$LFS/usr/lib/libmagic.la" \
  && echo "OK: libmagic.la removed"
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf file-5.46
```

---

## 6.8 Findutils 4.10.0

### Purpose

Findutils provides:

```text
find
locate
updatedb
xargs
```

Approximate build time:

```text
0.2 SBU
```

Required disk space:

```text
48 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf findutils-4.10.0
tar -xf findutils-4.10.0.tar.xz
cd findutils-4.10.0
```

### Configure

```bash
./configure --prefix=/usr \
  --localstatedir=/var/lib/locate \
  --host=$LFS_TGT \
  --build=$(build-aux/config.guess)
```

### Why `--localstatedir=/var/lib/locate` is used

This places the locate database in the FHS-compliant location:

```text
/var/lib/locate
```

### Build and install

```bash
make
make DESTDIR=$LFS install
```

### Verify

```bash
ls -l "$LFS/usr/bin/"{find,xargs,locate,updatedb}
```

### Executed in this build

The retained transcript confirmed:

```text
/mnt/lfs/usr/bin/find
/mnt/lfs/usr/bin/xargs
/mnt/lfs/usr/bin/locate
/mnt/lfs/usr/bin/updatedb
```

with successful installation under `/mnt/lfs/usr/bin`.

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf findutils-4.10.0
```

---

## 6.9 Gawk 5.3.2

### Purpose

Gawk manipulates text files using the AWK language.

Approximate build time:

```text
0.1 SBU
```

Required disk space:

```text
49 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf gawk-5.3.2
tar -xf gawk-5.3.2.tar.xz
cd gawk-5.3.2
```

### Prevent unneeded files from being installed

```bash
sed -i 's/extras//' Makefile.in
```

Verify:

```bash
grep extras Makefile.in
```

Expected:

```text
no output
```

### Executed in this build

The retained transcript confirmed that:

```bash
grep extras Makefile.in
```

returned no output.

### Configure, build, and install

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(build-aux/config.guess)

make
make DESTDIR=$LFS install
```

### Verify

```bash
ls -l "$LFS/usr/bin/"{gawk,awk}
```

### Executed in this build

The retained transcript confirmed:

```text
/mnt/lfs/usr/bin/awk -> gawk
/mnt/lfs/usr/bin/gawk
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf gawk-5.3.2
```

---

## 6.10 Grep 3.12

### Purpose

Grep searches the contents of files.

Approximate build time:

```text
0.1 SBU
```

Required disk space:

```text
32 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf grep-3.12
tar -xf grep-3.12.tar.xz
cd grep-3.12
```

### Configure, build, and install

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(./build-aux/config.guess)

make
make DESTDIR=$LFS install
```

### Verify

```bash
ls -l "$LFS/usr/bin/"{grep,egrep,fgrep}
```

### Executed in this build

The retained transcript confirmed:

```text
/mnt/lfs/usr/bin/grep
/mnt/lfs/usr/bin/egrep
/mnt/lfs/usr/bin/fgrep
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf grep-3.12
```

---

## 6.11 Gzip 1.14

### Purpose

Gzip compresses and decompresses files.

Approximate build time:

```text
0.1 SBU
```

Required disk space:

```text
12 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf gzip-1.14
tar -xf gzip-1.14.tar.xz
cd gzip-1.14
```

### Configure, build, and install

```bash
./configure --prefix=/usr --host=$LFS_TGT

make
make DESTDIR=$LFS install
```

### Verify

```bash
ls -l "$LFS/usr/bin/"{gzip,gunzip,zcat}
```

### Executed in this build

The retained transcript confirmed:

```text
/mnt/lfs/usr/bin/gzip
/mnt/lfs/usr/bin/gunzip
/mnt/lfs/usr/bin/zcat
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf gzip-1.14
```

---

## 6.12 Make 4.4.1

### Purpose

Make controls the generation of executables and other non-source files.

Approximate build time:

```text
less than 0.1 SBU
```

Required disk space:

```text
15 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf make-4.4.1
tar -xf make-4.4.1.tar.gz
cd make-4.4.1
```

### Configure, build, and install

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(build-aux/config.guess)

make
make DESTDIR=$LFS install
```

### Verify

```bash
ls -l "$LFS/usr/bin/make"
"$LFS/usr/bin/make" --version | head -n 1
```

### Executed in this build

The retained transcript confirmed:

```text
GNU Make 4.4.1
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf make-4.4.1
```

---

## 6.13 Patch 2.8

### Purpose

Patch applies modifications generated by tools such as `diff`.

Approximate build time:

```text
0.1 SBU
```

Required disk space:

```text
14 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf patch-2.8
tar -xf patch-2.8.tar.xz
cd patch-2.8
```

### Configure, build, and install

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(build-aux/config.guess)

make
make DESTDIR=$LFS install
```

### Verify

```bash
ls -l "$LFS/usr/bin/patch"
"$LFS/usr/bin/patch" --version | head -n 1
```

### Executed in this build

The retained transcript confirmed:

```text
GNU patch 2.8
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf patch-2.8
```

---

## 6.14 Sed 4.9

### Purpose

Sed is a stream editor.

Approximate build time:

```text
0.1 SBU
```

Required disk space:

```text
21 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf sed-4.9
tar -xf sed-4.9.tar.xz
cd sed-4.9
```

### Configure, build, and install

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(./build-aux/config.guess)

make
make DESTDIR=$LFS install
```

### Verify

```bash
ls -l "$LFS/usr/bin/sed"
"$LFS/usr/bin/sed" --version | head -n 1
```

### Executed in this build

The retained transcript confirmed:

```text
/mnt/lfs/usr/bin/sed (GNU sed) 4.9
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf sed-4.9
```

---

## 6.15 Tar 1.35

### Purpose

Tar creates, extracts, updates, and lists archive files.

Approximate build time:

```text
0.1 SBU
```

Required disk space:

```text
42 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf tar-1.35
tar -xf tar-1.35.tar.xz
cd tar-1.35
```

### Configure, build, and install

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(build-aux/config.guess)

make
make DESTDIR=$LFS install
```

### Verify

```bash
ls -l "$LFS/usr/bin/tar"
"$LFS/usr/bin/tar" --version | head -n 1
```

### Executed in this build

The retained transcript confirmed:

```text
tar (GNU tar) 1.35
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf tar-1.35
```

---

## 6.16 Xz 5.8.2

### Purpose

Xz provides compression and decompression for:

```text
.xz
.lzma
```

Approximate build time:

```text
0.1 SBU
```

Required disk space:

```text
24 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf xz-5.8.2
tar -xf xz-5.8.2.tar.xz
cd xz-5.8.2
```

### Configure, build, and install

```bash
./configure --prefix=/usr \
  --host=$LFS_TGT \
  --build=$(build-aux/config.guess) \
  --disable-static \
  --docdir=/usr/share/doc/xz-5.8.2

make
make DESTDIR=$LFS install
```

### Remove the harmful libtool archive

```bash
rm -v "$LFS/usr/lib/liblzma.la"
```

### Verify

```bash
"$LFS/usr/bin/xz" --version | head -n 1

test ! -e "$LFS/usr/lib/liblzma.la" \
  && echo "OK: liblzma.la removed"
```

### Executed in this build

The retained transcript confirmed:

```text
xz (XZ Utils) 5.8.2
liblzma.la removed
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf xz-5.8.2
```

---

## 6.17 Binutils 2.46.0 — Pass 2

### Purpose

This second Binutils pass builds target-aware temporary tools for the future `chroot` environment.

Approximate build time:

```text
0.4 SBU
```

Required disk space:

```text
557 MB
```

### Extract

```bash
cd "$LFS/sources"

rm -rf binutils-2.46.0
tar -xf binutils-2.46.0.tar.xz
cd binutils-2.46.0
```

### Apply the libtool workaround

```bash
sed '6031s/$add_dir//' -i ltmain.sh
```

### Why this workaround is required

The Binutils build system ships a libtool copy that may link mistakenly against host libraries.

This edit prevents the incorrect host-library path injection.

### Executed in this build

The retained transcript confirmed execution of:

```bash
sed '6031s/$add_dir//' -i ltmain.sh
```

### Create a dedicated build directory

```bash
mkdir -v build
cd build
```

### Configure Binutils Pass 2

```bash
../configure \
  --prefix=/usr \
  --build=$(../config.guess) \
  --host=$LFS_TGT \
  --disable-nls \
  --enable-shared \
  --enable-gprofng=no \
  --disable-werror \
  --enable-64-bit-bfd \
  --enable-new-dtags \
  --enable-default-hash-style=gnu
```

### Build and install

```bash
make
make DESTDIR=$LFS install
```

### Remove unnecessary static and libtool archive files

```bash
rm -v "$LFS/usr/lib/"lib{bfd,ctf,ctf-nobfd,opcodes,sframe}.{a,la}
```

### Executed in this build

The retained transcript confirmed removal of:

```text
libbfd.a
libbfd.la
libctf.a
libctf.la
libctf-nobfd.a
libctf-nobfd.la
libopcodes.a
libopcodes.la
libsframe.a
libsframe.la
```

### Verify

```bash
find "$LFS/usr/lib" \
  \( -name 'libbfd.*' \
  -o -name 'libctf.*' \
  -o -name 'libctf-nobfd.*' \
  -o -name 'libopcodes.*' \
  -o -name 'libsframe.*' \) \
  -print
```

Review the result manually.

The removed `.a` and `.la` files must not remain.

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf binutils-2.46.0
```

---

## 6.18 GCC 15.2.0 — Pass 2

### Purpose

This second GCC pass builds the temporary C and C++ compilers and target libraries needed inside `chroot`.

Approximate build time:

```text
4.5 SBU
```

Required disk space:

```text
6.0 GB
```

### Extract GCC

```bash
cd "$LFS/sources"

rm -rf gcc-15.2.0
tar -xf gcc-15.2.0.tar.xz
cd gcc-15.2.0
```

### Extract internal math dependencies

```bash
tar -xf ../mpfr-4.2.2.tar.xz
mv -v mpfr-4.2.2 mpfr

tar -xf ../gmp-6.3.0.tar.xz
mv -v gmp-6.3.0 gmp

tar -xf ../mpc-1.3.1.tar.gz
mv -v mpc-1.3.1 mpc
```

### Executed in this build

The retained transcript confirmed the directories:

```text
gmp
mpc
mpfr
```

inside:

```text
/mnt/lfs/sources/gcc-15.2.0
```

### Correct the x86_64 library directory name

```bash
case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' \
      -i.orig gcc/config/i386/t-linux64
  ;;
esac
```

### Executed in this build

The retained transcript confirmed:

```text
gcc/config/i386/t-linux64
gcc/config/i386/t-linux64.orig
```

### Enable POSIX threads for libgcc and Libstdc++ headers

```bash
sed '/thread_header =/s/@.*@/gthr-posix.h/' \
  -i libgcc/Makefile.in libstdc++-v3/include/Makefile.in
```

### Verify

```bash
grep 'thread_header =' \
  libgcc/Makefile.in \
  libstdc++-v3/include/Makefile.in
```

Expected:

```text
libgcc/Makefile.in:thread_header = gthr-posix.h
libstdc++-v3/include/Makefile.in:thread_header = gthr-posix.h
```

### Executed in this build

The retained transcript confirmed exactly these two values.

### Create a dedicated build directory

```bash
mkdir -v build
cd build
```

### Remove environment flags that could alter the build

```bash
unset {C,CPP,CXX,LD}FLAGS
```

### Executed in this build

The retained transcript confirmed execution of:

```bash
unset {C,CPP,CXX,LD}FLAGS
```

inside:

```text
/mnt/lfs/sources/gcc-15.2.0/build
```

### Configure GCC Pass 2

```bash
../configure \
  --build=$(../config.guess) \
  --host=$LFS_TGT \
  --target=$LFS_TGT \
  --prefix=/usr \
  --with-build-sysroot=$LFS \
  --enable-default-pie \
  --enable-default-ssp \
  --disable-nls \
  --disable-multilib \
  --disable-libatomic \
  --disable-libgomp \
  --disable-libquadmath \
  --disable-libsanitizer \
  --disable-libssp \
  --disable-libvtv \
  --enable-languages=c,c++ \
  LDFLAGS_FOR_TARGET=-L$PWD/$LFS_TGT/libgcc
```

### Meaning of important options

| Option | Purpose |
|---|---|
| `--with-build-sysroot=$LFS` | Makes additional GCC build tools find headers and libraries inside `/mnt/lfs` |
| `--target=$LFS_TGT` | Ensures target libraries use GCC Pass 1 rather than unsupported host compilers |
| `LDFLAGS_FOR_TARGET=...` | Lets Libstdc++ use the libgcc built in this pass |
| `--disable-libsanitizer` | Skips sanitizer runtime libraries not required in this temporary installation |

### Build and install

```bash
make
make DESTDIR=$LFS install
```

### Create the generic `cc` compiler link

```bash
ln -sv gcc "$LFS/usr/bin/cc"
```

### Verify

```bash
ls -l "$LFS/usr/bin/cc"
```

Expected:

```text
/mnt/lfs/usr/bin/cc -> gcc
```

### Executed in this build

The retained transcript confirmed:

```text
'/mnt/lfs/usr/bin/cc' -> 'gcc'
```

and:

```text
/mnt/lfs/usr/bin/cc -> gcc
```

### Clean extracted source

```bash
cd "$LFS/sources"
rm -rf gcc-15.2.0
```

---

## 6.19 End-of-chapter verification

Run as:

```text
lfs
```

Verify the build identity and environment:

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

Verify key temporary tools:

```bash
ls -l "$LFS/usr/bin/"{
m4,bash,find,xargs,locate,updatedb,gawk,awk,grep,egrep,fgrep,gzip,gunzip,zcat,make,patch,sed,tar,xz,cc,gcc
}
```

For easier copy-paste, use one line:

```bash
ls -l "$LFS/usr/bin/"{m4,bash,find,xargs,locate,updatedb,gawk,awk,grep,egrep,fgrep,gzip,gunzip,zcat,make,patch,sed,tar,xz,cc,gcc}
```

Verify `sh`:

```bash
ls -l "$LFS/bin/sh"
```

Verify key libraries and compatibility links:

```bash
ls -l "$LFS/usr/lib/libncurses.so"
test ! -e "$LFS/usr/lib/libmagic.la" && echo "OK: no libmagic.la"
test ! -e "$LFS/usr/lib/liblzma.la" && echo "OK: no liblzma.la"
```

Verify `chroot` location:

```bash
ls -l "$LFS/usr/sbin/chroot"
```

Verify `cc`:

```bash
ls -l "$LFS/usr/bin/cc"
```

Expected:

```text
/mnt/lfs/usr/bin/cc -> gcc
```

Verify that `/usr/lib64` was not created:

```bash
test ! -e "$LFS/usr/lib64" \
  && echo "OK: no /usr/lib64"
```

---

## 6.20 Full package order

Build packages in this exact order:

```text
1.  M4 1.4.21
2.  Ncurses 6.6
3.  Bash 5.3
4.  Coreutils 9.10
5.  Diffutils 3.12
6.  File 5.46
7.  Findutils 4.10.0
8.  Gawk 5.3.2
9.  Grep 3.12
10. Gzip 1.14
11. Make 4.4.1
12. Patch 2.8
13. Sed 4.9
14. Tar 1.35
15. Xz 5.8.2
16. Binutils 2.46.0 — Pass 2
17. GCC 15.2.0 — Pass 2
```

Do not reorder packages.

The sequence resolves temporary-tool dependencies before entering `chroot`.

---

## 6.21 Errors and lessons learned

### 1. Temporary tools are installed but not executed yet

**Potential confusion**

The tools appear under:

```text
/mnt/lfs/usr/bin
```

but Chapter 6 still relies on Debian host tools.

**Reason**

The current shell is outside `chroot`.

**Resolution**

Do not try to use the LFS temporary binaries directly unless the book explicitly instructs you to do so.

They become operational after Chapter 7 enters `chroot`.

---

### 2. Clean extracted directories after each successful package

Keep the archive files:

```text
*.tar.xz
*.tar.gz
```

Delete only extracted working trees:

```bash
rm -rf <package-source-directory>
```

This avoids stale source and build artifacts affecting later passes.

---

### 3. Remove `.la` files exactly where the book requires it

The build intentionally removed:

```text
/mnt/lfs/usr/lib/libmagic.la
/mnt/lfs/usr/lib/liblzma.la
```

Binutils Pass 2 also removed several static and `.la` files.

These files are harmful during cross-compilation because they may preserve incorrect library paths.

---

### 4. GCC Pass 2 requires POSIX thread header overrides

Do not skip:

```bash
sed '/thread_header =/s/@.*@/gthr-posix.h/' \
  -i libgcc/Makefile.in libstdc++-v3/include/Makefile.in
```

Verify both values before configuring GCC.

---

### 5. Unset optimization flags before GCC Pass 2

Do not tune GCC Pass 2 with inherited flags.

Run:

```bash
unset {C,CPP,CXX,LD}FLAGS
```

---

## 6.22 GitHub evidence capture

Create the evidence directory:

```bash
REPO_ROOT=~/Projects/LFS-13.0-Systemd-Build-Notes

mkdir -pv "$REPO_ROOT/logs/chapter-06"
```

Capture environment:

```bash
{
  echo "whoami=$(whoami)"
  echo "LFS=$LFS"
  echo "LFS_TGT=$LFS_TGT"
  echo "umask=$(umask)"
  echo "PATH=$PATH"
  set -o | grep hashall
} | tee "$REPO_ROOT/logs/chapter-06/environment.txt"
```

Capture key temporary tools:

```bash
ls -l "$LFS/usr/bin/"{m4,bash,find,xargs,locate,updatedb,gawk,awk,grep,egrep,fgrep,gzip,gunzip,zcat,make,patch,sed,tar,xz,cc,gcc} \
  | tee "$REPO_ROOT/logs/chapter-06/key-temporary-tools.txt"
```

Capture links and cleanup checks:

```bash
{
  ls -l "$LFS/bin/sh"
  ls -l "$LFS/usr/bin/cc"
  ls -l "$LFS/usr/lib/libncurses.so"
  ls -l "$LFS/usr/sbin/chroot"

  test ! -e "$LFS/usr/lib/libmagic.la" \
    && echo "OK: no libmagic.la"

  test ! -e "$LFS/usr/lib/liblzma.la" \
    && echo "OK: no liblzma.la"

  test ! -e "$LFS/usr/lib64" \
    && echo "OK: no /usr/lib64"
} | tee "$REPO_ROOT/logs/chapter-06/links-and-cleanup-checks.txt"
```

Capture selected tool versions:

```bash
{
  "$LFS/usr/bin/make" --version | head -n 1
  "$LFS/usr/bin/patch" --version | head -n 1
  "$LFS/usr/bin/sed" --version | head -n 1
  "$LFS/usr/bin/tar" --version | head -n 1
  "$LFS/usr/bin/xz" --version | head -n 1
} | tee "$REPO_ROOT/logs/chapter-06/selected-tool-versions.txt"
```

---

## Chapter 6 approval checklist

Before moving to Chapter 7 documentation:

```text
[ ] Review all commands in this file
[ ] Confirm all Chapter 6 work runs as lfs, not root
[ ] Confirm exact package order
[ ] Confirm host tic build for Ncurses
[ ] Confirm /mnt/lfs/bin/sh -> bash
[ ] Confirm chroot moved to /mnt/lfs/usr/sbin
[ ] Confirm Findutils binaries
[ ] Confirm awk -> gawk
[ ] Confirm Grep binaries
[ ] Confirm Gzip binaries
[ ] Confirm Make 4.4.1
[ ] Confirm Patch 2.8
[ ] Confirm Sed 4.9
[ ] Confirm Tar 1.35
[ ] Confirm Xz 5.8.2
[ ] Confirm liblzma.la removed
[ ] Confirm Binutils Pass 2 cleanup
[ ] Confirm GCC Pass 2 thread header override
[ ] Confirm GCC Pass 2 unset flags step
[ ] Confirm /mnt/lfs/usr/bin/cc -> gcc
[ ] Confirm /mnt/lfs/usr/lib64 does not exist
[ ] Capture logs/chapter-06 evidence
[ ] Do not commit or push until explicit approval
```

---

## Suggested repository placement

Place this reviewed file at:

```text
docs/chapter-06-cross-compiling-temporary-tools.md
```

Suggested companion evidence files:

```text
logs/chapter-06/environment.txt
logs/chapter-06/key-temporary-tools.txt
logs/chapter-06/links-and-cleanup-checks.txt
logs/chapter-06/selected-tool-versions.txt
```

---

## Current approval state

```text
DRAFT — CHAPTER 6 ONLY
DO NOT COMMIT
DO NOT PUSH
WAITING FOR REVIEW AND EVIDENCE CAPTURE
```
