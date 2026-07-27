# Configuring & Minifying the Kernel

Aegis is a single source tree that spans an enormous range of builds — from a
stripped, MCU-class kernel up to a full-featured x86-64 / arm64 system —
selected entirely by a `.config`, the same way `menuconfig` tailors a Linux
kernel. The goal is deliberately ambitious: **smaller than a Linux `tinyconfig`
at one end, and reaching for the full breadth of a general-purpose kernel at the
other — from one tree, one build system.**

!!! info "Where the configuration engine lives"

    The engine is its own repository: **[github.com/LoricaOS/kconfig](https://github.com/LoricaOS/kconfig)**.
    It is a small, **dependency-free** C program — no Python, no flex/bison, no
    ncurses — so it builds with a plain C compiler and can eventually run on
    LoricaOS itself. The kernel-specific `Kconfig` files, the `defconfig`s, and
    the Makefile hooks live in the [Aegis](https://github.com/LoricaOS/Aegis)
    tree; the tool just consumes them.

## How it works

`kconf` reads a tree of `Kconfig` files (a pragmatic, **bool-only** subset of
the Linux Kconfig language — LoricaOS has no loadable modules, so there is no
tristate), resolves your selection to a `.config`, and generates the two
artifacts the build consumes:

| Artifact | For | Example |
|----------|-----|---------|
| `build/generated/autoconf.h` | C source `#ifdef` guards | `#define CONFIG_NET 1` |
| `build/auto.conf` | Makefile object gating | `CONFIG_NET=y` |

The Makefile `-include`s `auto.conf` to gate which source files compile, and
`-include`s `autoconf.h` into every translation unit so `#ifdef CONFIG_*` works
in the source. Dependency resolution matches Linux semantics where they overlap:
`depends on` gates whether a symbol can be set, `select` forces its target on,
and values settle to a fixpoint.

## Tiers

Three `defconfig`s ship in `configs/`, and you can write your own:

| Tier | For | Includes |
|------|-----|----------|
| **`tiny`** | The minification floor — boots to the smoke-test panic | Security core + boot + mm + sched + a console. Nothing optional. |
| **`workstation`** | A text-only, networked system with minimal bloat | The above **plus** the full network stack, persistent `ext2`, `procfs`, pipes — but no audio, no debug trace, only the drivers a headless text workstation needs. |
| **`full`** | Everything on | Every subsystem and driver. **Identical to the historical bare-`make` build.** |

The `workstation` tier is the feature contract for an eventual **PicoCalc spin**
(see below): everything you want in a pocket text terminal, and nothing you
don't.

## Using it

```sh
make tiny_defconfig          # smallest bootable kernel
make workstation_defconfig   # text-only, networked
make full_defconfig          # everything on (the default)
make allnoconfig             # every optional prompt off
make oldconfig               # keep your .config, fill in new symbols

make                         # builds the selected .config
                             # (a bare make with no .config seeds full_defconfig,
                             #  so the historical everything-on build is unchanged)
```

## The one thing you can never turn off

!!! warning "The security core has no config symbol"

    The capability subsystem (`kernel/cap`), the `cap_check` at the syscall
    boundary, and `uaccess` are **the product**. There is deliberately **no
    `CONFIG_` knob** for them — they stay in the unconditional base build, and a
    link-time check keeps them present. A minified Aegis is still, in full, a
    capability kernel.

    This is the one place LoricaOS departs hard from Linux's `tinyconfig`
    philosophy: here, *"tiny" can never mean "drop the security model."*
    Minification only ever exposes knobs for genuinely optional code.

## `CONFIG_MMU` — the embedded axis

The paging-based memory model (VMM, page tables, per-process address spaces,
demand paging, `copy_to/from_user` with its page-fault exception table) sits
behind **`CONFIG_MMU`**, a first-class symbol (default `y`) from the very first
skeleton. The point is that a **no-MMU target is a config axis, not a fork.**

The long-term aim is a build that fits an **RP2350 (Raspberry Pi Pico 2)** —
520 KB of SRAM, a Cortex-M33 with an **MPU, not an MMU** — so a
[Clockwork PicoCalc](https://www.clockworkpi.com/) could run a real
capability-secure OS: a sibling to LoricaOS, or a dedicated PicoCalc spin. That
needs `CONFIG_MMU=n` to select a flat, MPU-isolated memory backend, a new
`arch/armv8m` port, and PicoCalc drivers (LCD console, matrix keyboard, SD,
Wi-Fi) — all future work. But the elegant part is already true: the capability
model is **software-enforced and MMU-independent**, so the thing that makes
Aegis *Aegis* survives the port. You lose hardware memory *isolation* (a real,
loudly-flagged trade-off); you keep capability *authority*.

On the RP2350's 520 KB SRAM the relevant budget is **`.bss` + `.data`, not
`.text`** — the chip executes code from external QSPI flash via XIP, so code and
rodata live in flash (up to 16 MB) and only static/heap/stack data occupy SRAM.
That is exactly why the arena work matters more than code trimming for that
target: `tiny`'s `.bss` is **already 396 KB — under the RP2350's 520 KB SRAM** —
and it still carries x86-only per-CPU / GDT / TSS structures an armv8-M core
wouldn't have, plus a PMM bitmap that sizes to actual RAM (a fraction on an MCU).
Every knob a constrained target needs now exists; what's left is the arch port
itself.

## What it buys you (measured)

Gated so far — the network stack, the trace ring, the Hyper-V stack, the HDA
driver, and the whole VirtIO family (per device) — across the three tiers on
x86-64:

| tier | `.text` (code) | `.bss` (static RAM) |
|------|----------------|---------------------|
| `full` | 585,886 B | 3,089,924 B |
| `workstation` | 551,182 B | 2,403,396 B |
| `tiny` | 453,910 B (**−22.5 %**) | 396,516 B (**−2.69 MB, −87 %**) |

The `.bss` collapse comes from making the fixed static arenas numeric knobs
(`kconf`'s `int` symbols). At the defaults they cost, per instance/table:

| arena | knob | default | in `tiny` |
|-------|------|---------|-----------|
| tmpfs `/tmp` + `/run` | `RAMFS_MAX_INODES` × `…PAGES_PER_FILE` | ~556 KB each | 32×32 → 13.8 KB each |
| per-CPU (TSS/GDT/areas) | `MAX_CPUS` | 1024 → ~250 KB | 8 → ~2 KB |
| PTY pool | `PTY_MAX_PAIRS` | 16 → ~140 KB | 4 → 35 KB |
| kernel log ring | `KLOG_SIZE_KB` | 64 KB | 8 KB |
| ext2 block cache | `EXT2_CACHE_SLOTS` | 16 → 66 KB | 4 → 16 KB |
| PMM boot bitmap | `PMM_BOOT_GB` | 4 → 128 KB | 2 → 64 KB |

**`tiny`'s 396 KB `.bss` already fits under an RP2350's 520 KB SRAM** — and it
still carries x86-only per-CPU/GDT/TSS structures an armv8-M port wouldn't have.

Every `tiny` build still boots to the no-init smoke panic with the capability
core intact. `CONFIG_NET` was the proving ground — the *most* coupled subsystem;
everything since (the 640 KB trace ring, the Hyper-V stack, the HDA driver, the
VirtIO devices) followed the same mechanism with far less friction. The next
targets — xHCI/USB, VMware, and the static `ramfs` arenas — are the same again.

## Not just tiers — deeply customizable

The tiers are only `defconfig`s; each one just *sets* a collection of
independent `CONFIG_*` symbols. Anything a tier cuts, you can cut or keep on its
own. Families are gated at **per-device** granularity, Linux-style: a
transport/core symbol plus one knob per member that `depends on` it. VirtIO is
the worked example — `CONFIG_VIRTIO` is the transport, and `CONFIG_VIRTIO_BLK`,
`CONFIG_VIRTIO_GPU`, `CONFIG_VIRTIO_NET`, … each stand alone:

```sh
make tiny_defconfig                    # everything off
echo CONFIG_VIRTIO=y     >> .config    # add the transport back…
echo CONFIG_VIRTIO_BLK=y >> .config    # …and *only* virtio-blk
make oldconfig && make                 # compiles virtio_blk + virtio_pci + a
                                       # no-op stub for the other 10 devices
```

So "tiny", "workstation", and "full" are starting points, not a cage — the real
surface is the full `Kconfig` tree, drivable with `make menuconfig` or by hand.

## For adopters: how a subsystem is severed

Gating a subsystem cleanly is a repeatable pattern, proven first on the
hardest-coupled one:

1. **Makefile gate.** `auto.conf`'s `CONFIG_*` variables drop the subsystem's
   source files (and anything that depends on it) from the build.
2. **`sys_ni_syscall`-style stubs.** For the syscalls and cross-boundary symbols
   the core still references, a small stub TU — compiled *only* when the
   subsystem is off — returns `-ENOSYS` / inert values, so the syscall table and
   callers link unchanged (the Linux `ni_syscall` pattern).
3. **`#ifdef CONFIG_*` hooks.** A handful of init / IRQ / poll call-sites are
   guarded directly, so a disabled subsystem is genuinely absent, not merely
   inert.
4. **Occasional extraction.** Where a *general* facility was co-located with a
   subsystem (e.g. `poll`/`select`/`epoll` living inside the socket file), it is
   moved to its own always-compiled unit — cleaner code, and the subsystem file
   then gates without collateral.
