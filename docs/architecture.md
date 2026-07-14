# Architecture

LoricaOS is not one program. It is a **fleet of independently versioned repositories** — a kernel, a compositor, a toolkit, a shell, a package manager, and a set of leaf applications — each built, released, and signed on its own cadence, then assembled into a bootable image.

The organizing principle is the same one that governs the security model: **nothing is implicit.** Just as a process holds no authority it was not granted, the OS image embeds no component it did not explicitly fetch and pin. The repository that produces the ISO (`LoricaOS/LoricaOS`) builds almost none of what ends up on it. It **fetches** the kernel, the GUI components, the toolkit, and the base utilities as versioned, signed artifacts, and assembles them.

This is exactly how a Linux distribution consumes a kernel: the distro does not compile Linus's tree as part of its image build — it pulls a released kernel artifact and packages a userland around it. LoricaOS applies that model to the *whole* graphical stack.

## The layered picture

From the metal up:

| Layer | Component | Repo | Role |
|-------|-----------|------|------|
| Kernel | **Aegis** | `LoricaOS/Aegis` | The from-scratch x86-64 kernel: capability model, scheduler, memory (PMM/VMM), VFS, sockets, drivers. Ships as a standalone `aegis.elf` artifact. |
| Base userland | **coreutils** | `LoricaOS/coreutils` | The unprivileged base utilities (`/bin`). Present in both profiles, graphical or not. |
| Display server | **Lumen** | `LoricaOS/lumen` | The compositor / display server. Owns the framebuffer; every GUI app is one of its clients over the `/run/lumen.sock` window protocol. |
| Toolkit | **glyph** | `LoricaOS/glyph` | The GUI toolkit components *link against* — software renderer, theme system, fonts, the Lumen client protocol, terminal core, auth helpers. A build-time dependency, not a runtime package. |
| Shell | **bastion** | `LoricaOS/bastion` | The greeter / display manager — login, unlock, session start. |
| Shell | **citadel-dock** | `LoricaOS/citadel-dock` | The desktop dock. |
| Apps | **lumen-\*** | `LoricaOS/lumen-*` | The applications — terminal, files, editor, settings, calculator, and more. Each in its own repo. |
| Packaging | **herald + Chancery** | `LoricaOS/herald` | The package manager (`herald`) and the signed package repository (Chancery). The `.hpkg` format with an ECDSA-P256 trust chain. |

!!! info "Aegis is the kernel; LoricaOS is the OS"

    "Aegis" names **only** the kernel — the `aegis.elf` artifact, the
    `/etc/aegis/caps.d/*` capability paths, "the Aegis kernel." Everything
    user-facing — installers, banners, window titles, ISO filenames — is
    **LoricaOS**. The two version independently and are released from separate
    repositories.

### Aegis — the kernel

Aegis is a standalone, clean-slate x86-64 kernel and nothing else. It **embeds no userland**: at boot it mounts the root filesystem the bootloader provides and execs `/bin/vigil` as init, panicking "no init found" if there is none — exactly like Linux. Its surface is a small, deliberate one: a capability-checked syscall boundary, an SMP scheduler, paging and a physical/virtual memory manager, a VFS (ext2, ramfs, procfs, memfd), an IP/TCP/UDP network stack with sockets, and drivers (NVMe, xHCI, PS/2, framebuffer, HDA). The capability subsystem is written in `no_std` Rust; the rest is C.

Crucially, Aegis is **independently adoptable**. It publishes a versioned `aegis.elf` that *an* OS consumes — LoricaOS is the reference OS, but it is not a privileged one. Another operating system could build its own userland on Aegis the same way LoricaOS does: fetch the artifact, provide a rootfs with an init at `/bin/vigil`, and boot. The kernel knows nothing about Lumen, herald, or the desktop.

### Lumen, glyph, and the desktop shell

