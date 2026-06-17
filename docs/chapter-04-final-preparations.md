# Chapter 4 — Final Preparations

> Repository: `github.com/Abeerithm/LFS-13.0-Systemd-Build-Notes`  
> Local repository path: `~/Projects/LFS-13.0-Systemd-Build-Notes`  
> Review branch: `docs-audit-lfs13`  
> Status: **Draft for review — do not commit or push yet**  
> Book source: *Linux From Scratch — Version 13.0-systemd*, Chapter 4  
> Actual LFS mount point used in this build: `/mnt/lfs`

---

## 4.1 Objective

This chapter prepares the isolated build environment used in Chapters 5 and 6.

The required outcome is:

1. a limited directory structure inside `/mnt/lfs`;
2. compatibility symlinks such as `/mnt/lfs/bin -> usr/bin`;
3. a dedicated cross-toolchain directory at `/mnt/lfs/tools`;
4. an unprivileged user named `lfs`;
5. ownership of the temporary-build directories assigned to `lfs`;
6. a clean Bash login environment for the `lfs` user;
7. a deterministic `PATH`, locale, target triplet, and `CONFIG_SITE`;
8. a controlled parallel-build setting through `MAKEFLAGS`.

The chapter ends when a fresh `lfs` login shell can verify the expected environment.

---

## Documentation convention

Every section separates:

### Official book command
Commands taken from LFS 13.0-systemd.

### Executed in this build
Commands or outcomes confirmed from the actual project history.

### Environment-specific note
Operational decisions specific to the Debian VirtualBox host used for this build.

---

## 4.1.1 Starting conditions

Run the commands in Sections 4.2 and 4.3 as `root` on the Debian host, not inside `chroot`.

Before continuing, verify the LFS mount point:

```bash
export LFS=/mnt/lfs
umask 022

echo "$LFS"
umask
findmnt "$LFS"
```

Expected:

```text
/mnt/lfs
0022
```

`findmnt` must confirm that the LFS filesystem is mounted at:

```text
/mnt/lfs
```

### Executed in this build

The active project mount point was:

```text
/mnt/lfs
```

The same value was later verified again before entering `chroot`.

---

## 4.2 Creating a Limited Directory Layout in the LFS Filesystem

### Purpose

The temporary programs built in Chapters 5 and 6 must be installed into a controlled directory hierarchy.

The Chapter 6 temporary programs, together with Glibc and Libstdc++ from Chapter 5, are installed into their intended final locations so that the Chapter 8 builds can overwrite them cleanly.

---

### 4.2.1 Create the limited directory hierarchy

### Official book command

Run as `root`:

```bash
mkdir -pv $LFS/{etc,var} $LFS/usr/{bin,lib,sbin}
```

### Executed in this build

The project used:

```bash
mkdir -pv $LFS/{etc,var} $LFS/usr/{bin,lib,sbin}
```

with:

```text
LFS=/mnt/lfs
```

### Verify

```bash
ls -ld \
  "$LFS/etc" \
  "$LFS/var" \
  "$LFS/usr" \
  "$LFS/usr/bin" \
  "$LFS/usr/lib" \
  "$LFS/usr/sbin"
```

Expected paths:

```text
/mnt/lfs/etc
/mnt/lfs/var
/mnt/lfs/usr
/mnt/lfs/usr/bin
/mnt/lfs/usr/lib
/mnt/lfs/usr/sbin
```

---

### 4.2.2 Create compatibility symlinks

### Official book command

Run as `root`:

```bash
for i in bin lib sbin; do
  ln -sv usr/$i $LFS/$i
done
```

### Result

The links must become:

```text
/mnt/lfs/bin  -> usr/bin
/mnt/lfs/lib  -> usr/lib
/mnt/lfs/sbin -> usr/sbin
```

### Why these links exist

Modern Linux systems commonly use a merged `/usr` layout.

These symlinks preserve compatibility with programs and scripts that still expect:

```text
/bin
/lib
/sbin
```

while the actual files live under:

```text
/usr/bin
/usr/lib
/usr/sbin
```

### Verify

```bash
ls -ld "$LFS/bin" "$LFS/lib" "$LFS/sbin"
```

Expected pattern:

```text
/mnt/lfs/bin  -> usr/bin
/mnt/lfs/lib  -> usr/lib
/mnt/lfs/sbin -> usr/sbin
```

---

### 4.2.3 Create `/lib64` on x86_64 only

### Official book command

```bash
case $(uname -m) in
  x86_64) mkdir -pv $LFS/lib64 ;;
esac
```

