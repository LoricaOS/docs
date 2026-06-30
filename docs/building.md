# Building from Source

This page covers building **LoricaOS** — the userland, root filesystem, and
bootable ISOs — from source. LoricaOS does **not** build the Aegis kernel or the
graphical applications here; it fetches them as pinned, versioned artifacts and
assembles them into images. That model is the key thing to understand before you
build, so it comes first.

## The fetched-artifact model

LoricaOS is an *assembler*, not a from-scratch build of the whole system. The
heavy pieces are produced by their own repositories, published as release
artifacts, and pulled in at build time — each pinned to an exact version and
cached under `vendor/` after the first download.

| Artifact | Source repo | Pin file | Fetched by |
| --- | --- | --- | --- |
| Aegis kernel (`aegis.elf`) | `LoricaOS/Aegis` | `KERNEL_VERSION` | `tools/fetch-kernel.sh` |
| Desktop GUI components (`.hpkg`) | `LoricaOS/lumen-*`, `LoricaOS/bastion`, … | `tools/components.list` | `tools/fetch-components.sh` |
| Coreutils (`coreutils.hpkg`) | `LoricaOS/coreutils` | `COREUTILS_VERSION` | `tools/fetch-coreutils.sh` |
| Limine bootloader | `limine-bootloader/limine` | `tools/limine/VERSION` | `tools/fetch-limine.sh` |

What that means in practice:

- **The kernel is consumed, not compiled.** `fetch-kernel.sh` resolves
  `vendor/aegis-<KERNEL_VERSION>.elf` from the local cache, or downloads it from
  the `LoricaOS/Aegis` release for that version. The OS version (`VERSION`) and
  the kernel version (`KERNEL_VERSION`) move independently — this fetch is the
  single point that couples them.
- **The GUI stack is consumed, not compiled.** The Lumen compositor, Bastion
  display manager, Citadel dock, and every `lumen-*` application are released as
  herald `.hpkg` packages by their own repos. `fetch-components.sh` downloads
  each one listed in `tools/components.list`, unpacks the payloads into a single
  overlay tree, and emits the herald package database the desktop image
  pre-seeds.
- **Coreutils and the bootloader are likewise fetched.** The unprivileged
  coreutils (`ls`, `cat`, `cp`, `grep`, `ps`, `find`, …) come from the
  `LoricaOS/coreutils` package; Limine's BIOS+UEFI binaries are fetched (and its
  host deploy tool built) from the pinned tag.

!!! note "Building offline"
    Every fetch checks a local cache before reaching the network. To build with
    no network access, pre-populate the caches: drop the kernel at
    `vendor/aegis-<KERNEL_VERSION>.elf`, the component packages at
    `vendor/components/<id>-<version>.hpkg`, the coreutils package at
    `vendor/coreutils/coreutils-<COREUTILS_VERSION>.hpkg`, and the Limine
    checkout at `vendor/limine-<tag>/`.

What *is* built in this repo: the trusted base userland — init (`vigil`), the
shell, login/auth, the installers (`installer` + the graphical `gui-installer`),
system-control and networking tools, the herald package manager, and the
in-tree test/stress suite — plus the rootfs skeletons (`rootfs/`,
`rootfs-desktop/`) and the final ISO construction.

## Prerequisites

The build must run on a **Linux x86_64 host**. It compiles the userland against
**musl**: `tools/build-musl.sh` downloads musl (1.2.5) and builds it as a shared
library, producing a `musl-gcc` wrapper around the host C compiler that links
every in-tree program dynamically. There is no separate cross-compiler to
install — but you do need a working host toolchain for it to wrap.

Required on the host:

- **C toolchain** — `gcc` and `binutils` (the build also uses `strip` /
  `x86_64-linux-gnu-strip` to strip binaries at packaging time).
- **`make`, `bash`, `python3`** — the driver, the tool scripts, and the rootfs
  truncation step.
- **`curl` and `git`** — `curl` fetches the kernel, components, and coreutils;
  `git` clones the pinned Limine release.
