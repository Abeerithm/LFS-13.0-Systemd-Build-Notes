# Chapter 7 — Entering Chroot and Building Additional Temporary Tools

> Repository: `github.com/Abeerithm/LFS-13.0-Systemd-Build-Notes`  
> Local repository path: `~/Projects/LFS-13.0-Systemd-Build-Notes`  
> Review branch: `docs-audit-lfs13`  
> Status: **Draft for review — do not commit or push yet**  
> Book source: *Linux From Scratch — Version 13.0-systemd*, Chapter 7  
> Actual LFS mount point used in this build: `/mnt/lfs`  
> Build mode before chroot: `root` on Debian host  
> Build mode after chroot: `root` inside LFS chroot

---

## 7.1 Objective

This chapter completes the temporary LFS system.

At the end of Chapter 6, the temporary programs exist under:

```text
/mnt/lfs
```

but the build is still running from the Debian host.

Chapter 7 changes that.

The required outcome is:

1. transfer ownership of the LFS tree from host-only user `lfs` to `root`;
2. mount virtual kernel filesystems under `/mnt/lfs`;
3. enter an isolated `chroot` environment;
4. create the full base filesystem hierarchy;
5. create essential configuration files and symlinks;
6. create initial system accounts and groups;
7. install the final missing temporary tools:
   - Gettext
   - Bison
   - Perl
   - Python
   - Texinfo
   - Util-linux
8. remove temporary documentation, `.la` files, and `/tools`;
9. optionally save a reusable backup before Chapter 8.

From Section 7.4 onward, all commands must run inside `chroot`.

---

## Documentation convention

Every section separates:

### Official book command
The command sequence from LFS 13.0-systemd.

### Executed in this build
Commands or outputs confirmed from the actual project history.

### Environment-specific note
Operational details specific to this Debian VirtualBox build.

---

# 7.2 Changing Ownership

## Purpose

The files under `/mnt/lfs` were created by the host-only user:

```text
lfs
```

That account does not exist inside the future LFS system.

If ownership is not corrected, files inside LFS may appear to belong to an unnamed numeric UID. A later user account could accidentally receive the same UID and gain ownership of sensitive files.

Run this section as:

```text
root on the Debian host
```

not as `lfs`, and not inside `chroot`.

---

## 7.2.1 Switch to root and set the environment

```bash
sudo -i
```

Set the project path:

```bash
export LFS=/mnt/lfs
umask 022
```

Verify:

```bash
whoami
echo "$LFS"
umask
```

Expected:

```text
root
/mnt/lfs
0022
```

### Executed in this build

The retained transcript confirmed:

```text
root
/mnt/lfs
0022
```

---

## 7.2.2 Transfer ownership to root

### Official book command

```bash
chown --from lfs -R root:root $LFS/{usr,var,etc,tools}

case $(uname -m) in
  x86_64) chown --from lfs -R root:root $LFS/lib64 ;;
esac
```

### Why `--from lfs` matters

The option limits the ownership change to files currently owned by:

```text
lfs
```

This avoids changing unrelated ownership accidentally.

### Verify

```bash
ls -ld $LFS/{usr,var,etc,tools} $LFS/lib64
```

Expected owner and group:

```text
root root
```

### Executed in this build

The retained transcript confirmed:

```text
/mnt/lfs/etc    root root
/mnt/lfs/lib64  root root
/mnt/lfs/tools  root root
/mnt/lfs/usr    root root
/mnt/lfs/var    root root
```

---

# 7.3 Preparing Virtual Kernel File Systems

## Purpose

Programs inside `chroot` still need controlled communication with the running Debian host kernel.

The kernel exposes virtual filesystems for:

```text
devices
processes
kernel state
runtime state
pseudo terminals
shared memory
```

These filesystems occupy memory, not normal disk space.

Run this section as:

```text
root on the Debian host
```

---

## 7.3.1 Create mount points

```bash
mkdir -pv $LFS/{dev,proc,sys,run}
```

### Executed in this build

The retained transcript confirmed creation of:

```text
/mnt/lfs/dev
/mnt/lfs/proc
/mnt/lfs/sys
/mnt/lfs/run
```

---

## 7.3.2 Bind mount `/dev`

### Official book command

```bash
mount -v --bind /dev $LFS/dev
```

### Why a bind mount is used

A bind mount makes the host `/dev` tree visible inside:

```text
/mnt/lfs/dev
```

This works across hosts even when the kernel or distribution populates `/dev` differently.

### Verify

```bash
findmnt "$LFS/dev"
```

