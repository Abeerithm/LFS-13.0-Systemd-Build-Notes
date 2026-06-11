# Stage 02 — Packages and Patches

## Goal

Prepare the source packages and patches required to build Linux From Scratch 13.0 Systemd.

## Sources Directory

All downloaded source archives and patches were stored in:

```text
/mnt/lfs/sources
```

The directory was prepared earlier with the sticky bit enabled:

```bash
mkdir -v $LFS/sources
chmod -v a+wt $LFS/sources
```

## Download Strategy

The required packages and patches were downloaded according to the official LFS 13.0 Systemd book.

The official book should always remain the primary reference for:

* Package versions
* Patch versions
* Download URLs
* Checksums
* Required build order

## Verify the Sources Directory

Check the contents:

```bash
ls -lh $LFS/sources
```

Count the files:

```bash
find $LFS/sources -maxdepth 1 -type f | wc -l
```

## Verify Downloads

Checksums should be verified before starting the build.

Example workflow:

```bash
cd $LFS/sources
md5sum -c md5sums
```

Expected result:

```text
OK
```

for each verified file.

## Important Notes

* Keep all original source archives inside `/mnt/lfs/sources`.
* Extract packages only when needed.
* Remove extracted build directories after completing each package unless the book explicitly says otherwise.
* Do not remove the original archives.
* Avoid modifying downloaded source archives directly.
* Apply patches only when required by the official book.

## Common Archive Types

Typical package formats include:

```text
.tar.xz
.tar.gz
.tar.bz2
.patch
```

## Recommended Validation

Before continuing to the build stages, confirm:

```bash
echo $LFS
ls -ld $LFS/sources
ls -lh $LFS/sources | head
```

Expected main path:

```text
/mnt/lfs
```

## Personal Build Notes

The source directory was preserved as the central package archive throughout the project.

This made it possible to continue the build without re-downloading packages and also supported snapshot-based recovery inside VirtualBox.

## Reminder

This repository documents a practical build journey. Always compare package versions, checksums, and patch requirements with the official LFS 13.0 Systemd book before reuse.
