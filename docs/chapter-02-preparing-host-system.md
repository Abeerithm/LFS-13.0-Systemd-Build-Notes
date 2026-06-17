# Chapter 2 — Preparing the Host System

> **Repository:** `github.com/Abeerithm/LFS-13.0-Systemd-Build-Notes`  
> **Local repository path:** `~/Projects/LFS-13.0-Systemd-Build-Notes`  
> **Review branch:** `docs-audit-lfs13`  
> **Documentation status:** Reviewed and approved for GitHub upload  
> **Book baseline:** *Linux From Scratch — Version 13.0-systemd*, Chapter 2  
> **Actual mount point used in this build:** `/mnt/lfs`

---

## 2.1 Purpose of This Chapter

Chapter 2 prepares the host system before any LFS packages are downloaded or compiled.

At the end of this chapter, the host must provide:

1. the required development tools;
2. a working C++ compiler;
3. a supported Linux kernel with UNIX 98 pseudo-terminal support;
4. a dedicated filesystem for the new LFS system;
5. a stable mount point at `/mnt/lfs`;
6. the environment variable `LFS=/mnt/lfs`;
7. a safe default `umask` value of `022`.

This project was built inside a Debian virtual machine running on VirtualBox.

---

## Documentation Convention

This repository distinguishes between three categories:

### Official LFS procedure
Commands and requirements taken from the LFS 13.0-systemd book.

### Executed in this build
Commands or outputs confirmed from the actual build history.

### Environment-specific note
Operational observations, troubleshooting details, or decisions specific to this Debian VirtualBox environment.

> **Important:** The original terminal transcript does not preserve the exact disk device name used when the LFS partition was first created. This file therefore uses variables such as `LFS_DISK` and `LFS_PARTITION` instead of inventing historical values. Before re-executing any destructive command, identify the correct device on the current machine.

---

# 2.2 Host System Requirements

## 2.2.1 Hardware Requirements

### Official LFS procedure

The LFS editors recommend:

```text
CPU: at least 4 cores
RAM: at least 8 GB
```

Older or smaller systems may still work, but package compilation and test execution will take longer.

### Executed in this build

The build was performed in a Debian VirtualBox VM.

A later diagnostic during the final Glibc test suite showed approximately:

```text
RAM total: 3.9 GiB
```

### Environment-specific note

This VM was below the recommended memory size. The build remained possible, but several Glibc tests later timed out while GNOME and Firefox were consuming resources.

The solution that worked was:

1. close unnecessary graphical applications;
2. leave only the terminal open;
3. rerun sensitive tests sequentially with `-j1`;
4. increase the timeout factor when required.

This is documented again in the Chapter 8 troubleshooting notes.

---

## 2.2.2 Required Host Software

### Official LFS procedure

The host should provide at least the following versions:

| Tool | Minimum version | Required condition |
|---|---:|---|
| Bash | 3.2 | `/bin/sh` must invoke Bash |
| Binutils | 2.13.1 | Versions newer than 2.46.0 are not recommended by this book |
| Bison | 2.7 | `/usr/bin/yacc` must invoke Bison |
| Coreutils | 8.1 | Required by the validation script |
| Diffutils | 2.8.1 |  |
| Findutils | 4.2.31 |  |
| Gawk | 4.0.1 | `/usr/bin/awk` must invoke GNU awk |
| GCC | 5.4 | `g++` and standard C/C++ headers are required |
| Grep | 2.5.1a |  |
| Gzip | 1.3.12 |  |
| Linux kernel | 5.4 | UNIX 98 PTY support is also required |
| M4 | 1.4.10 |  |
| Make | 4.0 |  |
| Patch | 2.5.4 |  |
| Perl | 5.8.8 |  |
| Python | 3.4 | `python3` |
| Sed | 4.1.5 |  |
| Tar | 1.22 |  |
| Texinfo | 5.0 | `texi2any` |
| Xz | 5.0.0 |  |

The host kernel must support UNIX 98 pseudo terminals. A custom host kernel must include:

```text
CONFIG_UNIX98_PTYS=y
```

---

## 2.2.3 Create and Run the Host Validation Script

### Official LFS procedure

Create the script exactly as follows:

