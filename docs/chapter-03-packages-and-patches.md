# Chapter 3 — Packages and Patches

> Repository: `github.com/Abeerithm/LFS-13.0-Systemd-Build-Notes`  
> Local repository path: `~/Projects/LFS-13.0-Systemd-Build-Notes`  
> Review branch: `docs-audit-lfs13`  
> Status: **Draft for review — do not commit or push yet**  
> Book source: *Linux From Scratch — Version 13.0-systemd*, Chapter 3  
> Actual LFS mount point used in this build: `/mnt/lfs`

---

## 3.1 Objective

This chapter prepares the complete source archive and patch set required to build the LFS system.

The required outcome is:

```text
/mnt/lfs/sources
```

containing:

1. all package release tarballs required by LFS 13.0-systemd;
2. all required patches;
3. the pinned `wget-list-systemd` manifest;
4. the pinned `md5sums` integrity manifest;
5. permissions that allow controlled shared use during the build;
6. ownership suitable for the later `chroot` environment.

The source directory is used throughout the build as both:

```text
download archive storage
working directory for extracted package sources
```

---

## Documentation convention

Every section is separated into:

### Official book command
Commands or requirements taken from LFS 13.0-systemd.

### Executed in this build
Commands and observations confirmed from the actual build history.

### Environment-specific note
Operational adjustments used in this Debian VirtualBox environment.

---

## 3.1.1 Use the exact LFS package versions

### Official book guidance

Use the release tarballs and versions specified by LFS 13.0-systemd.

Do not replace a release tarball with a Git or SVN snapshot even if the names appear similar. Release tarballs may include generated files required for a successful build, such as `configure`.

Before downloading, review security advisories. If an upstream source removes an old vulnerable release, do not blindly restore it from a mirror. Use the fixed version only when an LFS erratum or security advisory instructs you to do so.

### Environment-specific note

This repository pins the build to:

```text
LFS 13.0-systemd
```

Do not silently substitute package versions during a rebuild.

Any deliberate version replacement must be documented explicitly in:

```text
docs/troubleshooting/
```

with the reason, source, and validation result.

---

## 3.1.2 Confirm the LFS environment variable

Run these commands on the Debian host:

```bash
export LFS=/mnt/lfs
umask 022
```

Verify:

```bash
echo "$LFS"
umask
```

Expected output:

```text
/mnt/lfs
0022
```

### Executed in this build

The active project path was confirmed as:

```text
/mnt/lfs
```

The same value was later verified again before entering `chroot`.

---

## 3.1.3 Create the sources directory

### Official book commands

Run as `root`:

```bash
mkdir -v $LFS/sources
chmod -v a+wt $LFS/sources
```

### Equivalent host-side commands used in this environment

When working from a normal Debian account, use `sudo`:

```bash
sudo mkdir -pv "$LFS/sources"
sudo chmod -v a+wt "$LFS/sources"
```

Verify:

```bash
ls -ld "$LFS/sources"
```

Expected permission pattern:

```text
drwxrwxrwt
```

The final `t` is the sticky bit.

### Why the sticky bit matters

The source directory is writable by multiple users during the build, but the sticky bit prevents one user from deleting files owned by another user.

This is the same protection model commonly used for:

```text
/tmp
```

---

## 3.1.4 Preserve the pinned download manifests inside the repository

To make this repository reproducible without consulting the book again, store the exact LFS 13.0-systemd manifest files in the repository:

```text
manifests/chapter-03/wget-list-systemd
manifests/chapter-03/md5sums
```

### First-time capture from the completed build

Run after the files exist in `/mnt/lfs/sources`:

```bash
REPO_ROOT=~/Projects/LFS-13.0-Systemd-Build-Notes

mkdir -pv "$REPO_ROOT/manifests/chapter-03"

cp -v "$LFS/sources/wget-list-systemd" \
      "$REPO_ROOT/manifests/chapter-03/"

cp -v "$LFS/sources/md5sums" \
      "$REPO_ROOT/manifests/chapter-03/"
```

### Rebuild procedure using repository copies

For a clean rebuild:

```bash
REPO_ROOT=~/Projects/LFS-13.0-Systemd-Build-Notes

cp -v "$REPO_ROOT/manifests/chapter-03/wget-list-systemd" \
      "$LFS/sources/"

cp -v "$REPO_ROOT/manifests/chapter-03/md5sums" \
      "$LFS/sources/"
```

### Audit note

The manifest files must be copied into the repository before Chapter 3 is marked final.

Until then:

```text
CHAPTER 3 IS REPRODUCIBLE IN PROCEDURE
BUT THE PINNED MANIFEST ARTIFACTS STILL NEED TO BE ADDED
```

---

## 3.1.5 Download all packages and patches

### Official book command