### Executed in this build

The retained transcript confirmed:

```text
TARGET       SOURCE FSTYPE   OPTIONS
/mnt/lfs/dev udev   devtmpfs ...
```

---

## 7.3.3 Mount pseudo terminals, process state, system state, and runtime state

### Official book commands

```bash
mount -vt devpts devpts -o gid=5,mode=0620 $LFS/dev/pts

mount -vt proc proc $LFS/proc

mount -vt sysfs sysfs $LFS/sys

mount -vt tmpfs tmpfs $LFS/run
```

### Meaning of the `devpts` options

| Option | Purpose |
|---|---|
| `gid=5` | Uses group ID 5 for `tty` |
| `mode=0620` | Gives the owning user read/write access and the group write access |

### Verify

```bash
findmnt "$LFS/dev/pts"
findmnt "$LFS/proc"
findmnt "$LFS/sys"
findmnt "$LFS/run"
```

### Executed in this build

The retained transcript confirmed:

```text
/mnt/lfs/dev/pts devpts devpts rw,relatime,gid=5,mode=620,ptmxmode=000
/mnt/lfs/proc    proc   proc   rw,relatime
/mnt/lfs/sys     sysfs  sysfs  rw,relatime
/mnt/lfs/run     tmpfs  tmpfs  rw,relatime,inode64
```

---

## 7.3.4 Prepare `/dev/shm`

### Official book command

```bash
if [ -h $LFS/dev/shm ]; then
  install -v -d -m 1777 $LFS$(realpath /dev/shm)
else
  mount -vt tmpfs -o nosuid,nodev tmpfs $LFS/dev/shm
fi
```

### Why the conditional is required

On some hosts:

```text
/dev/shm
```

is a symlink, often to:

```text
/run/shm
```

On other hosts it is a mount point.

The conditional supports both layouts.

### Verify

```bash
findmnt "$LFS/dev/shm" || ls -ld "$LFS/dev/shm"
```

### Executed in this build

The retained transcript confirmed:

```text
/mnt/lfs/dev/shm tmpfs tmpfs rw,nosuid,nodev,relatime,inode64
```

---

## 7.3.5 Combined mount verification

```bash
findmnt | grep "$LFS"
```

At minimum, verify:

```text
/mnt/lfs/dev
/mnt/lfs/dev/pts
/mnt/lfs/proc
/mnt/lfs/sys
/mnt/lfs/run
```

and either a mount or a valid directory for:

```text
/mnt/lfs/dev/shm
```

---

# 7.4 Entering the Chroot Environment

## Purpose

`chroot` changes the apparent root filesystem from:

```text
/
```

on Debian to:

```text
/mnt/lfs
```

inside the new shell.

From this point onward:

```text
/mnt/lfs/usr/bin/bash
```

appears as:

```text
/usr/bin/bash
```

The host Debian system remains outside the new root.

---

## 7.4.1 Enter chroot

Run as:

```text
root on the Debian host
```

### Official book command

```bash
chroot "$LFS" /usr/bin/env -i \
  HOME=/root \
  TERM="$TERM" \
  PS1='(lfs chroot) \u:\w\$ ' \
  PATH=/usr/bin:/usr/sbin \
  MAKEFLAGS="-j$(nproc)" \
  TESTSUITEFLAGS="-j$(nproc)" \
  /bin/bash --login
```

### Meaning of the important parts

| Part | Purpose |
|---|---|
| `env -i` | Clears inherited host environment variables |
| `HOME=/root` | Defines the root home directory inside chroot |
| `TERM="$TERM"` | Preserves terminal capability information |
| `PS1=...` | Makes the chroot prompt easy to recognize |
| `PATH=/usr/bin:/usr/sbin` | Uses tools installed in LFS, not `/tools/bin` |
| `MAKEFLAGS="-j$(nproc)"` | Enables bounded parallel compilation |
| `TESTSUITEFLAGS="-j$(nproc)"` | Enables parallel test suites where supported |
| `/bin/bash --login` | Starts a login Bash shell inside chroot |

### Critical observation

The path:

```text
/tools/bin
```

is deliberately absent from `PATH`.

The cross-toolchain is no longer used after entering chroot.

### Executed in this build

The retained transcript confirmed entry into chroot with the prompt:

```text
(lfs chroot) I have no name!:/#
```

This prompt is expected before `/etc/passwd` exists.

---

## 7.4.2 Verify chroot context

Run inside chroot:

```bash
pwd
echo "$PATH"
echo "$HOME"
echo "$MAKEFLAGS"
echo "$TESTSUITEFLAGS"
```

