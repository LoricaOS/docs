# The Aegis Kernel

Aegis is the from-scratch kernel LoricaOS is built on, targeting **x86-64 and arm64**. It is a **clean-slate, capability-based, POSIX-compatible** kernel with **no Linux or BSD lineage** — its own boot path, paging, scheduler, VFS, network stack, and drivers, all written from zero. It is written **entirely in C**.

Aegis kept its own name through the LoricaOS rebrand on purpose. It is **just the kernel** — it embeds no userland — and it is meant to be **independently adoptable**: an operating system other than LoricaOS can take the `aegis.elf` artifact and build a system on top of it. This page is for the kernel-curious and for would-be adopters.

!!! info "Where this kernel lives"

    Aegis is its own repository: **[github.com/LoricaOS/Aegis](https://github.com/LoricaOS/Aegis)**.
    LoricaOS consumes a released `aegis.elf` rather than building it in-tree —
    kernel and OS versions are independent by design.

## What Aegis is

- **Clean-slate.** No code shared with Linux, BSD, or any other kernel. The boot path, the SMP bring-up, the paging code, the VFS, the TCP/IP stack, and the drivers were all written from scratch.
- **Two architectures.** Aegis targets **x86-64** and **arm64/aarch64** from one source tree. The arm64 port runs on QEMU virt (UEFI/Limine) and boots **natively on the Raspberry Pi 5** (BCM2712) — NVMe, RP1 Gigabit Ethernet, USB HID, the VideoCore framebuffer, SoC thermal/fan control, and 4-core SMP all work on real hardware.
- **Capability-based.** There is no ambient authority. Authority comes only from unforgeable capability tokens validated at the syscall boundary; `uid 0` is not special (see [The Security Model](security-model.md)).
- **POSIX-compatible.** The syscall surface is POSIX-ish — processes, fork/exec, signals, file descriptors, pipes, sockets, `futex`, `epoll` — enough that ordinary Unix userland ports run on it.
- **Freestanding C.** The kernel is freestanding C end to end (`-ffreestanding -nostdlib`, no SSE, kernel code model). The capability subsystem (`kernel/cap`) was originally a `no_std` Rust static library; it was migrated to C so the kernel builds with a C cross-compiler alone — a prerequisite for LoricaOS self-hosting.
- **No userland.** Aegis ships exactly one thing: the kernel. It contains no init, no shell, no libc. An OS built on Aegis supplies all of that.

## The boot model

On **x86-64**, Aegis boots via the **Limine boot protocol** (it also still supports **multiboot2**). The bootloader loads the kernel and hands it a boot-info structure; Aegis parses the memory map and command line from it and takes a 32-bpp framebuffer at the native resolution.

On **arm64**, the same kernel boots two ways: via **Limine (UEFI)** on QEMU virt and Apple-silicon VMs, and via a **native, firmware-direct path** on the Raspberry Pi 5 — no bootloader, entered straight from the stock Pi firmware (TF-A at EL3), with the memory map, GIC bases, PCIe ECAM, and the initramfs all read from the **device tree** at runtime.

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

`kernel/syscall/` is the trust boundary where this is enforced, and `kernel/cap/` (plain C) is where capability tokens are defined and checked. The full model — token kinds, how grants are declared, why there is no root — is in [The Security Model](security-model.md).

## Source layout

A short tour of `kernel/`:

| Directory | What's there |
|-----------|--------------|
| `kernel/arch/x86_64/` | The x86-64 machine: Limine/multiboot2 boot + higher-half trampoline, GDT/IDT/TSS, ISRs, context switch, LAPIC/IOAPIC, SMP bring-up, paging, ACPI, the SYSCALL entry path. |
| `kernel/arch/arm64/` | The arm64 machine: EL2→EL1 boot (Limine and the native Pi 5 firmware-direct path), exception vectors, context switch, GICv2/v3, the generic timer, page tables, PSCI SMP bring-up, a device-tree reader, and the Pi 5 drivers (RP1 southbridge, Broadcom PCIe, VideoCore mailbox). |
| `kernel/syscall/` | **The trust boundary.** POSIX-ish syscall dispatch (`sys_*.c`) — process, memory, file, dir, socket, signal, time, identity, capability, and admin-config syscalls; `futex`. |
| `kernel/mm/` | Physical (`pmm`) and virtual (`vmm`) memory, VMAs, and **uaccess**: `copy_to_user`/`copy_from_user` guarded by an **exception table** so a bad user pointer faults into a recovery path instead of taking down the kernel. |
| `kernel/cap/` | The capability subsystem — the token kinds and the `cap_check` enforcement (`cap.c`, `cap_policy.c`). Plain C, migrated from an earlier `no_std` Rust core. |
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

- an **`x86_64-elf`** cross toolchain — `x86_64-elf-gcc`, `ld`, `objcopy`, `nm` (or an **`aarch64-linux-gnu-*`** toolchain for the arm64 build)
- **`nasm`** (the x86 assembly entry/trampoline/ISR stubs)
- **`xorriso`** plus the pinned Limine (`tools/fetch-limine.sh`) — only for the smoke-test ISO

No Rust toolchain is required — the kernel builds with a C cross-compiler alone.

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

The arm64 build is a parallel Makefile: `make -f Makefile.arm64` produces the aarch64 `aegis.elf` and `make -f Makefile.arm64 test` runs the same capability/SMP smoke test under `qemu-system-aarch64`. A separate `Makefile.pi5native` builds the firmware-direct image for booting on a real Raspberry Pi 5.

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
2. Provide a boot path — a Limine or multiboot2 bootloader on x86-64, Limine or a firmware-direct entry on arm64 — and an ext2 root filesystem.
3. Ship your own PID 1 at `/bin/vigil`, plus whatever userland, libc, and capability-grant policy (`/etc/aegis/...`) your system needs.

Because the kernel embeds no userland and keeps its own versioning, it slots in as a dependency rather than a fork. The one thing not to do is weaken the security model: the no-ambient-authority rule and capability checks at the syscall boundary are the reason the kernel exists.

!!! warning "Maturity: this is a v1 C kernel"

    The security *model* is enforced, but enforcing a good model is not the same
    as being production-hardened. **This C kernel almost certainly contains real,
    exploitable bugs** — memory-safety errors, missed edge cases at the syscall
    boundary, driver flaws. Don't expose it to untrusted input or networks, and
    don't trust it with anything you can't afford to lose. We would rather tell
    you where the edges are than pretend they aren't there.