Run from the directory containing `wget-list-systemd`:

```bash
wget --input-file=wget-list-systemd \
     --continue \
     --directory-prefix=$LFS/sources
```

### Repository-driven rebuild command

This version does not depend on the current shell directory:

```bash
wget --input-file="$LFS/sources/wget-list-systemd" \
     --continue \
     --directory-prefix="$LFS/sources"
```

### Meaning of the options

| Option | Purpose |
|---|---|
| `--input-file=...` | Reads all package and patch URLs from the pinned manifest |
| `--continue` | Resumes partially downloaded files instead of restarting them |
| `--directory-prefix=...` | Saves all downloaded files under `/mnt/lfs/sources` |

### Executed in this build

The project downloaded the LFS source set into:

```text
/mnt/lfs/sources
```

A captured download screen showed:

```text
linux-6.18.10.tar.xz
```

in progress at approximately 50%.

The later build successfully extracted and compiled packages from the same directory, confirming that the source archive location remained consistent throughout Chapters 5–8.

### Environment-specific note

Do not interrupt an active download unnecessarily.

Because `wget` uses:

```text
--continue
```

a stopped download can be resumed safely with the same command.

---

## 3.1.6 Verify downloaded files with MD5 checksums

### Official book commands

Place `md5sums` inside:

```text
$LFS/sources
```

Then run:

```bash
pushd $LFS/sources
  md5sum -c md5sums
popd
```

### Executed build workflow

After the source download completed, the verification command used for this environment was:

```bash
md5sum -c "$LFS/sources/md5sums"
```

For an auditable rebuild, preserve the complete output:

```bash
mkdir -pv "$LFS/sources/logs"

pushd "$LFS/sources"
  md5sum -c md5sums | tee logs/md5sum-check-output.txt
popd
```

### Validate that no checksum failed

Run:

```bash
grep -Ev ': OK$' "$LFS/sources/logs/md5sum-check-output.txt"
```

Expected result:

```text
no output
```

If any file reports:

```text
FAILED
```

delete that archive and download it again before continuing.

Example pattern:

```bash
rm -v "$LFS/sources/<failed-file>"
wget --input-file="$LFS/sources/wget-list-systemd" \
     --continue \
     --directory-prefix="$LFS/sources"
```

Then rerun:

```bash
pushd "$LFS/sources"
  md5sum -c md5sums | tee logs/md5sum-check-output.txt
popd
```

### Audit note

The original complete `md5sum -c` output is not preserved in the retained chat transcript.

Before Chapter 3 is marked final, rerun the checksum validation and add:

```text
logs/chapter-03/md5sum-check-output.txt
```

to the repository.

---

## 3.1.7 Fix ownership before later `chroot` work

### Official book command

If packages were downloaded as a non-root user:

```bash
chown root:root $LFS/sources/*
```

### Executed in this build

The host-side command used in this Debian environment was:

```bash
sudo chown -R root:root "$LFS/sources"
```

The sticky permission was then restored explicitly:

```bash
sudo chmod -v a+wt "$LFS/sources"
```

### Why ownership matters

A normal host user UID may not exist inside the final LFS system.

Without correcting ownership, files may later appear to belong to an unnamed numeric UID inside `chroot`.

### Difference between official and executed commands

| Form | Scope |
|---|---|
| `chown root:root $LFS/sources/*` | Official minimum: top-level downloaded files |
| `sudo chown -R root:root "$LFS/sources"` | Executed environment-specific form: recursively normalizes the entire source tree |

The recursive form is safe before active package extraction begins. During later chapters, avoid recursively changing ownership without understanding the current build state.

### Verify ownership and permissions

```bash
ls -ld "$LFS/sources"
find "$LFS/sources" -maxdepth 1 -type f -printf '%u:%g %f\n' | head -n 20
```

Expected:

```text
directory permissions include sticky bit: drwxrwxrwt
top-level package files owned by: root:root
```

---

## 3.2 Package manifest

The complete package URL list must be stored in:

```text
manifests/chapter-03/wget-list-systemd
```

The complete checksum list must be stored in:

```text
manifests/chapter-03/md5sums
```

The book states that the total package download size is approximately:

```text
603 MB
```

### Archives directly confirmed during this build

The following archives were directly observed or consumed during the build documented in this repository:

```text
binutils-2.46.0.tar.xz
gcc-15.2.0.tar.xz
linux-6.18.10.tar.xz
glibc-2.43.tar.xz
mpfr-4.2.2.tar.xz
gmp-6.3.0.tar.xz
mpc-1.3.1.tar.gz
m4-1.4.21.tar.xz
ncurses-6.6.tar.gz
bash-5.3.tar.gz
coreutils-9.10.tar.xz
diffutils-3.12.tar.xz
file-5.46.tar.gz
findutils-4.10.0.tar.xz
gawk-5.3.2.tar.xz
grep-3.12.tar.xz
gzip-1.14.tar.xz
make-4.4.1.tar.gz
patch-2.8.tar.xz
sed-4.9.tar.xz
tar-1.35.tar.xz
xz-5.8.2.tar.xz
bison-3.8.2.tar.xz
perl-5.42.0.tar.xz
Python-3.14.3.tar.xz
texinfo-7.2.tar.xz
util-linux-2.41.3.tar.xz
man-pages-6.17.tar.xz
iana-etc-20260202.tar.gz
zlib-1.3.2.tar.gz
bzip2-1.0.8.tar.gz
lz4-1.10.0.tar.gz
zstd-1.5.7.tar.gz
readline-8.3.tar.gz
pcre2-10.47.tar.bz2
bc-7.0.3.tar.xz
flex-2.6.4.tar.gz
tcl8.6.17-src.tar.gz
tzdata2025c.tar.gz
```

### Important audit rule

The list above is not a substitute for:

```text
manifests/chapter-03/wget-list-systemd
```

It records the archives already encountered in the actual build transcript.

The pinned manifest is the authoritative full inventory for a complete rebuild.

---

## 3.3 Required patches

LFS 13.0-systemd requires the following patches.

| Patch | Purpose | Download URL | MD5 |
|---|---|---|---|
| `bzip2-1.0.8-install_docs-1.patch` | Installs Bzip2 documentation correctly | `https://www.linuxfromscratch.org/patches/lfs/13.0/bzip2-1.0.8-install_docs-1.patch` | `6a5ac7e89b791aae556de0f745916f7f` |
| `coreutils-9.10-i18n-1.patch` | Internationalization fixes for Coreutils | `https://www.linuxfromscratch.org/patches/lfs/13.0/coreutils-9.10-i18n-1.patch` | `6e9aea31b1662176101e6438a39fdad4` |
| `expect-5.45.4-gcc15-1.patch` | GCC 15 compatibility fixes for Expect | `https://www.linuxfromscratch.org/patches/lfs/13.0/expect-5.45.4-gcc15-1.patch` | `0ca4d6bb8d572fbcdb13cb36cd34833e` |
| `glibc-fhs-1.patch` | Aligns Glibc paths with FHS expectations | `https://www.linuxfromscratch.org/patches/lfs/13.0/glibc-fhs-1.patch` | `9a5997c3452909b1769918c759eff8a2` |
| `kbd-2.9.0-backspace-1.patch` | Backspace/Delete behavior correction for Kbd | `https://www.linuxfromscratch.org/patches/lfs/13.0/kbd-2.9.0-backspace-1.patch` | `f75cca16a38da6caa7d52151f7136895` |

The book states that the total required patch size is approximately:

```text
156.4 KB
```

### Patches already used successfully in this build

The transcript confirms successful application of:

```text
glibc-fhs-1.patch
bzip2-1.0.8-install_docs-1.patch
```

The remaining required patches must stay in `/mnt/lfs/sources` until their packages are reached later in Chapter 8.

### Verify patch files exist

```bash
ls -l "$LFS/sources"/*.patch
```

### Verify required patch names explicitly

```bash
for patch in \
  bzip2-1.0.8-install_docs-1.patch \
  coreutils-9.10-i18n-1.patch \
  expect-5.45.4-gcc15-1.patch \
  glibc-fhs-1.patch \
  kbd-2.9.0-backspace-1.patch
do
  test -f "$LFS/sources/$patch" \
    && echo "OK: $patch" \
    || echo "MISSING: $patch"
done
```

Expected:

```text
OK: bzip2-1.0.8-install_docs-1.patch
OK: coreutils-9.10-i18n-1.patch
OK: expect-5.45.4-gcc15-1.patch
OK: glibc-fhs-1.patch
OK: kbd-2.9.0-backspace-1.patch
```

---

## 3.4 Repository audit logs

Create an evidence directory:

```bash
REPO_ROOT=~/Projects/LFS-13.0-Systemd-Build-Notes

mkdir -pv "$REPO_ROOT/logs/chapter-03"
```

Capture the directory permission:

```bash
ls -ld "$LFS/sources" \
  | tee "$REPO_ROOT/logs/chapter-03/sources-directory-permissions.txt"
```

Capture the source inventory:

```bash
find "$LFS/sources" -maxdepth 1 -type f -printf '%f\n' \
  | sort \
  | tee "$REPO_ROOT/logs/chapter-03/source-inventory.txt"
```

Capture checksum verification:

```bash
pushd "$LFS/sources"
  md5sum -c md5sums \
    | tee "$REPO_ROOT/logs/chapter-03/md5sum-check-output.txt"
popd
```