Expected:

```text
/
PATH=/usr/bin:/usr/sbin
HOME=/root
```

### Safety rule

Do not run host-side commands by accident after entering chroot.

Before continuing, confirm the prompt contains:

```text
(lfs chroot)
```

---

# 7.5 Creating Directories

## Purpose

Create the full base filesystem hierarchy required by the final LFS system.

Some directories may already exist. Repeating `mkdir -pv` is safe.

Run all commands inside:

```text
chroot
```

---

## 7.5.1 Create root-level directories

```bash
mkdir -pv /{boot,home,mnt,opt,srv}
```

### Executed in this build

The retained transcript confirmed creation of:

```text
/boot
/home
/mnt
/opt
/srv
```

---

## 7.5.2 Create required subdirectories

```bash
mkdir -pv /etc/{opt,sysconfig}

mkdir -pv /lib/firmware

mkdir -pv /media/{floppy,cdrom}

mkdir -pv /usr/{,local/}{include,src}

mkdir -pv /usr/lib/locale

mkdir -pv /usr/local/{bin,lib,sbin}

mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man}

mkdir -pv /usr/{,local/}share/{misc,terminfo,zoneinfo}

mkdir -pv /usr/{,local/}share/man/man{1..8}

mkdir -pv /var/{cache,local,log,mail,opt,spool}

mkdir -pv /var/lib/{color,misc,locate}
```

### Note about manual-page directories

Use:

```bash
man{1..8}
```

with two dots.

This creates:

```text
man1
man2
man3
man4
man5
man6
man7
man8
```

### Executed in this build

The retained transcript confirmed creation of the required directories, including:

```text
/etc/opt
/etc/sysconfig
/lib/firmware
/media/floppy
/media/cdrom
/usr/src
/usr/local
/usr/local/bin
/usr/local/lib
/usr/local/sbin
/usr/share/zoneinfo
/var/cache
/var/log
/var/mail
/var/spool
/var/lib/color
/var/lib/misc
```

---

## 7.5.3 Create compatibility links for runtime paths

```bash
ln -sfv /run /var/run

ln -sfv /run/lock /var/lock
```

### Verify

```bash
ls -ld /var/run /var/lock
```

Expected:

```text
/var/run  -> /run
/var/lock -> /run/lock
```

### Executed in this build

The retained transcript confirmed:

```text
/var/run  -> /run
/var/lock -> /run/lock
```

---

## 7.5.4 Apply special permissions

```bash
install -dv -m 0750 /root

install -dv -m 1777 /tmp /var/tmp
```

### Why these permissions matter

| Path | Mode | Purpose |
|---|---:|---|
| `/root` | `0750` | Prevents unrestricted access to root's home |
| `/tmp` | `1777` | Writable by all users, protected by sticky bit |
| `/var/tmp` | `1777` | Writable by all users, protected by sticky bit |

### Verify

```bash
ls -ld /root /tmp /var/tmp
```

Expected:

```text
drwxr-x--- /root
drwxrwxrwt /tmp
drwxrwxrwt /var/tmp
```

### Executed in this build

The retained transcript confirmed exactly these permission patterns.

---

## 7.5.5 Verify that `/usr/lib64` does not exist

```bash
test ! -e /usr/lib64 && echo "OK: no /usr/lib64"
```

Expected:

```text
OK: no /usr/lib64
```

### Why this is critical

The LFS build deliberately avoids:

```text
/usr/lib64
```

Creating it may break later instructions.

---

# 7.6 Creating Essential Files and Symlinks

Run this section inside:

```text
chroot
```

---

## 7.6.1 Create `/etc/mtab`

```bash
ln -sv /proc/self/mounts /etc/mtab
```

### Verify

```bash
ls -l /etc/mtab
```

Expected:

```text
/etc/mtab -> /proc/self/mounts
```

### Executed in this build

The retained transcript confirmed:

```text
/etc/mtab -> /proc/self/mounts
```

---

## 7.6.2 Create `/etc/hosts`

```bash
cat > /etc/hosts << EOF
127.0.0.1 localhost $(hostname)
::1 localhost
EOF
```

### Verify

```bash
cat /etc/hosts
```

### Executed in this build

The retained transcript confirmed:

```text
127.0.0.1 localhost debian
::1 localhost
```

The hostname inherited from the Debian VM was:

```text
debian
```

---

## 7.6.3 Create `/etc/passwd`