```bash
cat > version-check.sh << "EOF"
#!/bin/bash
# A script to list version numbers of critical development tools
# If you have tools installed in other directories, adjust PATH here AND
# in ~lfs/.bashrc (section 4.4) as well.
LC_ALL=C
PATH=/usr/bin:/bin
bail() { echo "FATAL: $1"; exit 1; }
grep --version > /dev/null 2> /dev/null || bail "grep does not work"
sed '' /dev/null || bail "sed does not work"
sort /dev/null || bail "sort does not work"
ver_check()
{
 if ! type -p $2 &>/dev/null
 then
 echo "ERROR: Cannot find $2 ($1)"; return 1;
 fi
 v=$($2 --version 2>&1 | grep -E -o '[0-9]+\.[0-9\.]+[a-z]*' | head -n1)
 if printf '%s\n' $3 $v | sort --version-sort --check &>/dev/null
 then
 printf "OK: %-9s %-6s >= $3\n" "$1" "$v"; return 0;
 else
 printf "ERROR: %-9s is TOO OLD ($3 or later required)\n" "$1";
 return 1;
 fi
}
ver_kernel()
{
 kver=$(uname -r | grep -E -o '^[0-9\.]+')
 if printf '%s\n' $1 $kver | sort --version-sort --check &>/dev/null
 then
 printf "OK: Linux Kernel $kver >= $1\n"; return 0;
 else
 printf "ERROR: Linux Kernel ($kver) is TOO OLD ($1 or later required)\n" "$kver";
 return 1;
 fi
}
# Coreutils first because --version-sort needs Coreutils >= 7.0
ver_check Coreutils sort 8.1 || bail "Coreutils too old, stop"
ver_check Bash bash 3.2
ver_check Binutils ld 2.13.1
ver_check Bison bison 2.7
ver_check Diffutils diff 2.8.1
ver_check Findutils find 4.2.31
ver_check Gawk gawk 4.0.1
ver_check GCC gcc 5.4
ver_check "GCC (C++)" g++ 5.4
ver_check Grep grep 2.5.1a
ver_check Gzip gzip 1.3.12
ver_check M4 m4 1.4.10
ver_check Make make 4.0
ver_check Patch patch 2.5.4
ver_check Perl perl 5.8.8
ver_check Python python3 3.4
ver_check Sed sed 4.1.5
ver_check Tar tar 1.22
ver_check Texinfo texi2any 5.0
ver_check Xz xz 5.0.0
ver_kernel 5.4
if mount | grep -q 'devpts on /dev/pts' && [ -e /dev/ptmx ]
then echo "OK: Linux Kernel supports UNIX 98 PTY";
else echo "ERROR: Linux Kernel does NOT support UNIX 98 PTY"; fi
alias_check() {
 if $1 --version 2>&1 | grep -qi $2
 then printf "OK: %-4s is $2\n" "$1";
 else printf "ERROR: %-4s is NOT $2\n" "$1"; fi
}
echo "Aliases:"
alias_check awk GNU
alias_check yacc Bison
alias_check sh Bash
echo "Compiler check:"
if printf "int main(){}" | g++ -x c++ -
then echo "OK: g++ works";
else echo "ERROR: g++ does NOT work"; fi
rm -f a.out
if [ "$(nproc)" = "" ]; then
 echo "ERROR: nproc is not available or it produces empty output"
else
 echo "OK: nproc reports $(nproc) logical cores are available"
fi
EOF
```

Run it:

```bash
bash version-check.sh
```

### Required validation

Do not continue until the output shows `OK` for all required tools, aliases, compiler checks, kernel checks, UNIX 98 PTY support, and `nproc`.

### Executed in this build

The original validator output is not preserved in the retained transcript. The later successful build of the cross-toolchain confirms that the host environment was sufficiently functional, but this does not replace the validator output as historical evidence.

### Rebuild note

For a repeat build, preserve the complete validator output before continuing:

```bash
bash version-check.sh | tee version-check-output.txt
```

---

## 2.2.4 Optional Debian Remediation When the Validator Reports Missing Tools

### Environment-specific note

This subsection is not a substitute for the validation script. Use it only when Debian reports missing development tools.

A practical baseline installation command is:

```bash
sudo apt update
sudo apt install build-essential bison gawk texinfo python3 m4 patch xz-utils
```

Then rerun:

```bash
bash version-check.sh
```

If an alias check still fails, inspect it directly:

```bash
readlink -f /bin/sh
command -v awk
command -v yacc
```

Do not force symlink changes blindly. Correct the failing alias only after confirming the current Debian configuration.

---

# 2.3 Building LFS in Stages

## Official LFS procedure

LFS may be built across multiple sessions, but some setup work must be repeated after rebooting or changing users.

### Chapters 1–4

Commands run on the host system.

For procedures performed as `root` after Section 2.4, ensure that `LFS` is set for the `root` user.

### Chapters 5–6

The LFS filesystem must be mounted:

```bash
mount | grep ' /mnt/lfs '
```

Work must be performed as the dedicated `lfs` user:

```bash
su - lfs
```

If a package installation is uncertain, remove the expanded source directory, extract it again, and repeat that package section from a clean source tree.

### Chapters 7–10

The LFS filesystem must be mounted.

Before entering `chroot`, perform the required preparation as `root` with `LFS` set in the root environment.

The virtual kernel filesystems must also be mounted. Their exact commands belong to Chapter 7 and are documented there.

After entering `chroot`, the `LFS` variable is no longer used because `/mnt/lfs` becomes the new `/` root directory.

## Executed in this build

The actual build progression was:

```text
Debian host
  -> lfs user
  -> root
  -> chroot
```

The root environment was later verified with:

```bash
export LFS=/mnt/lfs
umask 022

echo $LFS
umask
```

Recorded output:

```text
/mnt/lfs
0022
```

## Environment-specific note

VirtualBox snapshots were used as checkpoints.

Useful snapshot points:

```text
Chapter 6 complete
Temporary system ready before Chapter 8
Before major Glibc operations
```

---

# 2.4 Creating a New Partition

## 2.4.1 Design the Disk Layout

### Official LFS procedure

LFS is normally installed on a dedicated Linux-native partition.

Space guidance:

```text
Minimal build partition: approximately 10 GB
Good general root partition compromise: approximately 20 GB
Reasonable space for growth: approximately 30 GB
```

The host swap partition may be reused. A separate swap partition is optional.

If swap is created specifically for this build, monitor it with:

```bash
free -h
top
```

### Additional partition considerations

A BIOS boot partition may be required when GRUB is installed on a GPT disk in BIOS mode. It is typically about 1 MB and is not formatted.

Optional convenience partitions include:

```text
/boot
/boot/efi
/home
/usr
/opt
/tmp
/usr/src
```

For a first LFS build, keep the layout simple unless there is a clear reason to split these directories.

### Executed in this build

The retained transcript confirms the final mount point:

```text
/mnt/lfs
```

The exact historical disk and partition device names are not preserved in the conversation transcript.

---

## 2.4.2 Identify the Correct Disk Before Partitioning

### Reproducible execution procedure

List block devices and filesystems:

```bash
lsblk -f
```

Inspect the output carefully. Then define the disk device manually.

Example only:

```bash
export LFS_DISK=/dev/sdb
```

> **Danger:** Replace `/dev/sdb` with the correct disk on the current machine. Do not copy the example blindly.

Open a partitioning tool:

```bash
sudo cfdisk "$LFS_DISK"
```

or:

```bash
sudo fdisk "$LFS_DISK"
```

Create:

1. a Linux-native partition for LFS;
2. a swap partition only if needed.

After saving the partition table, list devices again:

```bash
lsblk -f
```

Define the actual LFS partition manually.

Example only:

```bash
export LFS_PARTITION=/dev/sdb2
```

If a new swap partition was created, define it too.

Example only:

```bash
export LFS_SWAP=/dev/sdb3
```

### Validation

```bash
printf 'LFS_DISK=%s\nLFS_PARTITION=%s\nLFS_SWAP=%s\n' \
  "$LFS_DISK" "$LFS_PARTITION" "${LFS_SWAP:-not-set}"

lsblk -f
```

Do not continue until the selected devices are correct.

---

# 2.5 Creating a Filesystem on the Partition

## Official LFS procedure

LFS assumes an `ext4` root filesystem.

Create it with:

```bash
sudo mkfs -v -t ext4 "$LFS_PARTITION"
```

If a new swap partition was created, initialize it once:

