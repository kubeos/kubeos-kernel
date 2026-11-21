# kubeos-kernel

The **kernel a KubeOS node boots**: `linux-image-kubeos` is a one-line flavour meta that names
exactly one kernel package and nothing else. Source package `kubeos-kernel`, binary package
`linux-image-kubeos` (source and binary names must differ here — see "Why this name" below).

## What a KubeOS node runs

| | |
|---|---|
| Kernel | `linux-image-6.12.117-kubeos` `6.12.117-1kubeos1` |
| `uname -r` | `6.12.117-kubeos` (`CONFIG_LOCALVERSION="-kubeos"`) |
| Where from | built by KubeOS from the upstream 6.12.117 tree with `make deb-pkg`, published to https://repo.kubeos.in |
| What it replaces | Debian's `linux-image-amd64` → `linux-image-6.12.117+deb13-amd64` from the trixie archive |
| Pin record | [`KERNEL`](KERNEL) — package, version and sha256 of the .deb |

`apt upgrade` can only reach a kernel published here, because Debian's `linux-image-amd64` meta
is not installed on a KubeOS node and this package **conflicts** with it. A kernel change is
therefore a KubeOS decision, taken by publishing a new `.deb` plus a new version of this meta.

## Why this name (`linux-image-kubeos`, not `kubeos-kernel`)

live-build decides the ISO's kernel from two options, and the installed package name is their
concatenation — verified in the installed live-build source, not assumed:

    /usr/lib/live/build/chroot_linux-image:
        echo ${PACKAGE}-${FLAVOUR} >> chroot/root/packages.chroot

and later stages find the kernel by globbing `vmlinuz-*${FLAVOUR}` (`binary_grub_cfg`,
`binary_syslinux`). So with

    --linux-packages "linux-image" --linux-flavours "kubeos"

the build installs `linux-image-kubeos`, and `chroot/boot/vmlinuz-<version>-kubeos` is found.
`--linux-packages none` is **not** an option: `binary_linux-image` exits early in that case and
the ISO would be built without a kernel at all.

## Moving the kernel

1. Build the new kernel `.deb` and copy it to `/root/kernel/` on the build VM.
2. Publish it: the archive publisher accepts `kubeos-*` and `*-kubeos` names, so no foreign-package
   exception is needed.
3. Update [`KERNEL`](KERNEL) and `debian/control`'s `Depends`; set this package's version to the
   kernel version (`<kernel-version>-1kubeos1`), bump `debian/changelog`, build, publish.
4. `auto/config` needs **no** change: it names this meta, never the kernel.

## Known limit: Secure Boot

The kernel is **unsigned**. Debian's is signed and the ISO carries shim, so on a machine with UEFI
Secure Boot enabled GRUB refuses to load it. Secure Boot must be off on a KubeOS node (user
decision 2026-11-22). Signing with a KubeOS key plus MOK enrolment would be a separate piece of
work.