```bash
cat > /etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/usr/bin/false
daemon:x:6:6:Daemon User:/dev/null:/usr/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/run/dbus:/usr/bin/false
systemd-journal-gateway:x:73:73:systemd Journal Gateway:/:/usr/bin/false
systemd-journal-remote:x:74:74:systemd Journal Remote:/:/usr/bin/false
systemd-journal-upload:x:75:75:systemd Journal Upload:/:/usr/bin/false
systemd-network:x:76:76:systemd Network Management:/:/usr/bin/false
systemd-resolve:x:77:77:systemd Resolver:/:/usr/bin/false
systemd-timesync:x:78:78:systemd Time Synchronization:/:/usr/bin/false
systemd-coredump:x:79:79:systemd Core Dumper:/:/usr/bin/false
uuidd:x:80:80:UUID Generation Daemon User:/dev/null:/usr/bin/false
systemd-oom:x:81:81:systemd Out Of Memory Daemon:/:/usr/bin/false
nobody:x:65534:65534:Unprivileged User:/dev/null:/usr/bin/false
EOF
```

### Verify

```bash
cat /etc/passwd
```

### Executed in this build

The retained transcript confirmed the expected `/etc/passwd` entries.

---

## 7.6.4 Create `/etc/group`

```bash
cat > /etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
clock:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
systemd-journal:x:23:
input:x:24:
mail:x:34:
kvm:x:61:
systemd-journal-gateway:x:73:
systemd-journal-remote:x:74:
systemd-journal-upload:x:75:
systemd-network:x:76:
systemd-resolve:x:77:
systemd-timesync:x:78:
systemd-coredump:x:79:
uuidd:x:80:
systemd-oom:x:81:
wheel:x:97:
users:x:999:
nogroup:x:65534:
EOF
```

### Verify

```bash
cat /etc/group
```

### Executed in this build

The retained transcript confirmed the expected `/etc/group` entries.

---

## 7.6.5 Create the temporary `tester` account

Some Chapter 8 test suites need a regular unprivileged user.

```bash
echo "tester:x:101:101::/home/tester:/bin/bash" >> /etc/passwd

echo "tester:x:101:" >> /etc/group

install -o tester -d /home/tester
```

### Verify

```bash
tail -n 1 /etc/passwd
tail -n 1 /etc/group
ls -ld /home/tester
```

Expected:

```text
tester:x:101:101::/home/tester:/bin/bash
tester:x:101:
```

### Executed in this build

The retained transcript confirmed:

```text
tester:x:101:101::/home/tester:/bin/bash
tester:x:101:
drwxr-xr-x 2 tester root ... /home/tester
```

---

## 7.6.6 Start a fresh login shell

```bash
exec /usr/bin/bash --login
```

### Why this matters

Before `/etc/passwd` existed, the prompt showed:

```text
I have no name!
```

A fresh login shell resolves the root username correctly.

### Executed in this build

The retained transcript confirmed the prompt changed from:

```text
(lfs chroot) I have no name!:/#
```

to:

```text
(lfs chroot) root:/#
```

---

## 7.6.7 Create initial login log files

```bash
touch /var/log/{btmp,lastlog,faillog,wtmp}

chgrp -v utmp /var/log/lastlog

chmod -v 664 /var/log/lastlog

chmod -v 600 /var/log/btmp
```

### Verify

```bash
ls -l /var/log/{btmp,lastlog,faillog,wtmp}
```

Expected:

```text
-rw------- root root /var/log/btmp
-rw-rw-r-- root utmp /var/log/lastlog
```

### Executed in this build

The retained transcript confirmed:

```text
-rw------- 1 root root ... /var/log/btmp
-rw-r--r-- 1 root root ... /var/log/faillog
-rw-rw-r-- 1 root utmp ... /var/log/lastlog
-rw-r--r-- 1 root root ... /var/log/wtmp
```

---

# 7.7 Gettext 1.0

## Purpose

For the temporary system, install only:

```text
msgfmt
msgmerge
xgettext
```

Approximate build time:

```text
1.5 SBU
```

Required disk space:

```text
526 MB
```

---

## 7.7.1 Extract, configure, build, and install

```bash
cd /sources

rm -rf gettext-1.0
tar -xf gettext-1.0.tar.xz
cd gettext-1.0

./configure --disable-shared

make

cp -v gettext-tools/src/{msgfmt,msgmerge,xgettext} /usr/bin
```

### Why `--disable-shared` is used

The shared Gettext libraries are not needed yet.

### Verify

```bash
ls -l /usr/bin/{msgfmt,msgmerge,xgettext}
```

### Executed in this build