```bash
sudo mkswap "$LFS_SWAP"
```

If an existing host swap partition is reused, do not format it.

## Reproducible execution procedure

Confirm that the variables are set before running destructive commands:

```bash
printf 'LFS_PARTITION=%s\nLFS_SWAP=%s\n' \
  "$LFS_PARTITION" "${LFS_SWAP:-not-set}"
```

Create the root filesystem:

```bash
sudo mkfs -v -t ext4 "$LFS_PARTITION"
```

Initialize new swap only when applicable:

```bash
sudo mkswap "$LFS_SWAP"
```

## Validation

```bash
lsblk -f
```

Expected result:

```text
The LFS root partition reports FSTYPE ext4.
A new swap partition reports FSTYPE swap, if one was created.
```

## Executed in this build

The retained transcript does not preserve the original `mkfs` command output. The later build confirms that the filesystem mounted successfully at `/mnt/lfs` and supported the complete build process.

---

# 2.6 Setting the `$LFS` Variable and the Umask

## Official LFS procedure

Set the mount point variable:

```bash
export LFS=/mnt/lfs
```

Set the file creation mask:

```bash
umask 022
```

The `umask` value ensures that newly created files and directories are writable only by their owner while remaining readable and searchable by other users when default modes are used.

Typical permissions:

```text
Files:       644
Directories: 755
```

## Executed in this build

The following commands were executed and later verified:

```bash
export LFS=/mnt/lfs
umask 022

echo $LFS
umask
```

Recorded output:

```text
/mnt/lfs
0022
```

## Required validation after every login, reboot, or user switch

```bash
echo $LFS
umask
```

Expected output:

```text
/mnt/lfs
0022
```

Some systems print `022` instead of `0022`. Both forms are valid.

## Optional persistence

To persist these values for a login shell, add the following lines to the relevant user's `.bash_profile`:

```bash
export LFS=/mnt/lfs
umask 022
```

When logging in through a graphical display manager, `.bash_profile` may not be loaded for a newly opened terminal. In that case, add the same lines to the relevant user's `.bashrc` before any early return for non-interactive shells.

### Executed in this build

The retained transcript does not confirm whether persistence was configured. The verified manual export was sufficient for the active build session.

---

# 2.7 Mounting the New Partition

## 2.7.1 Mount the LFS Filesystem

### Official LFS procedure

Create the mount point:

```bash
sudo mkdir -pv "$LFS"
```

Mount the filesystem:

```bash
sudo mount -v -t ext4 "$LFS_PARTITION" "$LFS"
```

Set the owner and permissions of the mounted LFS root directory:

```bash
sudo chown root:root "$LFS"
sudo chmod 755 "$LFS"
```

### Reproducible execution procedure

Ensure the variables are set:

```bash
export LFS=/mnt/lfs
umask 022

printf 'LFS=%s\nLFS_PARTITION=%s\n' "$LFS" "$LFS_PARTITION"
```

Mount and set permissions:

```bash
sudo mkdir -pv "$LFS"
sudo mount -v -t ext4 "$LFS_PARTITION" "$LFS"
sudo chown root:root "$LFS"
sudo chmod 755 "$LFS"
```

### Validation

```bash
findmnt "$LFS"
df -h "$LFS"
mount | grep " $LFS "
ls -ld "$LFS"
```

Expected conditions:

```text
The mount point is /mnt/lfs.
The filesystem type is ext4.
The owner is root:root.
The mode is drwxr-xr-x (755).
The mount options do not include nosuid or nodev.
```

If `nosuid` or `nodev` appears, remount the filesystem without restrictive options before continuing.

### Executed in this build

The confirmed mount point used throughout this build was:

```text
/mnt/lfs
```

The original device-specific mount command is not preserved in the transcript.

---

## 2.7.2 Optional Multi-Partition Layout

### Official LFS procedure

For separate partitions such as `/home`, mount the root filesystem first, then create and mount the nested mount point:

```bash
sudo mkdir -pv "$LFS"
sudo mount -v -t ext4 /dev/<root-partition> "$LFS"
sudo mkdir -v "$LFS/home"
sudo mount -v -t ext4 /dev/<home-partition> "$LFS/home"
```

### Executed in this build

The retained transcript does not show a multi-partition layout. Do not add optional mounts unless the current build design requires them.