**Lumen** is the compositor and display server. It owns the framebuffer; every graphical program is a *client* that connects to `/run/lumen.sock`, is handed a shared-memory pixel buffer over `SCM_RIGHTS`, draws into a private back buffer, and presents. There are no in-process built-in apps masquerading as part of the compositor — even the calculator is an external client.

**glyph** is the GUI toolkit every graphical component links against: software-framebuffer drawing primitives, the runtime theme/accent system, TTF and bitmap text, image decoding, the *client side* of the Lumen window protocol, a reusable terminal-emulator core, and credential helpers. It is a **build-time dependency only** — four static archives (`libglyph`, `libcitadel`, `libaudio`, `libauth`), no shared object, nothing installed at runtime. Like the kernel, glyph publishes a versioned artifact (`glyph-<version>.tar.gz`) that consumers fetch and link, rather than rebuilding from source.

**bastion** (the greeter) and **citadel-dock** (the dock) are the shell chrome around the apps — login/unlock and the launcher/dock — each its own repo and its own package.

### The lumen-\* applications

Each desktop application lives in its **own `LoricaOS/lumen-*` repository** and publishes a `class=system` herald package. They are leaves of the graphical stack: a `lumen-*` app fetches a pinned glyph toolkit, compiles against it, runs as an external Lumen client, and ships as a signed `.hpkg`. The default set includes the terminal, file manager, editor, settings, calculator, run dialog, applications menu, system monitor, network manager, and more.

!!! info "Why `class=system`"

    A first-party component installs across multiple trees — its binary under
    `/apps/<id>/`, its bundle descriptor (`app.ini`), **and** its capability
    policy under `/etc/aegis/caps.d/`. That cross-tree, signature-trusted,
    installed-verbatim shape is exactly what `class=system` denotes, versus an
    ordinary single-prefix package.

### coreutils

**coreutils** is the unprivileged base userland — the `/bin` utilities that exist on every LoricaOS system, graphical or not. It is built by its own repo's CI and consumed as a pinned, signature-trusted `coreutils.hpkg` artifact. Nothing in it requires elevated capability; it is the plain, everyday command set the shared base rootfs is built from.

### herald + Chancery

**herald** is the package manager and **Chancery** is the signed repository it pulls from. Packages are `.hpkg` files — manifest-first, uncompressed POSIX `ustar` archives — carrying a detached ECDSA-P256/SHA-256 signature. Every component above (except the in-tree installer and the kernel) is delivered as a herald package. The desktop image is *assembled from* these packages with the herald installed-package database pre-seeded, and additional packages are installable on demand from Chancery once a system is running.

## The fetched-artifact model

The `LoricaOS/LoricaOS` repository does not build the kernel and does not build the GUI apps. It **fetches them as versioned artifacts and assembles ISOs.** The in-tree source is small and deliberate: the init (`user/bin/vigil`), the installers, some coreutils/net tooling, and the rootfs skeletons. Everything else arrives pre-built.

Four fetch scripts do the work, each pinning a version and caching the artifact under `vendor/` so an offline build is possible by dropping the file in place:

| Script | Pins | Fetches | Into |
|--------|------|---------|------|
| `tools/fetch-kernel.sh` | `KERNEL_VERSION` | `aegis.elf` from the `LoricaOS/Aegis` release | `vendor/aegis-<ver>.elf` |
| `tools/fetch-components.sh` | `tools/components.list` | each component `.hpkg` from its own `LoricaOS/<id>` release | merged into `build/desktop-overlay/` |
| `tools/fetch-coreutils.sh` | `COREUTILS_VERSION` | `coreutils.hpkg` | `vendor/coreutils/bin/` |
| `tools/fetch-glyph.sh` *(per component)* | each app's `GLYPH_VERSION` | the glyph toolkit tarball | the component's build tree |