### Executed in this build

The project target was x86_64.

The `/mnt/lfs/lib64` directory was created and later verified before entering `chroot`.

Later verification from the actual build showed:

```text
/mnt/lfs/lib64  root root
```

after ownership was transferred back to `root` in Chapter 7.

### Verify

```bash
uname -m
ls -ld "$LFS/lib64"
```

Expected architecture:

```text
x86_64
```

Expected path:

```text
/mnt/lfs/lib64
```

### Important rule

Do **not** create:

```text
/usr/lib64
```

The LFS book deliberately avoids `/usr/lib64`.

A stray `/usr/lib64` may break the toolchain.

Verify that it does not exist:

```bash
test ! -e "$LFS/usr/lib64" && echo "OK: no /usr/lib64"
```

Expected:

```text
OK: no /usr/lib64
```

---

### 4.2.4 Create the dedicated cross-toolchain directory

### Official book command

```bash
mkdir -pv $LFS/tools
```

### Purpose

The cross-compiler and linker built in Chapter 5 are installed under:

```text
/mnt/lfs/tools
```

This keeps the temporary cross-toolchain separate from both:

```text
the Debian host tools
the final LFS system tools
```

### Executed in this build

The `/mnt/lfs/tools` directory was used successfully throughout Chapters 5 and 6.

Later verification showed:

```text
/mnt/lfs/tools
```

before it was intentionally deleted during Chapter 7 cleanup.

### Verify

```bash
ls -ld "$LFS/tools"
```

Expected:

```text
/mnt/lfs/tools
```

---

## 4.3 Adding the `lfs` User

### Purpose

Chapters 5 and 6 must be built as an unprivileged user.

Building the cross-toolchain as `root` is unsafe because a typo could damage the host system.

---

### 4.3.1 Create the `lfs` group

### Official book command

Run as `root`:

```bash
groupadd lfs
```

### Verify

```bash
getent group lfs
```

Expected pattern:

```text
lfs:x:<gid>:
```

The numeric GID may differ between hosts.

---

### 4.3.2 Create the `lfs` user

### Official book command

```bash
useradd -s /bin/bash -g lfs -m -k /dev/null lfs
```

### Meaning of the options

| Option | Purpose |
|---|---|
| `-s /bin/bash` | Sets Bash as the default shell |
| `-g lfs` | Assigns the primary group `lfs` |
| `-m` | Creates `/home/lfs` |
| `-k /dev/null` | Prevents copying skeleton files from `/etc/skel` |

### Verify

```bash
getent passwd lfs
```

Expected pattern:

```text
lfs:x:<uid>:<gid>::/home/lfs:/bin/bash
```

Verify the home directory:

```bash
ls -ld /home/lfs
```

Expected owner and group:

```text
lfs lfs
```

---

### 4.3.3 Set a password for `lfs`

### Official book command

```bash
passwd lfs
```

Enter a temporary password when prompted.

### Security note

The password is not stored in this repository.

Use a temporary local password suitable for the build VM.

---

### 4.3.4 Assign ownership of the temporary-build directories

### Official book commands

```bash
chown -v lfs $LFS/{usr{,/*},var,etc,tools}
```

For x86_64:

```bash
case $(uname -m) in
  x86_64) chown -v lfs $LFS/lib64 ;;
esac
```

### Why ownership changes here

The `lfs` user must be able to install the temporary toolchain and temporary utilities into the LFS filesystem during Chapters 5 and 6.

### Executed in this build

The project successfully completed Chapters 5 and 6 as the `lfs` user.

At the beginning of Chapter 7, ownership was intentionally transferred back to `root:root`, confirming that the Chapter 4 ownership stage had been in effect.

### Verify before Chapter 5

```bash
ls -ld \
  "$LFS/etc" \
  "$LFS/var" \
  "$LFS/tools" \
  "$LFS/usr" \
  "$LFS/usr/bin" \
  "$LFS/usr/lib" \
  "$LFS/usr/sbin" \
  "$LFS/lib64"
```

Expected owner for writable build directories:

```text
lfs
```

---

### 4.3.5 Switch to the `lfs` user

### Official book command

```bash
su - lfs
```

### Why the hyphen matters

Use:

```bash
su - lfs
```

not:

```bash
su lfs
```

The hyphen requests a login shell and ensures that the `lfs` home directory and login startup files are used.

### Executed in this build

Chapters 5 and 6 were built from a shell prompt similar to:

```text
lfs@debian:/mnt/lfs/sources$
```