---

## 2.7.3 Enable Swap When Used

### Official LFS procedure

If a swap partition is used, ensure that it is enabled:

```bash
sudo /sbin/swapon -v "$LFS_SWAP"
```

### Validation

```bash
swapon --show
free -h
```

### Executed in this build

The transcript later showed active swap usage inside the Debian VM during performance diagnosis. The exact swap device is not preserved in the retained transcript.

---

## 2.7.4 Reboot Persistence

### Official LFS procedure

If the host will be restarted during the build, either remount the LFS filesystem manually after rebooting or add a host-side `/etc/fstab` entry.

Template:

```text
/dev/<lfs-partition> /mnt/lfs ext4 defaults 1 1
```

If a dedicated swap partition was created, add it to the host `/etc/fstab` as appropriate.

### Validation after reboot

```bash
export LFS=/mnt/lfs
umask 022

findmnt "$LFS"
echo $LFS
umask
```

### Executed in this build

The transcript does not confirm whether the Debian host `/etc/fstab` was modified. The build did use `/mnt/lfs` consistently across sessions.

---

# Chapter 2 Troubleshooting Notes

## Host Resources Below the Book Recommendation

### Symptom observed later

During final Glibc testing, some tests timed out.

A diagnostic showed approximately:

```text
RAM total: 3.9 GiB
```

Desktop applications such as GNOME and Firefox were consuming resources.

### Cause

The VM was functional but resource-constrained relative to the book recommendation.

### Solution that worked

Close unnecessary applications, leave only the terminal open, and rerun time-sensitive tests sequentially.

### Rebuild note

A VM with more RAM is preferable. When that is not possible, keep the graphical workload minimal during heavy builds and test suites.

---

# End-to-End Chapter 2 Execution Checklist

Run these commands on the Debian host in order.

## A. Validate the host

```bash
bash version-check.sh | tee version-check-output.txt
```

Proceed only when all required checks report `OK`.

## B. Identify storage

```bash
lsblk -f
```

Set device variables manually after inspection:

```bash
export LFS_DISK=/dev/<correct-disk>
export LFS_PARTITION=/dev/<correct-lfs-partition>
# Optional only when a new swap partition exists:
export LFS_SWAP=/dev/<correct-swap-partition>
```

## C. Create the filesystem

```bash
sudo mkfs -v -t ext4 "$LFS_PARTITION"
```

Optional new swap initialization:

```bash
sudo mkswap "$LFS_SWAP"
```

## D. Set the LFS environment

```bash
export LFS=/mnt/lfs
umask 022

echo $LFS
umask
```

Expected:

```text
/mnt/lfs
0022
```

## E. Mount the filesystem

```bash
sudo mkdir -pv "$LFS"
sudo mount -v -t ext4 "$LFS_PARTITION" "$LFS"
sudo chown root:root "$LFS"
sudo chmod 755 "$LFS"
```

## F. Enable swap when applicable

```bash
sudo /sbin/swapon -v "$LFS_SWAP"
```

## G. Validate the mounted filesystem

```bash
findmnt "$LFS"
df -h "$LFS"
mount | grep " $LFS "
ls -ld "$LFS"
swapon --show
```

Do not continue unless:

```text
/mnt/lfs is mounted.
The root filesystem is ext4.
The mount does not use nosuid or nodev.
/mnt/lfs is owned by root:root.
/mnt/lfs has mode 755.
```

---

# Historical Record for This Build

## Confirmed from the build conversation

```text
Host environment: Debian VM on VirtualBox
LFS mount point:  /mnt/lfs
Verified LFS var: /mnt/lfs
Verified umask:   0022
```

## Not preserved in the retained transcript

```text
Exact disk device
Exact LFS partition device
Exact swap device
Original mkfs output
Original mount output
Original version-check.sh output
Whether the Debian host /etc/fstab was modified
```

These values are intentionally not guessed.

---

# Approval State

```text
APPROVED FOR GITHUB UPLOAD
Scope: Chapter 2 only
Branch: docs-audit-lfs13
Commit: not created
Push: not performed
```

The file is self-contained: it includes all Chapter 2 execution commands, validation commands, safety gates, build-specific facts confirmed from the transcript, and explicit variables where the original device names are unavailable.
