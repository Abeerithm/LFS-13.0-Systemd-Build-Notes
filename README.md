# Linux From Scratch 13.0-systemd Build Notes

This repository documents my practical journey of building a Linux system from source using the official Linux From Scratch 13.0-systemd book.

This project is maintained by Abeer Alkhalifa as part of a Linux systems documentation portfolio.

The purpose of this project is to understand how a Linux system is built from the ground up, starting from host preparation and the temporary toolchain, then entering the chroot environment, and finally building a complete bootable Linux system.

---

## Project Goal

The main goals of this project are:

* Build Linux From Scratch manually step by step.
* Document each chapter clearly.
* Record commands, errors, fixes, and lessons learned.
* Create a clean technical reference for future Linux system customization.
* Build a strong foundation for creating controlled Linux environments for academic and lab use.

---

## Base Information

| Item                 | Details                         |
| -------------------- | ------------------------------- |
| LFS Version          | 13.0-systemd                    |
| Host System          | Debian on VirtualBox            |
| Build Path           | `/mnt/lfs`                      |
| Target Architecture  | x86_64                          |
| Init System          | systemd                         |
| Documentation Folder | `docs/`                         |
| Main Book            | Linux From Scratch 13.0-systemd |

---

## Repository Structure

```text
LFS-13.0-Systemd-Build-Notes/
├── README.md
├── docs/
│   ├── chapter-01-introduction.md
│   ├── chapter-02-preparing-host-system.md
│   ├── chapter-03-packages-and-patches.md
│   ├── chapter-04-final-preparations.md
│   ├── chapter-05-compiling-cross-toolchain.md
│   ├── chapter-06-cross-compiling-temporary-tools.md
│   ├── chapter-07-entering-chroot-and-building-tools.md
│   ├── chapter-08-installing-basic-system-software.md
│   ├── chapter-09-system-configuration.md
│   ├── chapter-10-making-lfs-system-bootable.md
│   └── chapter-11-the-end.md
├── troubleshooting/
├── scripts/
└── screenshots/
```

---

## Current Build Status

The build has reached the `chroot` environment and is currently in Chapter 8: Installing Basic System Software.

The temporary toolchain and temporary tools stages have been completed successfully.

Current package: `Tcl 8.6.17`

Current working directory inside chroot: `/sources/tcl8.6.17/unix`

Current source directory variable: `SRCDIR=/sources/tcl8.6.17`

---

## Completed Major Stages

The following major stages have been completed:

* Preparing the host system
* Creating the LFS partition and mount point
* Downloading packages and patches
* Creating the temporary toolchain
* Building cross-compilation tools
* Building temporary tools
* Changing ownership to `root`
* Mounting virtual kernel file systems
* Entering the chroot environment
* Creating essential system directories
* Creating basic `/etc/passwd` and `/etc/group` files
* Creating the `tester` user
* Creating initial log files
* Building temporary tools inside chroot
* Cleaning temporary documentation and `.la` files
* Removing `/tools`
* Starting Chapter 8 final system software

---

## Chapter Progress

| Chapter                                                  | Status                    |
| -------------------------------------------------------- | ------------------------- |
| Chapter 1 - Introduction                                 | In progress documentation |
| Chapter 2 - Preparing the Host System                    | Completed                 |
| Chapter 3 - Packages and Patches                         | Completed                 |
| Chapter 4 - Final Preparations                           | Completed                 |
| Chapter 5 - Compiling a Cross-Toolchain                  | Completed                 |
| Chapter 6 - Cross Compiling Temporary Tools              | Completed                 |
| Chapter 7 - Entering Chroot and Building Temporary Tools | Completed                 |
| Chapter 8 - Installing Basic System Software             | In progress               |
| Chapter 9 - System Configuration                         | Not started               |
| Chapter 10 - Making the LFS System Bootable              | Not started               |
| Chapter 11 - The End                                     | Not started               |

---

## Important Build Notes

The LFS base path before entering chroot is:

```bash
/mnt/lfs
```

The main environment variable used before chroot was:

```bash
LFS=/mnt/lfs
```

The temporary toolchain was built using:

```bash
/mnt/lfs/tools
```

The target triplet used during the build was:

```bash
x86_64-lfs-linux-gnu
```

Inside the chroot environment, the system is built from:

```bash
/sources
```

---

## Important Troubleshooting Notes

Several issues were encountered and resolved during the build.

### Glibc Test Timeouts

Some Glibc tests failed because of timeouts inside VirtualBox.

The likely cause was high system load, GNOME resource usage, Firefox usage, and high I/O wait.

The issue was handled by closing unnecessary applications and running the failed tests individually with a higher timeout factor:

```bash
TIMEOUTFACTOR=30 make -j1 test t=<test-name>
```

The following tests passed after being run individually:

```text
stdlib/test-bz22786
stdlib/test-cxa_atexit-race2
libio/tst-asprintf-null
```

The remaining known failure was:

```text
io/tst-lchmod
```

This failure is considered expected inside the chroot environment.

### Glibc Installation Issue

During Glibc installation, a previous `sed` command caused a Makefile issue where `$echo` remained instead of `echo`.

The error message was similar to:

```text
/bin/sh: line 1: cho: command not found
```

The issue was fixed before continuing the build.

---

## Documentation Method

Each chapter document is intended to include:

* Chapter purpose
* Required commands
* Explanation of important commands
* Expected output
* Verification steps
* Problems encountered
* Fixes applied
* Final result

This approach keeps the main README clean while keeping the detailed technical work inside the `docs/` directory.

---

## Why This Project Matters

Linux From Scratch is not just a Linux installation project.

It is a deep learning process that explains how the core parts of a Linux system are built and connected, including:

* Toolchains
* Compilers
* Libraries
* Shell utilities
* System directories
* Boot process
* System configuration
* Package compilation
* Dependency relationships

This knowledge is useful for Linux administration, operating system customization, secure system design, and building controlled environments for labs or academic use.

---

## Future Direction

After completing the LFS build, this repository may be extended with:

* More detailed chapter notes
* Build screenshots
* Troubleshooting records
* Reusable helper scripts
* System customization notes
* Notes for building controlled academic Linux environments

---

## Disclaimer

This repository is not a replacement for the official Linux From Scratch book.

It is a personal technical documentation project based on my own build environment, practical notes, errors, fixes, and learning process.

Commands and notes may need adjustment on other systems.
=======
# Linux From Scratch 13.0 Systemd — Build Notes

This repository documents my hands-on journey building a Linux From Scratch (LFS) 13.0 Systemd system from source.

## Purpose

The goal is to preserve practical build notes, successful commands, validation steps, and troubleshooting records collected while following the official LFS book.

This repository is not an official LFS guide. It is a practical technical record intended to support future rebuilds and help others understand the build process more clearly.

## Reference

Official Linux From Scratch documentation:

https://www.linuxfromscratch.org/

## Build Environment

* Host system: Debian running inside VirtualBox
* LFS edition: 13.0 Systemd
* Main LFS path: `/mnt/lfs`
* Build method: Manual compilation from source
* Current progress: Preparing for kernel configuration and boot setup

## Repository Structure

* `docs/` — Notes organized by build stage
* `troubleshooting/` — Errors, causes, and verified fixes
* `scripts/` — Small helper and verification scripts
* `screenshots/` — Selected screenshots of important milestones

## Important Note

Always refer to the official LFS book before executing commands. The notes in this repository reflect a specific build environment and may require adjustments on other systems.