---

## 4.4 Setting Up the Environment

### Purpose

The `lfs` user must work in a clean, deterministic shell environment.

The environment must not accidentally reuse host-specific paths, locale settings, cached command paths, or distribution-specific `config.site` values.

---

### 4.4.1 Create `.bash_profile`

Run as the `lfs` user:

```bash
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
```

### Purpose

The new login shell starts a clean Bash process with only:

```text
HOME
TERM
PS1
```

preserved initially.

This reduces contamination from the Debian host shell environment.

### Verify

```bash
cat ~/.bash_profile
```

Expected:

```text
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
```

---

### 4.4.2 Create `.bashrc`

### Official book command

```bash
cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
CONFIG_SITE=$LFS/usr/share/config.site
export LFS LC_ALL LFS_TGT PATH CONFIG_SITE
EOF
```

### Explanation of each setting

| Setting | Purpose |
|---|---|
| `set +h` | Disables Bash command hashing so newly built tools are found immediately |
| `umask 022` | Applies safe default file and directory permissions |
| `LFS=/mnt/lfs` | Defines the LFS build root |
| `LC_ALL=POSIX` | Makes command output deterministic and prevents locale-related build issues |
| `LFS_TGT=$(uname -m)-lfs-linux-gnu` | Defines the cross-compilation target triplet |
| `PATH=/usr/bin` | Starts from a controlled base path |
| `if [ ! -L /bin ]; then PATH=/bin:$PATH; fi` | Supports hosts where `/bin` is not merged into `/usr/bin` |
| `PATH=$LFS/tools/bin:$PATH` | Prioritizes the cross-toolchain once it is installed |
| `CONFIG_SITE=$LFS/usr/share/config.site` | Prevents configure scripts from reading Debian host-specific settings |
| `export ...` | Makes the variables available to child processes |

### Verify

```bash
cat ~/.bashrc
```

---

### 4.4.3 Move `/etc/bash.bashrc` out of the way when present

### Official book command

Run as `root` before relying on the clean `lfs` environment:

```bash
[ ! -e /etc/bash.bashrc ] || mv -v /etc/bash.bashrc /etc/bash.bashrc.NOUSE
```

### Purpose

Some host distributions load:

```text
/etc/bash.bashrc
```

for non-login shells.

That file may introduce aliases, functions, paths, or shell customizations that contaminate the LFS build environment.

### Verify

```bash
ls -l /etc/bash.bashrc /etc/bash.bashrc.NOUSE 2>/dev/null
```

Expected outcomes:

```text
either /etc/bash.bashrc never existed
or /etc/bash.bashrc.NOUSE exists
```

### Restore note

At the beginning of Chapter 7, after the `lfs` user is no longer needed, the host file may be restored if desired:

```bash
mv -v /etc/bash.bashrc.NOUSE /etc/bash.bashrc
```

Only run that restore command on the Debian host and only if the `.NOUSE` file exists.

---

### 4.4.4 Configure parallel builds

### Official book command

Append to `~/.bashrc`:

```bash
cat >> ~/.bashrc << "EOF"
export MAKEFLAGS=-j$(nproc)
EOF
```

### Purpose

This allows `make` to use all available logical CPU cores by default.

### Important safety rule

Never use:

```bash
make -j
```

without a numeric value.

Never set:

```bash
MAKEFLAGS=-j
```

without a numeric value.

An unbounded `-j` can spawn excessive build jobs and destabilize the system.

### Environment-specific note

This build ran inside a VM with limited resources.

Later Glibc tests timed out while GNOME and Firefox were open.

For repeat builds, the default book setting is still documented:

```bash
export MAKEFLAGS=-j$(nproc)
```

but heavy test suites may need sequential execution:

```bash
make -j1 check
```

or package-specific sequential reruns.

Do not globally replace the book setting unless the VM remains unstable.

### Verify

```bash
tail -n 3 ~/.bashrc
```

Expected to include:

```text
export MAKEFLAGS=-j$(nproc)
```

---

### 4.4.5 Load the clean environment

### Official book command

```bash
source ~/.bash_profile
```

### Purpose

This starts a clean Bash shell and causes the new `.bashrc` to be read.

### Executed in this build

The later cross-toolchain builds used the expected target-prefixed tools, confirming that the environment loaded successfully.

---

## 4.4.6 Validate the final `lfs` environment

Run as the `lfs` user:

```bash
whoami
```

Expected:

```text
lfs
```

Check the mount point variable:

```bash
echo "$LFS"
```

