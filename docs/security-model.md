# The Security Model

This is the page that explains why LoricaOS exists.

Every mainstream operating system in wide use shares one assumption: a process,
once running, has *some* baseline of ambient power, and a privileged identity
(`root`, `uid 0`, `Administrator`, `SYSTEM`) that can do anything. Security is
then the work of taking power *away* — sandboxes, seccomp filters, SELinux,
AppArmor, capabilities-on-top-of-root. You start from omnipotence and subtract.

LoricaOS starts from zero and adds. A process begins with almost no authority
over the system, and gains a specific power only when it was explicitly handed
an **unforgeable capability** for it. There is no superuser to subtract from.
This is the single idea the whole system is built around, and this page
describes exactly how the **Aegis** kernel enforces it.

!!! info "Where this lives in the source"

    Nothing on this page is aspirational. The mechanisms below are in the kernel
    today:

    - Capability table + `cap_check` — `kernel/cap/cap.h`, `kernel/cap/src/lib.rs`
    - Policy engine (`/etc/aegis/caps.d`) — `kernel/cap/cap_policy.c`
    - Per-process security fields — `kernel/proc/proc.h`
    - The syscall gates — `kernel/syscall/sys_socket.c`, `sys_disk.c`, `sys_identity.c`, `sys_cap.c`
    - Admin elevation (userspace half) — `user/bin/login/main.c`

---

## No ambient authority

The core principle, stated precisely:

> A process can affect the system only through capabilities that were granted to
> it at the moment it was `exec`'d. It holds no standing authority by virtue of
> who started it, what its uid is, or that it is simply running.

Concretely, every Aegis process carries a fixed-size **capability table** in its
process control block:

```c
/* kernel/proc/proc.h */
cap_slot_t caps[CAP_TABLE_SIZE];   /* CAP_TABLE_SIZE == 64 */
```

Each slot is an 8-byte `(kind, rights)` pair. The table is not something the
process can write to. It is populated **by the kernel** from policy when the
process is created (`exec`/`spawn`), and from then on it is the complete,
exhaustive statement of what that process is allowed to ask the kernel to do.

When a process makes a privileged syscall, the handler consults this table
before doing anything else. If there is no matching capability, the syscall
**fails closed** — it returns `ENOCAP` and the operation never happens.

!!! note "Fail closed, not fail open"

    `ENOCAP` is the default answer. A syscall reaches its real work only by
    *passing* an explicit `cap_check`. A missing check is a missing grant, not a
    silent allow. The design bias is that forgetting to add authority is safe;
    you simply cannot do the thing. (Forgetting to add a *check* is the dangerous
    direction, which is why the privileged surface is small and audited.)

---

## There is no root — `uid 0` is just a number

LoricaOS has users and uids. The primary user created at install **is `uid 0`**.
But `uid 0` is not special to the kernel. Read the comment the kernel author left
on the `setuid` syscall:

```c
/* kernel/syscall/sys_identity.c */
/*
 * uid is cosmetic in Aegis (capabilities are the real authority), but it still
 * drives ext2 DAC checks ...
 */
```

`uid` exists only because POSIX file ownership (ext2 permission bits) needs an
identity to compare against. It is a **label for discretionary file access**,
nothing more. The kernel privileges `uid 0` with *no* capability, bypasses *no*
`cap_check` for it, and grants it *no* implicit power. There is no code path
anywhere in Aegis of the form "if `uid == 0`, allow." Authority is the
capability table; identity is just a name on a file.

This flips the most dangerous assumption in conventional systems. Compromising
the `uid 0` process does not hand an attacker the keys to the machine, because
`uid 0` *as such* holds no keys. A process running as `uid 0` with only the
baseline capabilities can no more reconfigure the network or write the raw disk
than any other process — it lacks `NET_ADMIN` and `DISK_ADMIN`, and the kernel
does not care what its uid is.

!!! warning "`setuid` cannot be used to forge identity, either"

    Because uid drives DAC checks, an unrestricted `setuid(0)` would be a back
    door to root-owned files. So `setuid` is itself capability-gated **and**
    identity-bound: a caller needs `CAP_KIND_SETUID`, and may only transition to
    the uid that was actually authenticated for this session (`proc->auth_uid`,
    bound by `login` at credential-check time). A process that never
    authenticated has `auth_uid == AUTH_ID_NONE` and **cannot change its uid at
    all**. A `SETUID` holder can drop to the one identity whose password was
    checked — not "become any uid including 0."

---

## Capabilities at the syscall boundary

The enforcement point is the syscall handler. Every privileged operation begins
with a `cap_check` against the calling process's table. The check itself is tiny
and total:

```c
/* kernel/cap/cap.c — 0 if a matching slot exists, else -ENOCAP */
int cap_check(const cap_slot_t *table, uint32_t n, uint32_t kind, uint32_t rights)
{
    if (table == 0 || n == 0)
        return -ENOCAP;
    if (n > CAP_TABLE_SIZE)
        n = CAP_TABLE_SIZE;
    for (uint32_t i = 0; i < n; i++) {
        if (table[i].kind == kind && (table[i].rights & rights) == rights)
            return 0;
    }
    return -ENOCAP;
}
```

A capability is matched on its **kind** (what class of operation) and its
**rights** bitfield (`READ`/`WRITE`/`EXEC`). The handler asks for exactly what it
is about to do, and proceeds only on a `0` return. Some real gates, verbatim:

=== "Opening a socket"

    ```c
    /* kernel/syscall/sys_socket.c — sys_socket */
    if (cap_check(proc->caps, CAP_TABLE_SIZE,
                  CAP_KIND_NET_SOCKET, CAP_RIGHTS_READ) != 0)
        return SYS_ERR(ENOCAP);
    ```

=== "Binding a privileged port"

    ```c
    /* kernel/syscall/sys_socket.c — bind to a port < 1024 */
    if (cap_check(proc->caps, CAP_TABLE_SIZE,
                  CAP_KIND_NET_LISTEN, CAP_RIGHTS_WRITE) != 0)
        return SYS_ERR(ENOCAP);
    ```

=== "Raw disk I/O"

    ```c
    /* kernel/syscall/sys_disk.c — sys_blkdev_io */
    if (cap_check(proc->caps, CAP_TABLE_SIZE,
                  CAP_KIND_DISK_ADMIN, CAP_RIGHTS_WRITE) != 0)
        return SYS_ERR(ENOCAP);
    ```

There is no way for userspace to forge a slot. The capability table is kernel
memory; the only paths that write it are the process-creation paths, which read
the kernel's own policy. There is **no runtime "inject a capability into a PID"
syscall** — one used to exist (`sys_cap_grant_runtime`, 363) and it was
deliberately *removed*, because a primitive that grants standing authority into a
running process is exactly the ambient authority the model exists to forbid.

### The capability kinds

These are every capability kind the kernel defines (`kernel/cap/cap.h`). The ones
marked *baseline* are granted unconditionally to every process; the rest must be
named in policy.

| Token | Kind | Gates | Tier |
|---|---|---|---|
| `VFS_OPEN` | 1 | `open()` | baseline |
| `VFS_WRITE` | 2 | `write()` | baseline |
| `VFS_READ` | 3 | `read()` | baseline |
| `IPC` | 15 | AF_UNIX sockets, memfd | baseline |
| `PROC_READ` | 10 | read another PID under `/proc` | baseline |
| `THREAD_CREATE` | 9 | `clone(CLONE_VM)` | baseline |
| `AUTH` | 4 | read credentials, call `sys_auth_session` | service / admin |
| `SETUID` | 6 | `setuid`/`setgid` (to the bound identity) | admin |
| `NET_SOCKET` | 7 | create TCP/UDP sockets | service |
| `NET_LISTEN` | 18 | bind/listen on a port < 1024 | service |
| `NET_ADMIN` | 8 | set IP/mask/gateway (`sys_netcfg`) | service |
| `FB` | 12 | map the framebuffer | service |
| `POWER` | 16 | `reboot`/`shutdown` | service / admin |
| `CAP_DELEGATE` | 13 | restrict a child's caps on spawn | admin |
| `CAP_QUERY` | 14 | introspect another process's caps | admin |
| `DISK_ADMIN` | 11 | raw whole-disk read/write | **admin-session** |
| `INSTALL` | 17 | mutate `/apps` and `/etc/aegis` (herald) | **admin-session** |
| `ADMIN_AUTH` | 19 | grant an admin session (held only by `login`) | service |

The baseline is intentionally humble: open/read/write files (still subject to
ext2 ownership), talk over local IPC, read your own `/proc`, make a thread.
A program that does only computation and local I/O needs nothing more. Everything
with real blast radius — the network, the raw disk, power, the install trees,
identity — is opt-in and named.

---

## Capability policy: `/etc/aegis/caps.d/<exec>`

Where do a process's non-baseline capabilities come from? From a **policy file
named after the binary**, under `/etc/aegis/caps.d/`. At `exec` time the kernel
resolves the executable's path to its basename, looks up the matching policy
entry, and grants the caps it declares (`cap_apply_policy`, called from
`sys_exec.c`).