The retained transcript confirmed:

```text
/usr/bin/msgfmt
/usr/bin/msgmerge
/usr/bin/xgettext
```

---

## 7.7.2 Clean extracted source

```bash
cd /sources
rm -rf gettext-1.0
```

---

# 7.8 Bison 3.8.2

## Purpose

Bison is a parser generator.

Approximate build time:

```text
0.2 SBU
```

Required disk space:

```text
58 MB
```

---

## 7.8.1 Extract, configure, build, and install

```bash
cd /sources

rm -rf bison-3.8.2
tar -xf bison-3.8.2.tar.xz
cd bison-3.8.2

./configure --prefix=/usr \
  --docdir=/usr/share/doc/bison-3.8.2

make

make install
```

### Verify

```bash
bison --version | head -n 1
ls -l /usr/bin/bison
```

### Executed in this build

The retained transcript confirmed:

```text
bison (GNU Bison) 3.8.2
/usr/bin/bison
```

---

## 7.8.2 Clean extracted source

```bash
cd /sources
rm -rf bison-3.8.2
```

---

# 7.9 Perl 5.42.0

## Purpose

Perl is needed by multiple build systems and test suites.

Approximate build time:

```text
0.6 SBU
```

Required disk space:

```text
295 MB
```

---

## 7.9.1 Extract and configure

```bash
cd /sources

rm -rf perl-5.42.0
tar -xf perl-5.42.0.tar.xz
cd perl-5.42.0

sh Configure -des \
  -D prefix=/usr \
  -D vendorprefix=/usr \
  -D useshrplib \
  -D privlib=/usr/lib/perl5/5.42/core_perl \
  -D archlib=/usr/lib/perl5/5.42/core_perl \
  -D sitelib=/usr/lib/perl5/5.42/site_perl \
  -D sitearch=/usr/lib/perl5/5.42/site_perl \
  -D vendorlib=/usr/lib/perl5/5.42/vendor_perl \
  -D vendorarch=/usr/lib/perl5/5.42/vendor_perl
```

---

## 7.9.2 Build and install

```bash
make

make install
```

### Expected informational output

The installer may print:

```text
Manual page installation was disabled by Configure
```

This is not an error for the temporary Perl build.

### Verify

```bash
perl -v | head -n 2

ls -l /usr/bin/perl
```

### Executed in this build

The retained transcript confirmed:

```text
This is perl 5, version 42, subversion 0 (v5.42.0) built for x86_64-linux
/usr/bin/perl
```

---

## 7.9.3 Clean extracted source

```bash
cd /sources
rm -rf perl-5.42.0
```

---

# 7.10 Python 3.14.3

## Purpose

Python is required for later build tooling.

Approximate build time:

```text
0.5 SBU
```

Required disk space:

```text
592 MB
```

---

## 7.10.1 Extract the correct archive

The source archive starts with an uppercase:

```text
P
```

Use:

```bash
cd /sources

rm -rf Python-3.14.3
tar -xf Python-3.14.3.tar.xz
cd Python-3.14.3
```

Do not accidentally extract a different archive whose name starts with lowercase:

```text
python
```

---

## 7.10.2 Configure, build, and install

```bash
./configure --prefix=/usr \
  --enable-shared \
  --without-ensurepip \
  --without-static-libpython

make

make install
```

### Expected SSL message

During the temporary build, OpenSSL is not installed yet.

The build may print:

```text
Could not build the ssl module!
Python requires a OpenSSL 1.1.1 or newer
```

This is expected at this stage.

Confirm that the top-level `make` command did not fail.

### Verify

```bash
python3 --version

ls -l /usr/bin/python3
```

### Executed in this build

The retained transcript confirmed:

```text
Python 3.14.3
/usr/bin/python3 -> python3.14
```

The build summary also confirmed:

```text
0 failed on import
```

---

## 7.10.3 Clean extracted source

```bash
cd /sources
rm -rf Python-3.14.3
```

---

# 7.11 Texinfo 7.2

## Purpose

Texinfo provides programs for reading, writing, and converting GNU Info documentation.

Approximate build time:

```text
0.2 SBU
```

Required disk space:

```text
152 MB
```

---

## 7.11.1 Extract, configure, build, and install

```bash
cd /sources

rm -rf texinfo-7.2
tar -xf texinfo-7.2.tar.xz
cd texinfo-7.2

./configure --prefix=/usr

make

make install
```

### Verify

```bash
makeinfo --version | head -n 1

ls -l /usr/bin/{makeinfo,info}
```

### Executed in this build