Expected:

```text
/mnt/lfs
```

Check the `umask`:

```bash
umask
```

Expected:

```text
0022
```

Check the target triplet:

```bash
echo "$LFS_TGT"
```

Expected on this build:

```text
x86_64-lfs-linux-gnu
```

Check the locale:

```bash
echo "$LC_ALL"
```

Expected:

```text
POSIX
```

Check the configure-site override:

```bash
echo "$CONFIG_SITE"
```

Expected:

```text
/mnt/lfs/usr/share/config.site
```

Check the path ordering:

```bash
echo "$PATH" | tr ':' '\n'
```

Expected first entry:

```text
/mnt/lfs/tools/bin
```

Check the parallel-build setting:

```bash
echo "$MAKEFLAGS"
```

Expected pattern:

```text
-j<number-of-logical-cores>
```

Check logical core count:

```bash
nproc
```

Check that command hashing is disabled:

```bash
set -o | grep hashall
```

Expected:

```text
hashall         off
```

---

## 4.5 About Standard Build Units (SBUs)

### Official book concept

An SBU is a relative measure of build time.

The book uses the first compilation of Binutils in Chapter 5 as the baseline:

```text
Binutils Pass 1 = 1 SBU
```

Other package build times are expressed relative to that baseline.

### Example

If Binutils Pass 1 takes:

```text
10 minutes
```

and a later package is listed as:

```text
4.5 SBU
```

the approximate build time is:

```text
45 minutes
```

### Environment-specific note

SBU values are estimates, not guarantees.

They vary with:

```text
CPU
RAM
storage speed
VM configuration
parallel-build settings
background desktop load
```

The resource-pressure behavior observed during Glibc testing confirms that VM workload matters.

---

## 4.6 About Test Suites

### Official book concept

Most packages provide test suites.

The tests help verify that newly built programs behave as expected.

### Practical rules for this repository

1. Run the test suite when the chapter instructions require it.
2. Do not ignore failures without diagnosis.
3. Distinguish between:
   - `PASS`
   - `FAIL`
   - `XFAIL`
   - `XPASS`
   - `ERROR`
   - timeout failures
4. Save meaningful failures and successful fixes under:

```text
docs/troubleshooting/
```

### Environment-specific lesson from this build

Later Glibc test failures were not skipped.

They were diagnosed as resource-related timeouts inside VirtualBox and resolved by:

```text
closing Firefox and unnecessary desktop applications
running affected tests individually
using -j1
using TIMEOUTFACTOR=30
```

That troubleshooting belongs to the Chapter 8 documentation, but the testing discipline begins here.

---

## 4.7 Full clean-rebuild procedure

Run the following sequence from the Debian host.

### Phase A — root preparation

```bash
sudo -i

export LFS=/mnt/lfs
umask 022

echo "$LFS"
umask
findmnt "$LFS"

mkdir -pv $LFS/{etc,var} $LFS/usr/{bin,lib,sbin}

for i in bin lib sbin; do
  ln -sv usr/$i $LFS/$i
done

case $(uname -m) in
  x86_64) mkdir -pv $LFS/lib64 ;;
esac

mkdir -pv $LFS/tools

groupadd lfs

useradd -s /bin/bash -g lfs -m -k /dev/null lfs

passwd lfs

chown -v lfs $LFS/{usr{,/*},var,etc,tools}

case $(uname -m) in
  x86_64) chown -v lfs $LFS/lib64 ;;
esac

[ ! -e /etc/bash.bashrc ] || mv -v /etc/bash.bashrc /etc/bash.bashrc.NOUSE
```

### Phase B — switch to `lfs`

```bash
su - lfs
```

### Phase C — create the isolated shell environment

```bash
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF

cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
CONFIG_SITE=$LFS/usr/share/config.site
export LFS LC_ALL LFS_TGT PATH CONFIG_SITE
EOF

cat >> ~/.bashrc << "EOF"
export MAKEFLAGS=-j$(nproc)
EOF

source ~/.bash_profile
```

### Phase D — validation

```bash
whoami
echo "$LFS"
umask
echo "$LFS_TGT"
echo "$LC_ALL"
echo "$CONFIG_SITE"
echo "$PATH" | tr ':' '\n'
echo "$MAKEFLAGS"
nproc
set -o | grep hashall
```

Expected key values:

```text
whoami:      lfs
LFS:         /mnt/lfs
umask:       0022
LFS_TGT:     x86_64-lfs-linux-gnu
LC_ALL:      POSIX
CONFIG_SITE: /mnt/lfs/usr/share/config.site
PATH first:  /mnt/lfs/tools/bin
hashall:     off
```