The file format is deliberately trivial — one or more lines, each a **tier**
keyword followed by **capability tokens**:

```
<tier> TOKEN [TOKEN ...]
```

The tier is `service` or `admin`. Real, shipped examples (these are the actual
files in `rootfs/etc/aegis/caps.d/`):

```text
# caps.d/curl  — an HTTP client needs to open sockets, nothing else
service NET_SOCKET
```

```text
# caps.d/sshd  — listen on the privileged SSH port
service NET_SOCKET NET_LISTEN
```

```text
# caps.d/dhcp  — open sockets AND set interface addressing
service NET_SOCKET NET_ADMIN
```

```text
# caps.d/login — verify credentials, bind identity, and grant admin sessions
service AUTH SETUID ADMIN_AUTH
```

```text
# caps.d/installer — write the raw disk, check credentials, set identity
admin DISK_ADMIN AUTH SETUID
```

```text
# caps.d/herald — the package manager mutates the install-protected trees
admin INSTALL
```

The two tiers mean exactly this (`cap_policy.c`, `cap.h`):

- **`service`** — granted **unconditionally** at `exec`. The binary gets these
  caps whether or not anyone has logged in. This is for daemons and tools whose
  authority is intrinsic to their job (a web server *is* a thing that opens
  sockets).
- **`admin`** — granted **only if the session is authenticated** (`proc->authenticated`,
  set by `login` after a successful credential check). An admin-tier cap on a
  binary launched from an unauthenticated context is silently withheld.

!!! note "No policy file means baseline only"

    A binary with **no** `caps.d/<name>` entry receives exactly the baseline
    capabilities and nothing else. This is the common case: most programs never
    need a policy file. Authority is the exception you write down, not the
    default you inherit. The same is true of a binary the kernel doesn't trust
    the *location* of — see below.

!!! warning "Authority is anchored to trusted, write-protected paths"

    Because caps are derived from a binary's path, that path must be unforgeable.
    The kernel only grants policy caps to executables that live under a trusted
    anchor — `/bin`, `/sbin`, `/apps`, or a registered `/etc/aegis/anchors`
    entry — **and** are confirmed install-protected at the filesystem layer at
    grant time (`ext2_path_under_protected`). The "trusted-for-granting" set and
    the "write-protected" set are deliberately the *same* set. Without this, an
    unprivileged user could drop a forged ELF at `/tmp/reboot` (basename
    `reboot` → `POWER`) or `/tmp/login` (→ `AUTH`) and inherit authority by
    choosing a filename. Path traversal (`..`) is rejected, and the protected
    trees can only be modified through an admin-gated install. This was a real
    privilege-escalation hole that was found and closed; the defense is
    load-bearing.

---

## Admin elevation: even `uid 0` must re-authenticate

This is the part that most clearly separates LoricaOS from "sudo on Linux."

The most dangerous capabilities — `DISK_ADMIN` (raw whole-disk I/O, which lands
*beneath* the filesystem and bypasses every fs-layer defense at once) and
`INSTALL` (mutate the protected system trees) — are **not** granted on mere login,
even at `admin` tier, even for `uid 0`. They require a separate, second
authentication: a **sudo-style admin session**.

The flow is a true authority gate, not a uid check:

1. A logged-in shell wants admin power. It cannot grant itself an admin session —
   no shell holds the capability to do so.
2. The shell runs **`login -elevate`** as a child process. `login` is the single
   trusted authenticator: it holds `CAP_KIND_ADMIN_AUTH` (`caps.d/login`) and owns
   the credential-checking machinery.
3. `login -elevate` reads a **separate credential** — `/etc/aegis/admin`, distinct
   from any login password — prompts for it, and verifies it in userspace:

    ```c
    /* user/bin/login/main.c — do_elevate() */
    fd = open(ADMIN_CRED_FILE, O_RDONLY);          /* "/etc/aegis/admin" */
    /* ... read stored hash, prompt with echo off ... */
    ok = (auth_verify(password, stored) == 0);
    if (!ok) { /* admin authentication failed */ return 1; }
    if (syscall(SYS_ADMIN_SESSION, 1L) != 0) { /* kernel denied */ return 1; }
    ```

4. Only after the credential verifies does `login` call the kernel's
   `sys_admin_session(1)` (syscall 517). The kernel re-checks `CAP_KIND_ADMIN_AUTH`,
   and applies the admin session to `login`'s **direct parent** — the shell that
   invoked it, and nothing else:

    ```c
    /* kernel/syscall/sys_cap.c — sys_admin_session */
    if (cap_check(proc->caps, CAP_TABLE_SIZE,
                  CAP_KIND_ADMIN_AUTH, CAP_RIGHTS_WRITE) != 0)
        return SYS_ERR(EPERM);
    parent = proc_find_by_pid(proc->ppid);
    parent->admin_session = 1u;
    cap_apply_policy(parent->caps, parent->exe_path, ...);  /* re-derive caps now */
    ```