The retained transcript confirmed:

```text
texi2any (GNU texinfo) 7.2
/usr/bin/info
/usr/bin/makeinfo -> texi2any
```

---

## 7.11.2 Clean extracted source

```bash
cd /sources
rm -rf texinfo-7.2
```

---

# 7.12 Util-linux 2.41.3

## Purpose

Util-linux provides miscellaneous core system utilities.

Approximate build time:

```text
0.2 SBU
```

Required disk space:

```text
192 MB
```

---

## 7.12.1 Extract the source

```bash
cd /sources

rm -rf util-linux-2.41.3
tar -xf util-linux-2.41.3.tar.xz
cd util-linux-2.41.3
```

---

## 7.12.2 Create the FHS-compliant hardware-clock directory

```bash
mkdir -pv /var/lib/hwclock
```

### Executed in this build

The retained transcript confirmed creation of:

```text
/var/lib/hwclock
```

---

## 7.12.3 Configure, build, and install

```bash
./configure --libdir=/usr/lib \
  --runstatedir=/run \
  --disable-chfn-chsh \
  --disable-login \
  --disable-nologin \
  --disable-su \
  --disable-setpriv \
  --disable-runuser \
  --disable-pylibmount \
  --disable-static \
  --disable-liblastlog2 \
  --without-python \
  ADJTIME_PATH=/var/lib/hwclock/adjtime \
  --docdir=/usr/share/doc/util-linux-2.41.3

make

make install
```

### Meaning of important options

| Option | Purpose |
|---|---|
| `--libdir=/usr/lib` | Keeps shared-library links under `/usr/lib` |
| `--runstatedir=/run` | Places runtime state correctly |
| `ADJTIME_PATH=/var/lib/hwclock/adjtime` | Uses the FHS-compliant hardware-clock path |
| `--without-python` | Avoids unnecessary Python bindings |
| `--disable-*` | Skips components whose dependencies are not installed yet |

### Verify

```bash
ls -l /usr/bin/{lsblk,uuidgen}

ls -l /usr/sbin/{blkid,fdisk}
```

### Executed in this build

The retained transcript confirmed:

```text
/usr/bin/lsblk
/usr/bin/uuidgen
/usr/sbin/blkid
/usr/sbin/fdisk
```

---

## 7.12.4 Clean extracted source

```bash
cd /sources
rm -rf util-linux-2.41.3
```

---

# 7.13 Cleaning Up and Saving the Temporary System

## Purpose

Remove temporary files that should not be inherited by the final Chapter 8 system.

Run the cleaning commands inside:

```text
chroot
```

---

## 7.13.1 Remove temporary documentation

```bash
rm -rf /usr/share/{info,man,doc}/*
```

### Verify

```bash
du -sh /usr/share/{info,man,doc}
```

### Executed in this build

The retained transcript confirmed:

```text
4.0K /usr/share/info
4.0K /usr/share/man
4.0K /usr/share/doc
```

The directories remained, but their temporary contents were removed.

---

## 7.13.2 Remove libtool archive files

```bash
find /usr/{lib,libexec} -name \*.la -delete
```

### Verify

```bash
find /usr/{lib,libexec} -name \*.la
```

Expected:

```text
no output
```

### Executed in this build

The retained transcript confirmed no `.la` files remained.

---

## 7.13.3 Remove `/tools`

```bash
rm -rf /tools
```

### Verify

```bash
test ! -e /tools && echo "/tools removed"
```

Expected:

```text
/tools removed
```

### Executed in this build

The retained transcript confirmed:

```text
/tools removed
```

### Why `/tools` is removed

The cross-toolchain is no longer required.

Removing it saves approximately:

```text
1 GB
```

---

## 7.13.4 Recommended checkpoint

At this point:

```text
Chapter 7 complete
temporary system cleaned
ready for Chapter 8
```

Create a VirtualBox snapshot before starting Chapter 8.

Suggested snapshot name:

```text
LFS-13.0 - Temporary System Ready
```

Suggested description:

```text
Chapter 7 completed.
Temporary tools installed inside chroot.
Temporary documentation and .la files removed.
Initial /tools cross-toolchain removed.
Ready to begin Chapter 8 final system build.
```

### Environment-specific note

A VirtualBox snapshot is a practical host-level checkpoint for this project.

The official book also provides an archive-based backup procedure below.

---

# 7.13.5 Optional official backup procedure

The remaining steps are optional.

They must run:

```text
outside chroot
as root on the Debian host
```

Leave chroot:

```bash
exit
```

