# The Aegis Kernel

Aegis is the from-scratch x86-64 kernel LoricaOS is built on. It is a **clean-slate, capability-based, POSIX-compatible** kernel with **no Linux or BSD lineage** — its own boot path, paging, scheduler, VFS, network stack, and drivers, all written from zero. It is written in C with a small **Rust capability core** (`no_std`).

Aegis kept its own name through the LoricaOS rebrand on purpose. It is **just the kernel** — it embeds no userland — and it is meant to be **independently adoptable**: an operating system other than LoricaOS can take the `aegis.elf` artifact and build a system on top of it. This page is for the kernel-curious and for would-be adopters.

!!! info "Where this kernel lives"

    Aegis is its own repository: **[github.com/LoricaOS/Aegis](https://github.com/LoricaOS/Aegis)**.
    LoricaOS consumes a released `aegis.elf` rather than building it in-tree —
    kernel and OS versions are independent by design.

## What Aegis is

- **Clean-slate.** No code shared with Linux, BSD, or any other kernel. The boot path, the SMP bring-up, the paging code, the VFS, the TCP/IP stack, and the drivers were all written from scratch.
- **Capability-based.** There is no ambient authority. Authority comes only from unforgeable capability tokens validated at the syscall boundary; `uid 0` is not special (see [The Security Model](security-model.md)).
- **POSIX-compatible.** The syscall surface is POSIX-ish — processes, fork/exec, signals, file descriptors, pipes, sockets, `futex`, `epoll` — enough that ordinary Unix userland ports run on it.
- **C with a Rust capability core.** The bulk of the kernel is freestanding C (`-ffreestanding -nostdlib`, no SSE, kernel code model). The capability subsystem (`kernel/cap`) is a `no_std` Rust static library linked into the image.
- **No userland.** Aegis ships exactly one thing: the kernel. It contains no init, no shell, no libc. An OS built on Aegis supplies all of that.

## The boot model

Aegis boots via **multiboot2**. The bootloader (LoricaOS and the in-tree smoke test both use Limine) loads the kernel and hands it the multiboot2 info structure; Aegis parses the memory map and kernel command line from it, and takes its framebuffer through the **multiboot2 framebuffer tag** (the kernel requests a 32-bpp framebuffer at the native resolution and renders into whatever the firmware/bootloader provides).

The kernel does **not** carry a userland. At boot, after subsystem init, it mounts the root filesystem the bootloader/OS provided (an ext2 image — a Limine module on live media, an NVMe/AHCI partition on an installed system) and execs **`/bin/vigil`** as PID 1:

```
proc_spawn_init()  ->  reads /bin/vigil from the root filesystem  ->  enters ring 3
```

If there is no `/bin/vigil` on the root filesystem, the kernel panics with `[INIT] no init found on root filesystem` — exactly the way Linux panics when it cannot find `/sbin/init`. That panic is not a bug; for a kernel-only build it is the *expected* end state (see `make test` below).

!!! info "Why `vigil` and not `init`"

    On LoricaOS, PID 1 is **vigil**. The path `/bin/vigil` is the kernel's
    hard-coded init path. An adopting OS can ship its own PID 1 at that path, or
    patch the path, and otherwise reuse the kernel unchanged.

Each tagged release publishes a stripped, version-named `aegis.elf`. An OS targeting Aegis fetches the kernel image for the version it wants and assembles its own boot image around it.

## The security model at the kernel level

The security model *is* the product, and it lives in the kernel. Two rules:

1. **No ambient authority.** No process — including one running as `uid 0` — holds power it was not explicitly granted. `uid 0` is cosmetic: it is just the first uid handed out and carries no inherent privilege.
2. **Capabilities are validated at the syscall boundary.** Authority comes only from unforgeable capability tokens held in each process's capability table. A privileged operation calls `cap_check` against that table; a process that was never granted the relevant capability gets `ENOCAP`/`EFAULT`, no matter who runs it. When the kernel is unsure, it **fails closed**.

`kernel/syscall/` is the trust boundary where this is enforced, and `kernel/cap/` (the Rust core) is where capability tokens are defined and checked. The full model — token kinds, how grants are declared, why there is no root — is in [The Security Model](security-model.md).

## Source layout

A short tour of `kernel/`:

| Directory | What's there |
|-----------|--------------|
| `kernel/arch/x86_64/` | The machine: multiboot2 boot + higher-half trampoline, GDT/IDT/TSS, ISRs, context switch, LAPIC/IOAPIC, SMP bring-up, paging, ACPI, the SYSCALL entry path. |
| `kernel/syscall/` | **The trust boundary.** POSIX-ish syscall dispatch (`sys_*.c`) — process, memory, file, dir, socket, signal, time, identity, capability, and admin-config syscalls; `futex`. |
| `kernel/mm/` | Physical (`pmm`) and virtual (`vmm`) memory, VMAs, and **uaccess**: `copy_to_user`/`copy_from_user` guarded by an **exception table** so a bad user pointer faults into a recovery path instead of taking down the kernel. |
| `kernel/cap/` | The capability subsystem — `no_std` Rust, built to a static `libcap.a` and linked into the kernel. |
| `kernel/sched/` | Scheduler, run queues, wait queues, and process plumbing (fork/exec live here and in `kernel/proc/`). |
| `kernel/proc/`, `kernel/signal/` | Process model + ELF loading (`elf.c`, the init spawn); POSIX signals. |
| `kernel/fs/` | VFS, ext2 (with a block cache), ramfs, procfs, initrd, pipes, `memfd`, `eventfd`, GPT/block layer. |
| `kernel/net/` | netdev, Ethernet, IP, UDP, TCP, BSD-style sockets, AF_UNIX, `epoll`. |
| `kernel/drivers/` | Storage (**NVMe**, AHCI), USB (**xHCI** + HID), the **framebuffer** console, HDA audio, a full virtio family, several real NICs (e1000, rtl8139/8169, vmxnet3), and Hyper-V/VMBus paravirtual devices. |
| `kernel/tty/` | TTY and PTY. |
| `kernel/core/` | Early init (`main.c`), `printk`, panic, RNG, the in-kernel symbol table, tracing. |

## Building the kernel

### Toolchain

Building Aegis needs a bare-metal cross toolchain and a couple of extras:

- an **`x86_64-elf`** cross toolchain — `x86_64-elf-gcc`, `ld`, `objcopy`, `nm`
- **`nasm`** (the assembly entry/trampoline/ISR stubs)
- **Rust nightly** with the **`x86_64-unknown-none`** target (the capability core)
- **`xorriso`** plus the pinned Limine (`tools/fetch-limine.sh`) — only for the smoke-test ISO

### Targets

The version is a single value in the `VERSION` file (not derived from git), stamped into the kernel so builds are reproducible anywhere.

```sh
make            # build/aegis.elf — the shipped artifact (default target)
make iso        # build/aegis.iso — kernel-only, bootable via Limine
make test       # boot the kernel alone; the "no init found" panic is SUCCESS
make dist       # build/dist/aegis-<VERSION>.elf — the stripped release image
make version    # print the version from VERSION
make clean
```

!!! info "`make test` is a smoke test, and the panic is the pass"

    There is no userland in this repo, so a clean boot has nothing to exec as
    init. `make test` boots two images: a tiny rootfs carrying a freestanding
    **capability test** (it checks that pid/write work *and* that a POWER-gated
    syscall is **denied** to baseline-cap init — i.e. no ambient authority), and
    the kernel-only ISO, where reaching the `[INIT] no init found` panic is the
    success condition. Real, full boots happen in LoricaOS.

The final link is a **two-pass link** that embeds an in-kernel symbol table (so a `[PANIC]` backtrace resolves to `function+offset`); `make sym ADDR=0x...` turns an address back into source:line.

## Adopting Aegis

Aegis is built to underpin an OS that is not LoricaOS. To adopt it you:

1. Take a released `aegis.elf` (or build your own from [github.com/LoricaOS/Aegis](https://github.com/LoricaOS/Aegis)).
2. Provide a bootloader that speaks multiboot2 and an ext2 root filesystem.
3. Ship your own PID 1 at `/bin/vigil`, plus whatever userland, libc, and capability-grant policy (`/etc/aegis/...`) your system needs.

Because the kernel embeds no userland and keeps its own versioning, it slots in as a dependency rather than a fork. The one thing not to do is weaken the security model: the no-ambient-authority rule and capability checks at the syscall boundary are the reason the kernel exists.

!!! warning "Maturity: this is a v1 C kernel"

    The security *model* is enforced, but enforcing a good model is not the same
    as being production-hardened. **This C kernel almost certainly contains real,
    exploitable bugs** — memory-safety errors, missed edge cases at the syscall
    boundary, driver flaws. Don't expose it to untrusted input or networks, and
    don't trust it with anything you can't afford to lose. We would rather tell
    you where the edges are than pretend they aren't there.
