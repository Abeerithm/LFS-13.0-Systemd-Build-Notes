# Stage 03 — Final Preparations

## Goal

Prepare the isolated LFS build environment before compiling the cross-toolchain.

This stage creates the required directory structure, configures the dedicated `lfs` user, and defines a controlled shell environment.

## LFS Mount Point

The main LFS path was defined as:

```bash
export LFS=/mnt/lfs
```

Verify it:

```bash
echo $LFS
```

Expected result:

```text
/mnt/lfs
```

## Create the Limited Directory Layout

Create the essential directory structure inside the LFS filesystem:

```bash
mkdir -pv $LFS/{etc,var} $LFS/usr/{bin,lib,sbin}
```

Create symbolic links for the top-level directories:

```bash
for i in bin lib sbin; do
  ln -sv usr/$i $LFS/$i
done
```

For a 64-bit system, create the `lib64` directory:

```bash
case $(uname -m) in
  x86_64) mkdir -pv $LFS/lib64 ;;
esac
```

Create the tools directory:

```bash
mkdir -pv $LFS/tools
```

## Create the Dedicated LFS User

A dedicated user named `lfs` was created to isolate the temporary toolchain build from the Debian host system.

Create the group:

```bash
groupadd lfs
```

Create the user:

```bash
useradd -s /bin/bash -g lfs -m -k /dev/null lfs
```

Set a password:

```bash
passwd lfs
```

Grant ownership of the required LFS directories:

```bash
chown -v lfs $LFS/{usr{,/*},lib,var,etc,bin,sbin,tools}
```

For a 64-bit system:

```bash
case $(uname -m) in
  x86_64) chown -v lfs $LFS/lib64 ;;
esac
```

Switch to the build user:

```bash
su - lfs
```

## Configure the LFS User Environment

The shell environment for the `lfs` user was configured to avoid contamination from the host system.

Create `.bash_profile`:

```bash
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
```

Create `.bashrc`:

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

Load the new environment:

```bash
source ~/.bash_profile
```

## Validation

Verify the most important environment variables:

```bash
echo $LFS
echo $LFS_TGT
echo $PATH
echo $LC_ALL
```

Expected key values:

```text
LFS=/mnt/lfs
LC_ALL=POSIX
```

The target triplet should resemble:

```text
x86_64-lfs-linux-gnu
```

## Why This Stage Matters

The temporary toolchain must be built in a controlled environment.

The dedicated `lfs` user and restricted shell configuration reduce the risk of accidentally using host-system libraries or tools during the cross-toolchain build.

## Personal Notes

* Work started first under the `lfs` user.
* Administrative preparation was completed using `root`.
* Later stages moved into the LFS `chroot` environment.
* The main LFS path before entering `chroot` remained `/mnt/lfs`.

## Reminder

These notes reflect a practical LFS 13.0 Systemd build. Always verify commands against the official book before reuse.