Set the host-side environment explicitly:

```bash
export LFS=/mnt/lfs
umask 022
```

Verify:

```bash
whoami
echo "$LFS"
```

Expected:

```text
root
/mnt/lfs
```

Unmount virtual filesystems:

```bash
mountpoint -q $LFS/dev/shm && umount $LFS/dev/shm

umount $LFS/dev/pts

umount $LFS/{sys,proc,run,dev}
```

Verify:

```bash
findmnt | grep "$LFS"
```

The virtual filesystems should no longer appear.

Create the backup archive outside the LFS tree:

```bash
cd $LFS

tar -cJpf $HOME/lfs-temp-tools-13.0-systemd.tar.xz .
```

### Important rule

Do not store the archive under:

```text
/mnt/lfs
```

The backup target must be outside the LFS tree.

---

# 7.13.6 Optional official restore procedure

## Extreme caution

The restore procedure contains:

```bash
rm -rf ./*
```

Running it from the wrong directory as `root` can destroy the Debian host.

Before restoring, run:

```bash
export LFS=/mnt/lfs

test "$LFS" = "/mnt/lfs" \
  && test -d "$LFS/sources" \
  && echo "SAFE CHECK PASSED: LFS=$LFS"
```

Expected:

```text
SAFE CHECK PASSED: LFS=/mnt/lfs
```

Then restore:

```bash
cd $LFS

pwd
```

Expected:

```text
/mnt/lfs
```

Only after verifying the path manually:

```bash
rm -rf ./*

tar -xpf $HOME/lfs-temp-tools-13.0-systemd.tar.xz
```

After restoring, remount virtual filesystems using Section 7.3, then re-enter chroot using Section 7.4.

---

# 7.14 Resuming after reboot or leaving chroot

If the VM is rebooted or the chroot shell is exited, run the following as `root` on the Debian host:

```bash
export LFS=/mnt/lfs
umask 022

findmnt "$LFS"

mkdir -pv $LFS/{dev,proc,sys,run}

mountpoint -q $LFS/dev || mount -v --bind /dev $LFS/dev

mountpoint -q $LFS/dev/pts || \
  mount -vt devpts devpts -o gid=5,mode=0620 $LFS/dev/pts

mountpoint -q $LFS/proc || mount -vt proc proc $LFS/proc

mountpoint -q $LFS/sys || mount -vt sysfs sysfs $LFS/sys

mountpoint -q $LFS/run || mount -vt tmpfs tmpfs $LFS/run

if [ -h $LFS/dev/shm ]; then
  install -v -d -m 1777 $LFS$(realpath /dev/shm)
elif ! mountpoint -q $LFS/dev/shm; then
  mount -vt tmpfs -o nosuid,nodev tmpfs $LFS/dev/shm
fi

findmnt | grep "$LFS"
```

Then enter chroot:

```bash
chroot "$LFS" /usr/bin/env -i \
  HOME=/root \
  TERM="$TERM" \
  PS1='(lfs chroot) \u:\w\$ ' \
  PATH=/usr/bin:/usr/sbin \
  MAKEFLAGS="-j$(nproc)" \
  TESTSUITEFLAGS="-j$(nproc)" \
  /bin/bash --login
```

---

# 7.15 Full package order

Build the Chapter 7 temporary tools in this exact order:

```text
1. Gettext 1.0
2. Bison 3.8.2
3. Perl 5.42.0
4. Python 3.14.3
5. Texinfo 7.2
6. Util-linux 2.41.3
```

Do not reorder packages.

---

# 7.16 Errors and lessons learned

## 1. `I have no name!` immediately after entering chroot

### Symptom

```text
(lfs chroot) I have no name!:/#
```

### Cause

`/etc/passwd` does not exist yet.

### Resolution

Create:

```text
/etc/passwd
/etc/group
```

then run:

```bash
exec /usr/bin/bash --login
```

### Executed result

The prompt changed to:

```text
(lfs chroot) root:/#
```

---

## 2. Missing Python SSL module during temporary build

### Symptom

```text
Could not build the ssl module!
Python requires a OpenSSL 1.1.1 or newer
```

### Cause

OpenSSL is not installed yet.

### Resolution

Do not treat the message as a failure if top-level `make` completes successfully.

The full Python build occurs later in Chapter 8.

---

## 3. Manual page installation disabled during Perl install

### Symptom

```text
Manual page installation was disabled by Configure
```

### Cause

The temporary Perl configuration does not install manual pages.

### Resolution

No action is required.

---

## 4. `/tools` removal is intentional