Only with `proc->admin_session` set does `cap_apply_policy` grant `DISK_ADMIN` or
`INSTALL`:

```c
/* kernel/cap/cap_policy.c — the anti-ambient-authority gate */
int ok = (cap->kind == CAP_KIND_INSTALL ||
          cap->kind == CAP_KIND_DISK_ADMIN)
             ? admin_session       /* the strict tier */
             : authenticated;      /* ordinary admin-tier */
if (ok) cap_grant(...);
```

!!! info "Login authority and install authority are deliberately distinct"

    The kernel's single chokepoint for "may this caller perform an administrative
    install operation" — `admin_session_active()` — checks `proc->admin_session`,
    **not** `proc->authenticated`. The comment in the source is explicit that this
    is the whole point: being logged in (even as `uid 0`) is not the same as being
    elevated. `uid 0` is not the authority gate. The admin session is.

So the real answer to "what is root in LoricaOS?" is: nothing is. The closest
equivalent — the ability to write the raw disk or rewrite the system — is a
*transient session state* you must re-authenticate for with a *separate
credential*, granted by *one minimal authenticator* to *one specific shell*, and
which that shell can drop at any time (`sys_admin_session(0)`, always permitted,
no credential — a session may always relinquish its own privilege).

---

## How an attack plays out

To make the model concrete, here is what happens when something hostile runs:

| Scenario | Conventional OS | LoricaOS |
|---|---|---|
| Malware runs as the logged-in user | Has the user's full ambient authority | Has the baseline caps of *its binary's* policy — typically file I/O only |
| Process is `uid 0` | Can do anything | Holds whatever caps its policy granted; uid buys it nothing extra |
| Tries to open a raw socket with no `NET_SOCKET` | Allowed (or gated by add-on LSM) | `sys_socket` → `ENOCAP`, no socket |
| Tries to overwrite the raw disk | Allowed if root | `sys_blkdev_io` → `ENOCAP` unless it holds `DISK_ADMIN` *and* an admin session |
| Drops a forged binary named `login` into `/tmp` and execs it | Filename is irrelevant; runs as the caller | No policy caps — `/tmp` is not a trusted, write-protected anchor |
| Steals a login password | Often a path to root | Gets that user's identity; still no `DISK_ADMIN`/`INSTALL` without the *separate* admin credential |

The model does not make the system unbreakable. It removes the single point of
total failure — the all-powerful identity — and replaces it with a graph of small,
explicit, individually-checked grants.

---

## An honest accounting of the limits

!!! warning "The model is sound; the implementation is a v1 C kernel"

    Everything above describes a coherent and, we believe, genuinely strong
    security *model*, and it is really enforced — these are not policies a config
    flag turns off.

    But a model is only as strong as the code that enforces it, and that code is
    a young, hand-written C kernel. **It almost certainly contains exploitable
    memory-safety bugs.** A buffer overflow in a driver, a missed bounds check at
    the syscall boundary, a use-after-free in the VFS, a TOCTOU race in the path
    resolver (one such race in the install-protected-path check is documented in
    the source as a known, not-yet-closed follow-up) — any of these could let an
    attacker bypass `cap_check` entirely by corrupting kernel memory, rather than
    by playing by its rules. When that happens, the capability table is just bytes
    an attacker can rewrite.

    The capability subsystem itself is kept deliberately tiny — a small,
    auditable core (`kernel/cap/cap.c`) so the most security-critical path is
    easy to review — but it is C like the rest of the kernel, so that attack
    surface is real. (The core was originally `no_std` Rust; it was migrated to
    C so the kernel builds with a C toolchain alone.)

    **Do not deploy LoricaOS where a breach would matter.** Do not expose it to
    untrusted networks, do not trust it with secrets, and do not run anything on it
    you cannot afford to lose. We would rather tell you exactly where the edges are
    than imply they aren't there. The point of this project is to show that a
    no-ambient-authority OS can be built and can work — not to claim it has been
    hardened against a determined attacker. It has not.

---

## See also

- [Architecture](architecture.md) — how the kernel, userland, and desktop fit together.
- [Packages (herald)](packages.md) — how `INSTALL` and the signed package chain work.
- [Reference](reference.md) — the full syscall and capability-token reference.
- [Kernel](kernel.md) — Aegis internals and the syscall surface.