The pattern is uniform. `fetch-kernel.sh` resolves a local cache first, otherwise downloads `…/LoricaOS/Aegis/releases/download/v<version>/aegis.elf`, and records the version it landed. `fetch-components.sh` walks `components.list` (one `<herald-id> <version> <fingerprint>` line per package — the fingerprint is a content hash of the extracted `.hpkg` payload, not a plain `sha256sum` of the archive; see `tools/relock-components.sh`, the only supported way to update it), downloads each from `…/LoricaOS/<id>/releases/download/v<version>/<id>.hpkg`, and unpacks every payload into a single overlay tree plus a database the desktop image pre-seeds. `fetch-coreutils.sh` does the same for the base utilities. Note that glyph is fetched *one layer down* — not by the OS repo, but by each `lumen-*` component at its own build time, because the toolkit is a link-time dependency of the app, not a thing installed onto the image.

```text
LoricaOS/Aegis  ──(release: aegis.elf)──────────────┐
LoricaOS/coreutils ──(release: coreutils.hpkg)──────┤
LoricaOS/lumen ─┐                                    │
LoricaOS/bastion │                                   ▼
LoricaOS/citadel-dock ├─(release: <id>.hpkg)─►  LoricaOS/LoricaOS
LoricaOS/lumen-* ─┘                              fetch + assemble
        ▲                                             │
        └── each fetches glyph toolkit at build time  ▼
                                              loricaos-*.iso
```

!!! info "Why fetch instead of build"

    The same three reasons hold at every layer — kernel, toolkit, components:

    - **Reproducibility** — everything that pins `glyph 1.0.0` (or kernel
      `1.x`, or any component version) gets bit-for-bit the same artifact,
      regardless of when or where the image is built.
    - **Decoupling** — the OS build needs no kernel cross-toolchain and no musl
      configuration for every app; it pulls finished, signed binaries.
    - **Independent cadence** — each piece releases on its own schedule. A glyph
      change never silently alters a component until that component bumps its
      pin; a kernel release never forces an OS release. Versions move
      independently, and the fetch scripts are the single points that couple
      them.

    This independence has a naming trap worth knowing about: the Aegis kernel
    repo, every component repo (lumen, bastion, …), and the LoricaOS repo
    itself each define a build-time macro called `AEGIS_VERSION` — but it means
    a *different* version in each one (the kernel's own version; that
    component's own version; and, in the LoricaOS repo specifically, this
    OS's own release, exported as `LORICA_VERSION` to avoid the collision).
    A GUI app reading its own compiled-in `AEGIS_VERSION` to show "the OS
    version" is reading the wrong thing — the actual OS release is exposed at
    runtime via `/etc/lorica-version`, stamped from the top-level `VERSION`
    file at rootfs-build time. This is exactly the bug that shipped in
    Lumen's About window until it was caught.

### Adding an app to the default image

Because the component set is just a list, the default desktop image is edited by editing data, not code. Add a `<herald-id> <version>` line to `tools/components.list`, place the built `.hpkg` at `vendor/components/<id>-<version>.hpkg`, and run `tools/relock-components.sh` to fill in the fingerprint — then the next build fetches and bakes it in. Apps *not* on the list are still installable on demand via herald once they are on Chancery — the mechanism for optional extras (games, niche tools) that should not bloat the base image.

## The two profiles

LoricaOS builds two production ISOs from a shared base:

| Profile | Target | Graphical stack | Rootfs |
|---------|--------|-----------------|--------|
| **Desktop** | `make desktop-iso` → `loricaos-desktop.iso` | Full Lumen desktop — compositor, shell, apps | `rootfs/` + `rootfs-desktop/` overlay + fetched components |
| **Server** | `make server-iso` → `loricaos-server.iso` | **None** — text console, login at `/bin/login` | `rootfs/` only |

`rootfs/` is the shared base both profiles are built from; `rootfs-desktop/` overlays the graphical profile on top, and the fetched component overlay merges in. The server profile is genuinely zero-graphical — it pulls the kernel and coreutils but none of the `lumen-*` components — so it boots straight to a text console.

The capability model is what makes this layering safe rather than merely tidy: a component arriving as a fetched, signed package brings its own `caps.d` policy, so it gains exactly the authority it declares and not one token more — whether it was baked into the image or installed later from Chancery.