- **`xorriso`** — builds the hybrid BIOS+UEFI ISO.
- **`mtools`** (`mmd`, `mcopy`) and **`dosfstools`** (`mkfs.fat`) — build the FAT
  ESP image embedded in the ISO and written to disk by the installer.
- **`e2fsprogs`** (`mke2fs`, `debugfs`, `e2fsck`, `resize2fs`) — create and
  populate the ext2 root filesystem images.
- **`sha256sum`** (GNU coreutils) — records component hashes into the herald db.

## Build commands

The Makefile produces two production ISOs. Default `make` builds the desktop
image.

```bash
make desktop-iso   # -> build/loricaos-desktop.iso
make server-iso    # -> build/loricaos-server.iso
make iso           # both of the above
make               # alias for the default (desktop) ISO
```

Both outputs are live, bootable images that can also install themselves to disk.
The first build downloads the fetched artifacts into `vendor/`; later builds
reuse the cache.

Other useful targets:

```bash
make version   # prints: LoricaOS <ver> (kernel <ver>)
make test      # builds all ISOs and boots them headless (see below)
make clean     # removes build outputs (ISOs, rootfs/ESP images, staging dirs)
```

`make test` builds the desktop, server, and a self-test ISO, then boots each
under QEMU: the desktop image must reach the Bastion greeter
(`[BASTION] greeter ready`), the server image must reach a text login, and the
self-test image must report `[CAPTEST] ALL PASS` — proving the kernel, userland,
display stack, and capability model all come up.

## The two profiles

LoricaOS builds two profiles from one source tree. They differ in which rootfs
layers compose the image:

| | **Desktop** | **Server** |
| --- | --- | --- |
| Output | `build/loricaos-desktop.iso` | `build/loricaos-server.iso` |
| Rootfs | `rootfs/` base **+** `rootfs-desktop/` overlay | `rootfs/` base only |
| Graphical stack | Full (Lumen / Bastion / Citadel / `lumen-*` apps) | None |
| Fonts / wallpaper | Yes | None (uses the kernel console font) |
| herald db | Pre-seeded with the desktop components | Empty |
| Boots to | Bastion graphical greeter | Text console login |

The **server rootfs** is built straight from `rootfs.manifest` and the `rootfs/`
skeleton (`AEGIS_PROFILE=server tools/build-rootfs.sh`): a pure text system with
no compositor, fonts, or graphical binaries.

The **desktop rootfs** is *assembled* on top of the server base by
`tools/assemble-desktop-rootfs.sh`: it grows the server image, lays down the
fetched component overlay, adds the in-tree `gui-installer` (the live-only
graphical installer, auto-removed on installed systems), and writes the
pre-seeded herald package database. It is assembled from fetched packages, not
compiled from graphical source — which is why there is no desktop manifest.

Each ISO pairs its rootfs with a matching FAT ESP image carrying Limine's UEFI
binaries, a profile-specific `limine.conf`, and a copy of the kernel; `xorriso`
then stages the kernel, rootfs, ESP, and Limine boot files into a hybrid
BIOS+UEFI ISO.

## Building a component, or the kernel

If you need to change something that LoricaOS fetches rather than builds, you
build it in its own repository and publish (or locally cache) the resulting
artifact:

- **A GUI component** — each desktop app lives in its own `LoricaOS/lumen-*`
  repo (plus `bastion`, `citadel-dock`, and the Lumen compositor) and publishes a
  `class=system` herald `.hpkg`. Build it there, then either bump its line in
  `tools/components.list` to the new release version or drop the built `.hpkg`
  into `vendor/components/` so the desktop assembly picks it up. To add a *new*
  app to the default desktop image, add a `<herald-id> <version>` line to
  `components.list`; apps not on that list remain installable on demand via
  herald.
- **The kernel** — Aegis builds in the [`LoricaOS/Aegis`](https://github.com/LoricaOS/Aegis)
  repository and is released as `aegis.elf`. To build LoricaOS against a kernel
  you built yourself, place it at `vendor/aegis-<KERNEL_VERSION>.elf` (or bump
  `KERNEL_VERSION` to a published release). See the
  [Aegis Kernel](kernel.md) page for the kernel build itself.
