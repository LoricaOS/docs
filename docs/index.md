# LoricaOS

A from-scratch, capability-secure, POSIX-compatible operating system for **x86-64 and arm64**, built on the **Aegis** kernel.

LoricaOS is not a Linux distribution and shares no code with one. The kernel, the userland, the desktop, and the package manager were all written from zero around a single idea: **no ambient authority**.

## The security thesis

No process — including `uid 0` — holds power it was not explicitly granted. There is no "root." `uid 0` is just the first uid handed out; it carries no special privilege of its own.

Authority comes only from **unforgeable capability tokens**, validated at the syscall boundary against each process's capability table. A program that was never granted a capability cannot use it, no matter who runs it or what uid it has. When the kernel is unsure, it fails closed.

The capability model carries all the way up the stack: a userland program touches the system only through the tokens declared for it in `/etc/aegis/caps.d/<exec>`. A pure GUI client gets `service` and nothing else; `NET_SOCKET`, `admin`, `SETUID`, and friends are granted one at a time, only where genuinely needed.

## What's here

- **A from-scratch kernel — Aegis.** Clean-slate, for **x86-64 and arm64**: its own boot path, SMP, paging, scheduler, VFS (ext2, ramfs, procfs, memfd), networking (IP/TCP/UDP, sockets), and drivers (NVMe, xHCI, PS/2, framebuffer, HDA). Written **entirely in C**. On arm64 it runs on QEMU virt and boots **natively on the Raspberry Pi 5**. Aegis ships as a versioned `aegis.elf` artifact and keeps its own name and version.
- **A real desktop.** The Lumen compositor with the Glyph toolkit, the Citadel shell, and the Bastion display manager — plus apps, each living in its own repo and shipped as a signed package.
- **A signed package manager — herald.** Components are distributed as `.hpkg` packages; the desktop image is assembled from them, and more are installable on demand.
- **Two profiles, two architectures.** A graphical desktop ISO and a zero-graphical text-console server ISO, both live and bootable on x86-64 — and an arm64 build that runs on QEMU virt and boots natively on the Raspberry Pi 5.

!!! warning "Maturity: this is a v1, not a fortress"

    This is the first public release of LoricaOS. The security *model* is the
    point of the project, and it is enforced — but enforcing a good model is not
    the same as being production-hardened.

    **The C kernel almost certainly contains real, exploitable vulnerabilities.**
    Memory-safety bugs, missed edge cases at the syscall boundary, and driver
    flaws are all plausible. Do not run anything you cannot afford to lose, do not
    expose it to untrusted networks, and do not trust it with secrets. This
    honesty is part of the project: we would rather tell you where the edges are
    than pretend they aren't there.

## Start here

- [Getting Started](getting-started.md) — boot the live ISO and find your way around.
- [Security Model](security-model.md) — capabilities, the no-ambient-authority rule, and why there is no root.
- [Architecture](architecture.md) — how the kernel, userland, and desktop fit together.
- [Desktop](desktop.md) — Lumen, Glyph, Citadel, and Bastion.
- [Packages](packages.md) — herald and the `.hpkg` format.
- [Building](building.md) — build the ISOs from source.
- [Kernel](kernel.md) — Aegis internals and the syscall surface.
- [Reference](reference.md) — syscalls, capability tokens, and config paths.
