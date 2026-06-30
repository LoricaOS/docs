# Reference

Lookup tables for the Aegis capability model, the `caps.d` policy format, kernel
command-line options, and the ISO profiles. Values are taken from the kernel
source (`kernel/cap/`, `kernel/core/main.c`) and the LoricaOS rootfs.

## Capabilities

Every capability a process can hold is one of the kinds below. Authority is never
ambient: a syscall is allowed only if the calling process holds the matching kind
in its capability table. Kinds are defined in `kernel/cap/cap.h`.

| Kind | # | Purpose |
|------|---|---------|
| `CAP_KIND_NULL` | 0 | Empty-slot sentinel — not a grantable capability. |
| `CAP_KIND_VFS_OPEN` | 1 | Call `sys_open`. |
| `CAP_KIND_VFS_WRITE` | 2 | Call `sys_write`. |
| `CAP_KIND_VFS_READ` | 3 | Call `sys_read`. |
| `CAP_KIND_AUTH` | 4 | Open `/etc/shadow` for reading (password verification). |
| `CAP_KIND_CAP_GRANT` | 5 | Delegate capabilities to child processes (reserved). |
| `CAP_KIND_SETUID` | 6 | Call `sys_setuid` / `sys_setgid`. |
| `CAP_KIND_NET_SOCKET` | 7 | Call `sys_socket` / the socket syscalls. |
| `CAP_KIND_NET_ADMIN` | 8 | Call `sys_netcfg` — set IP / mask / gateway. |
| `CAP_KIND_THREAD_CREATE` | 9 | Call `clone(CLONE_VM)` (create threads). |
| `CAP_KIND_PROC_READ` | 10 | Read `/proc/[other-pid]` (another process's procfs). |
| `CAP_KIND_DISK_ADMIN` | 11 | Raw block-device I/O (bypasses the fs layer). |
| `CAP_KIND_FB` | 12 | Map the framebuffer into userspace. |
| `CAP_KIND_CAP_DELEGATE` | 13 | Restrict a child's caps on spawn via `cap_mask`. |
| `CAP_KIND_CAP_QUERY` | 14 | Introspect any process's capability set. |
| `CAP_KIND_IPC` | 15 | Create `AF_UNIX` sockets and `memfd` objects. |
| `CAP_KIND_POWER` | 16 | Call `sys_reboot` (shutdown / reboot); also gates hostname. |
| `CAP_KIND_INSTALL` | 17 | Mutate `/apps` and `/etc/aegis` (herald package manager). |
| `CAP_KIND_NET_LISTEN` | 18 | Bind / listen on a privileged TCP port (< 1024). |
| `CAP_KIND_ADMIN_AUTH` | 19 | Call `sys_admin_session` to elevate to an admin session (sudo-style). Held only by `/bin/login`. |

!!! note "Two kinds are doubly gated"

    Even at `admin` tier, `CAP_KIND_INSTALL` and `CAP_KIND_DISK_ADMIN` are granted
    only inside an **admin session** (`login -elevate` / the `stsh` `admin` flow),
    not on mere login. `DISK_ADMIN` is gated because raw whole-disk I/O bypasses
    every filesystem-layer defense.

## Capability policy format

A program receives capabilities only from `/etc/aegis/caps.d/<exec>`, where
`<exec>` is the binary's basename. The kernel loads every file in this directory
at boot (`cap_policy_load`).

**Syntax** — one or more lines, each:

```
<tier> <TOKEN> <TOKEN> ...
```

- The **first word** is the tier: `service` or `admin`.
- The remaining words are capability **tokens** — the `CAP_KIND_` prefix dropped
  (e.g. `NET_SOCKET` → `CAP_KIND_NET_SOCKET`). Valid tokens: `VFS_OPEN`,
  `VFS_WRITE`, `VFS_READ`, `AUTH`, `CAP_GRANT`, `SETUID`, `NET_SOCKET`,
  `NET_ADMIN`, `THREAD_CREATE`, `PROC_READ`, `DISK_ADMIN`, `FB`, `CAP_DELEGATE`,
  `CAP_QUERY`, `IPC`, `POWER`, `INSTALL`, `NET_LISTEN`, `ADMIN_AUTH`.
- A file may contain multiple lines (tokens accumulate across lines).

**Tiers**

| Tier | When granted |
|------|--------------|
| `service` | Unconditionally, at `exec`. |
| `admin` | Only if the process is authenticated. `INSTALL` / `DISK_ADMIN` additionally require an admin session. |

**Real examples** (from the LoricaOS rootfs)

| File | Contents |
|------|----------|
| `caps.d/httpd` | `service NET_SOCKET` |
| `caps.d/sshd` | `service NET_SOCKET NET_LISTEN` |
| `caps.d/dhcp` | `service NET_SOCKET NET_ADMIN` |
| `caps.d/vigil` | `service POWER INSTALL` |
| `caps.d/login` | `service AUTH SETUID ADMIN_AUTH` |
| `caps.d/herald` | `admin INSTALL` |
| `caps.d/aegisctl` | `admin POWER` |
| `caps.d/installer` | `admin DISK_ADMIN AUTH SETUID` |
| `caps.d/stsh` | `admin DISK_ADMIN POWER CAP_DELEGATE CAP_QUERY`<br>`admin PROC_READ` |

## Kernel command-line options

Tokens parsed by `kernel/core/main.c` from the boot command line. Most are plain
substring tokens; absence selects the default shown.

| Token | Effect |
|-------|--------|
| `boot=text` | Text console: no splash, `printk` writes to the framebuffer normally. |
| `boot=graphical` | Graphical boot: draw the boot splash (default). |
| `quiet` | Suppress `printk` on the framebuffer (graphical boot). |
| `nosmp_sched` | Disable AP scheduling — APs halt, BSP-only. (SMP scheduling is on by default.) |
| `nocow` | Force eager full-page-copy `fork`. (Copy-on-write fork is on by default.) |
| `nolazyfile` | Force eager file-backed `mmap`. (Demand-paged file mmap is on by default.) |
| `proc_trace` | Emit a `[PROC]` line on every fork / exec / exit. |
| `perfbench_mm` | Print a `[PERFMM]` line per fork (address-space dup cost + page count). |
| `perfbench_fs` | Run the ext2 large-file write/read throughput benchmark on root. |
| `hwwatch` | Diag: arm DR0–3 on every CPU to trap wild writes to per-CPU state. |
| `pmm_debug` | Diag: enable the PMM double-free sentinel (backtrace on double-free). |
| `pmm_acct` | Diag: enable the PMM ref/unref accounting ledger. |

## Profiles & key paths

**ISO profiles** (selected by `AEGIS_PROFILE`, default `desktop`)

| Profile | Graphical stack | Skeletons |
|---------|-----------------|-----------|
| `desktop` | Compositor, toolkit, GUI apps, fonts, wallpaper | `rootfs` + `rootfs-desktop` |
| `server` | None — text console only | `rootfs` |

**Key paths**

| Path | Role |
|------|------|
| `/etc/aegis/caps.d/<exec>` | Per-binary capability policy (see format above). |
| `/etc/aegis/admin` | Admin credential — crypt hash verified for session elevation. |
| `/etc/vigil/services/<name>/` | Service definitions (`run`, `policy`, `mode`) read by the `vigil` supervisor. |
| `/bin/vigil` | Service supervisor / init — launches and reaps the configured services. |