Capture patch verification:

```bash
for patch in \
  bzip2-1.0.8-install_docs-1.patch \
  coreutils-9.10-i18n-1.patch \
  expect-5.45.4-gcc15-1.patch \
  glibc-fhs-1.patch \
  kbd-2.9.0-backspace-1.patch
do
  test -f "$LFS/sources/$patch" \
    && echo "OK: $patch" \
    || echo "MISSING: $patch"
done | tee "$REPO_ROOT/logs/chapter-03/required-patches-check.txt"
```

Copy the pinned manifests:

```bash
mkdir -pv "$REPO_ROOT/manifests/chapter-03"

cp -v "$LFS/sources/wget-list-systemd" \
      "$REPO_ROOT/manifests/chapter-03/"

cp -v "$LFS/sources/md5sums" \
      "$REPO_ROOT/manifests/chapter-03/"
```

---

## 3.5 Full clean-rebuild procedure

The following block is the complete Chapter 3 rebuild sequence.

Run on the Debian host after Chapter 2 is complete:

```bash
export LFS=/mnt/lfs
umask 022

echo "$LFS"
umask

sudo mkdir -pv "$LFS/sources"
sudo chmod -v a+wt "$LFS/sources"

REPO_ROOT=~/Projects/LFS-13.0-Systemd-Build-Notes

cp -v "$REPO_ROOT/manifests/chapter-03/wget-list-systemd" \
      "$LFS/sources/"

cp -v "$REPO_ROOT/manifests/chapter-03/md5sums" \
      "$LFS/sources/"

wget --input-file="$LFS/sources/wget-list-systemd" \
     --continue \
     --directory-prefix="$LFS/sources"

pushd "$LFS/sources"
  md5sum -c md5sums | tee md5sum-check-output.txt
popd

grep -Ev ': OK$' "$LFS/sources/md5sum-check-output.txt"

sudo chown -R root:root "$LFS/sources"
sudo chmod -v a+wt "$LFS/sources"

ls -ld "$LFS/sources"

for patch in \
  bzip2-1.0.8-install_docs-1.patch \
  coreutils-9.10-i18n-1.patch \
  expect-5.45.4-gcc15-1.patch \
  glibc-fhs-1.patch \
  kbd-2.9.0-backspace-1.patch
do
  test -f "$LFS/sources/$patch" \
    && echo "OK: $patch" \
    || echo "MISSING: $patch"
done
```

### Required stopping condition

Do not continue to Chapter 4 until:

```text
[ ] all MD5 checks report OK
[ ] no archive reports FAILED
[ ] all five required patches report OK
[ ] /mnt/lfs/sources has sticky-bit permissions
[ ] package files are owned by root:root
[ ] pinned manifests have been copied into the repository
```

---

## Errors and lessons learned

### 1. Interrupted downloads are recoverable

**Risk**

Large archives such as:

```text
linux-6.18.10.tar.xz
```

may take time to download inside a VM.

**Solution**

Use:

```bash
--continue
```

with `wget`.

This resumes partial downloads instead of restarting them.

---

### 2. Ownership must be normalized before `chroot`

**Risk**

Files downloaded by a regular Debian user may later appear under an unnamed numeric UID inside LFS.

**Solution used in this build**

```bash
sudo chown -R root:root "$LFS/sources"
sudo chmod -v a+wt "$LFS/sources"
```

---

### 3. Do not use package snapshots as replacements for release tarballs

**Risk**

Repository snapshots may omit generated files such as:

```text
configure
```

**Solution**

Use the exact release archives pinned by:

```text
manifests/chapter-03/wget-list-systemd
```

---

## Chapter 3 approval checklist

Before this file is marked final:

```text
[ ] Copy the actual wget-list-systemd into manifests/chapter-03/
[ ] Copy the actual md5sums into manifests/chapter-03/
[ ] Capture a fresh full md5sum verification log
[ ] Capture the exact source inventory
[ ] Capture /mnt/lfs/sources permissions
[ ] Capture required patch verification output
[ ] Review the file
[ ] Do not commit or push until explicit approval
```

---

## Suggested repository placement

Place this reviewed file at:

```text
docs/chapter-03-packages-and-patches.md
```

Suggested companion files:

```text
manifests/chapter-03/wget-list-systemd
manifests/chapter-03/md5sums
logs/chapter-03/sources-directory-permissions.txt
logs/chapter-03/source-inventory.txt
logs/chapter-03/md5sum-check-output.txt
logs/chapter-03/required-patches-check.txt
```

---

## Current approval state

```text
DRAFT — CHAPTER 3 ONLY
DO NOT COMMIT
DO NOT PUSH
WAITING FOR REVIEW AND MANIFEST CAPTURE
```
