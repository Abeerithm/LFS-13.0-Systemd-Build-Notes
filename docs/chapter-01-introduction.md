# Chapter 1 — Introduction

> Repository: `github.com/Abeerithm/LFS-13.0-Systemd-Build-Notes`
> Local repository path: `~/Projects/LFS-13.0-Systemd-Build-Notes`
> Documentation status: Initial GitHub version
> Book baseline: *Linux From Scratch — Version 13.0-systemd*
> Build environment: Debian on VirtualBox
> Main LFS mount point: `/mnt/lfs`

---

## 1.1 Project Overview

This repository documents my practical journey of building a Linux system from source using the official Linux From Scratch 13.0-systemd book.

The goal is not only to complete the LFS build, but also to understand how the main components of a Linux system are created, connected, configured, and prepared to become a bootable operating system.

This documentation follows the official LFS book structure while adding practical notes from my own build environment, including commands, verification steps, troubleshooting notes, and lessons learned during the process.

---

## 1.2 Why Linux From Scratch?

Linux From Scratch is a deep technical learning project.

Instead of installing a ready-made Linux distribution, LFS guides the builder through the process of compiling and assembling the system manually.

This helps explain important Linux concepts such as:

* the build host and target system relationship;
* temporary toolchains;
* cross-compilation;
* system libraries;
* essential user-space tools;
* chroot environments;
* filesystem hierarchy;
* system configuration;
* kernel and bootloader preparation.

The value of this project is in understanding what normally remains hidden inside standard Linux distributions.

---

## 1.3 Project Goals

The main goals of this repository are:

1. document the LFS 13.0-systemd build step by step;
2. preserve important commands and verification outputs;
3. record errors, causes, and fixes;
4. create a clean reference for future rebuilds;
5. build practical knowledge for Linux system customization;
6. prepare a foundation for creating controlled Linux environments for academic or lab use.

---

## 1.4 Build Environment

The build was performed using:

| Item                    | Value          |
| ----------------------- | -------------- |
| LFS Version             | 13.0-systemd   |
| Host System             | Debian         |
| Virtualization Platform | VirtualBox     |
| Main LFS Path           | `/mnt/lfs`     |
| Target Architecture     | x86_64         |
| Init System             | systemd        |
| Documentation Format    | Markdown       |
| Version Control         | Git and GitHub |

The project files are maintained locally under:

```bash
~/Projects/LFS-13.0-Systemd-Build-Notes
```

The LFS build itself uses:

```bash
/mnt/lfs
```

---

## 1.5 Documentation Method

This repository separates documentation into chapter files under the `docs/` directory.

Each chapter is intended to include:

* chapter purpose;
* official LFS procedure;
* commands used in this build;
* explanation of important commands;
* expected output or verification steps;
* environment-specific notes;
* problems encountered;
* fixes applied;
* final chapter status.

This approach keeps the repository structured and makes it easier to review, update, and expand each chapter separately.

---

## 1.6 Repository Structure

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
│   ├── chapter-07-entering-chroot-and-temporary-tools.md
│   ├── chapter-08-installing-basic-system-software.md
│   ├── chapter-09-system-configuration.md
│   ├── chapter-10-making-lfs-system-bootable.md
│   └── chapter-11-the-end.md
├── screenshots/
├── scripts/
└── troubleshooting/
```

---

## 1.7 Current Build Status

The build has already progressed beyond the early preparation stages.

Completed major stages include:

* preparing the Debian host system;
* creating and mounting the LFS filesystem;
* downloading packages and patches;
* creating the `lfs` build user;
* setting up the temporary build environment;
* building the cross-toolchain;
* cross-compiling temporary tools;
* entering the `chroot` environment;
* completing additional temporary tools inside chroot;
* starting Chapter 8 final system software builds.

The current active build stage is:

```text
Chapter 8 — Installing Basic System Software
```

The current/next package is:

```text
Tcl 8.6.17
```

The current working directory inside chroot is:

```bash
/sources/tcl8.6.17/unix
```

---

## 1.8 Important Notes for Readers

This repository is not a replacement for the official Linux From Scratch book.

The official book remains the authoritative reference.

This repository documents one practical build experience based on a specific Debian VirtualBox environment. Some commands, outputs, or troubleshooting notes may differ on another host system.

Before reusing any command, especially disk, mount, ownership, or bootloader commands, verify the current system environment carefully.

---

## 1.9 Safety Notes

Several LFS commands can affect important system locations if environment variables are wrong.

Before running sensitive commands, always verify:

```bash
echo "$LFS"
whoami
pwd
```

Expected LFS path before chroot:

```text
/mnt/lfs
```

When working before chroot, commands may run as either:

```text
root
```

or:

```text
lfs
```

depending on the chapter.

When working inside chroot, commands run as:

```text
root inside the LFS environment
```

Never assume the current context. Always verify it.

---

## 1.10 Future Direction

After completing the base LFS system, this documentation may be extended with:

* additional troubleshooting logs;
* screenshots from important build stages;
* helper scripts for repeated verification;
* notes for rebuilding the system;
* notes for Linux system customization;
* controlled academic Linux environment concepts.

The long-term value of this project is to build a strong technical foundation for Linux customization, secure system design, and controlled lab environments.

---

## 1.11 Disclaimer

This documentation reflects my own LFS build process and environment.

It may include environment-specific decisions, troubleshooting steps, and observations that are not part of the official LFS book.

Use the notes carefully, verify commands before execution, and always compare critical steps with the official Linux From Scratch documentation.