### Symptom

```text
/tools removed
```

### Meaning

The initial cross-toolchain has completed its job.

### Resolution

Proceed to Chapter 8 only after confirming that the Chapter 7 temporary system is complete and cleaned.

---

## 5. Root context changes twice in this chapter

Before Section 7.4:

```text
root on Debian host
```

After Section 7.4:

```text
root inside LFS chroot
```

Always confirm the prompt before running destructive commands.

---

# 7.17 GitHub evidence capture

Create a Chapter 7 evidence directory on the Debian host:

```bash
REPO_ROOT=~/Projects/LFS-13.0-Systemd-Build-Notes

mkdir -pv "$REPO_ROOT/logs/chapter-07"
```

Because the repository is outside chroot, capture host-side and chroot-side logs separately.

---

## 7.17.1 Host-side mount verification

Before entering chroot:

```bash
{
  echo "LFS=$LFS"
  findmnt "$LFS/dev"
  findmnt "$LFS/dev/pts"
  findmnt "$LFS/proc"
  findmnt "$LFS/sys"
  findmnt "$LFS/run"
  findmnt "$LFS/dev/shm" || ls -ld "$LFS/dev/shm"
} | tee "$REPO_ROOT/logs/chapter-07/virtual-filesystems.txt"
```

---

## 7.17.2 Chroot-side account verification

Inside chroot:

```bash
{
  echo "=== /etc/passwd ==="
  cat /etc/passwd

  echo
  echo "=== /etc/group ==="
  cat /etc/group

  echo
  echo "=== tester ==="
  tail -n 1 /etc/passwd
  tail -n 1 /etc/group
  ls -ld /home/tester
} > /sources/chapter-07-accounts.txt
```

After leaving chroot, copy:

```bash
cp -v "$LFS/sources/chapter-07-accounts.txt" \
      "$REPO_ROOT/logs/chapter-07/"
```

---

## 7.17.3 Chroot-side tool verification

Inside chroot:

```bash
{
  ls -l /usr/bin/{msgfmt,msgmerge,xgettext}

  bison --version | head -n 1

  perl -v | head -n 2

  python3 --version

  makeinfo --version | head -n 1

  ls -l /usr/bin/{lsblk,uuidgen}

  ls -l /usr/sbin/{blkid,fdisk}

  test ! -e /tools && echo "/tools removed"

  echo
  echo "Remaining .la files:"
  find /usr/{lib,libexec} -name \*.la
} > /sources/chapter-07-temporary-tools.txt
```

After leaving chroot, copy:

```bash
cp -v "$LFS/sources/chapter-07-temporary-tools.txt" \
      "$REPO_ROOT/logs/chapter-07/"
```

---

# Chapter 7 approval checklist

Before moving to Chapter 8 documentation:

```text
[ ] Review all commands in this file
[ ] Confirm ownership transferred from lfs to root
[ ] Confirm virtual kernel filesystems are mounted
[ ] Confirm chroot command
[ ] Confirm full directory hierarchy
[ ] Confirm /usr/lib64 does not exist
[ ] Confirm /etc/mtab -> /proc/self/mounts
[ ] Confirm /etc/hosts
[ ] Confirm /etc/passwd
[ ] Confirm /etc/group
[ ] Confirm tester account and /home/tester
[ ] Confirm prompt changed from "I have no name!" to root
[ ] Confirm login log files and permissions
[ ] Confirm Gettext temporary tools
[ ] Confirm Bison 3.8.2
[ ] Confirm Perl 5.42.0
[ ] Confirm Python 3.14.3
[ ] Confirm temporary Python SSL warning is documented
[ ] Confirm Texinfo 7.2
[ ] Confirm Util-linux 2.41.3
[ ] Confirm temporary documentation removed
[ ] Confirm .la files removed
[ ] Confirm /tools removed
[ ] Create VirtualBox snapshot or official archive backup
[ ] Capture logs/chapter-07 evidence
[ ] Do not commit or push until explicit approval
```

---

## Suggested repository placement

Place this reviewed file at:

```text
docs/chapter-07-entering-chroot-and-temporary-tools.md
```

Suggested companion evidence files:

```text
logs/chapter-07/virtual-filesystems.txt
logs/chapter-07/chapter-07-accounts.txt
logs/chapter-07/chapter-07-temporary-tools.txt
```

---

## Current approval state

```text
DRAFT — CHAPTER 7 ONLY
DO NOT COMMIT
DO NOT PUSH
WAITING FOR REVIEW AND EVIDENCE CAPTURE
```