---

## 4.8 Repeat-build safety notes

### Do not run Chapters 5 and 6 as `root`

Use:

```bash
su - lfs
```

### Do not create `/usr/lib64`

Verify:

```bash
test ! -e "$LFS/usr/lib64" && echo "OK: no /usr/lib64"
```

### Do not allow host contamination

Confirm:

```bash
echo "$PATH" | tr ':' '\n'
echo "$CONFIG_SITE"
echo "$LC_ALL"
set -o | grep hashall
```

### Do not use unbounded parallel builds

Never run:

```bash
make -j
```

without a number.

### Re-check after reopening a terminal

Run:

```bash
whoami
echo "$LFS"
umask
echo "$LFS_TGT"
echo "$MAKEFLAGS"
```

---

## 4.9 Evidence capture for GitHub

Create an evidence directory:

```bash
REPO_ROOT=~/Projects/LFS-13.0-Systemd-Build-Notes

mkdir -pv "$REPO_ROOT/logs/chapter-04"
```

Capture root-side directory layout:

```bash
ls -ld \
  "$LFS/etc" \
  "$LFS/var" \
  "$LFS/tools" \
  "$LFS/usr" \
  "$LFS/usr/bin" \
  "$LFS/usr/lib" \
  "$LFS/usr/sbin" \
  "$LFS/bin" \
  "$LFS/lib" \
  "$LFS/sbin" \
  "$LFS/lib64" \
  | tee "$REPO_ROOT/logs/chapter-04/lfs-directory-layout.txt"
```

Capture user and group records:

```bash
getent passwd lfs \
  | tee "$REPO_ROOT/logs/chapter-04/lfs-user.txt"

getent group lfs \
  | tee "$REPO_ROOT/logs/chapter-04/lfs-group.txt"
```

Capture the final `lfs` environment from the `lfs` shell:

```bash
{
  echo "whoami=$(whoami)"
  echo "LFS=$LFS"
  echo "umask=$(umask)"
  echo "LFS_TGT=$LFS_TGT"
  echo "LC_ALL=$LC_ALL"
  echo "CONFIG_SITE=$CONFIG_SITE"
  echo "MAKEFLAGS=$MAKEFLAGS"
  echo "PATH=$PATH"
  set -o | grep hashall
} | tee "$REPO_ROOT/logs/chapter-04/lfs-environment.txt"
```

### Audit note

The original terminal output from Chapter 4 is not fully preserved in the retained conversation history.

Before marking this chapter final, rerun or capture the non-destructive verification commands and add the resulting logs.

---

## Chapter 4 approval checklist

Before moving to Chapter 5 documentation:

```text
[ ] Review all commands in this file
[ ] Confirm /mnt/lfs is mounted
[ ] Confirm /mnt/lfs/bin -> usr/bin
[ ] Confirm /mnt/lfs/lib -> usr/lib
[ ] Confirm /mnt/lfs/sbin -> usr/sbin
[ ] Confirm /mnt/lfs/lib64 exists on x86_64
[ ] Confirm /mnt/lfs/usr/lib64 does not exist
[ ] Confirm /mnt/lfs/tools exists
[ ] Confirm the lfs user and group exist
[ ] Confirm the lfs user owns the temporary-build directories
[ ] Confirm .bash_profile content
[ ] Confirm .bashrc content
[ ] Confirm PATH starts with /mnt/lfs/tools/bin
[ ] Confirm LFS_TGT=x86_64-lfs-linux-gnu
[ ] Confirm LC_ALL=POSIX
[ ] Confirm CONFIG_SITE=/mnt/lfs/usr/share/config.site
[ ] Confirm hashall is off
[ ] Confirm MAKEFLAGS has a bounded -j value
[ ] Capture logs/chapter-04 evidence
[ ] Do not commit or push until explicit approval
```

---

## Suggested repository placement

Place this reviewed file at:

```text
docs/chapter-04-final-preparations.md
```

Suggested companion evidence files:

```text
logs/chapter-04/lfs-directory-layout.txt
logs/chapter-04/lfs-user.txt
logs/chapter-04/lfs-group.txt
logs/chapter-04/lfs-environment.txt
```

---

## Current approval state

```text
DRAFT — CHAPTER 4 ONLY
DO NOT COMMIT
DO NOT PUSH
WAITING FOR REVIEW AND EVIDENCE CAPTURE
```
