# Getting Started

This page takes you from nothing to a booted LoricaOS system — download an
image, run it in a virtual machine, log in, and (optionally) install it to a
disk.

LoricaOS is a capability-secure operating system built on the
[Aegis](https://github.com/LoricaOS/Aegis) kernel. The kernel grants no ambient
authority: every program reaches the system only through capabilities it has
been explicitly granted. One visible consequence shows up the moment you install
— there is **no separate `root` account**. The first (and only) user you create
*is* uid 0.

## 1. Download an image

LoricaOS ships two ISO profiles. Both are live, bootable images that can also
install themselves to disk.

| Profile | File | What you get |
| --- | --- | --- |
| **Desktop** | `loricaos-desktop.iso` | Graphical session — the Lumen compositor, Glyph toolkit, Citadel shell, and the Bastion display manager (greeter). |
| **Server** | `loricaos-server.iso` | Text console only. Zero graphical stack. |

Grab the latest of either from GitHub Releases:

> **<https://github.com/LoricaOS/LoricaOS/releases>**

Pick `loricaos-desktop.iso` if you want the graphical experience, or
`loricaos-server.iso` for a headless / text-console system.

## 2. Boot it in QEMU

The quickest way to try LoricaOS is in QEMU. The images expect a standard PC
machine, a CD-ROM boot, and at least 2 GB of RAM.

### Desktop (graphical)

For the desktop ISO, give QEMU a display and a VGA adapter:

```bash
qemu-system-x86_64 \
    -machine pc \
    -cdrom loricaos-desktop.iso \
    -boot order=d \
    -m 2048M \
    -vga std \
    -display gtk \
    -no-reboot
```

LoricaOS boots through the Aegis kernel, mounts the root filesystem, execs init
(`/bin/vigil`), and brings up the display stack to the **Bastion greeter** — the
graphical login screen. (Swap `-display gtk` for `-display sdl` or `-display
cocoa` to match what your QEMU build supports.)

### Server / headless (text console)

For the server ISO — or to watch the boot on the serial console — drop the
display and route the console to your terminal. This mirrors how the project's
own boot test (`tools/ostest.sh`) drives the image:

```bash
qemu-system-x86_64 \
    -machine pc \
    -cdrom loricaos-server.iso \
    -boot order=d \
    -m 2048M \
    -display none \
    -vga std \
    -nodefaults \
    -serial stdio \
    -no-reboot
```

The server profile boots to a text login prompt instead of the graphical
greeter.

!!! note "It's a real OS, not an emulator target only"
    These same images boot on bare metal. QEMU is just the fastest way to look
    around first. To run on real hardware, write the ISO to a USB stick (see
    [below](#write-to-usb-bare-metal)).

## 3. Log in as the live user

The live ISO comes with one preconfigured account:

| Username | Password | uid |
| --- | --- | --- |
| `live` | `live` | **0** |

The live system's **admin password** — the credential that unlocks capability
elevation (the installer, Settings' privileged panes, `aegisctl`) — is:

```
administrator
```

There is no `root` login to fall back to — **the `live` user *is* uid 0**. That
is not a shortcut for the live image; it is how LoricaOS works. Authority comes
from capabilities, not from a magic account name.

- On the **desktop** ISO, sign in at the Bastion greeter as `live` / `live`.
- On the **server** ISO, enter `live` / `live` at the console login.

You now have a fully running, in-memory LoricaOS session. Nothing has touched
your disk yet.

## 4. Install to a disk

When you're ready to install LoricaOS permanently, run the installer from the
live session. There are two front-ends over the same install logic — pick the
one matching your profile:

- **Desktop:** launch **Install LoricaOS** from the session (a five-screen
  graphical wizard: Welcome → Disk → User → Confirm → Progress).
- **Server / text:** run the text installer:

  ```bash
  installer
  ```

Both flows are the same underneath. Here's what to expect.

### Admin authorization first

Writing raw disks requires the `DISK_ADMIN` capability, which the kernel grants
only to an **admin-elevated** session. So the installer asks for an **admin
password up front** — this is the admin credential of the *live system* you're
installing from. Without it, disk enumeration comes back empty and nothing is
written.

On the live ISO, the admin password is `administrator`.

### One user, created at uid 0

The defining step: the installer asks for exactly **one username and one
password**, and creates **that** user at **uid 0**.

- No separate `root` account is created.
- No `1000+` "regular user" convention — your account is the first user, and the
  first user is uid 0.
- **Admin password, your choice of mode.** Both installers then ask for an
  admin password — the credential that authorizes elevated actions later (the
  sudo-style flow). Leave it **blank** and your account password doubles as
  the admin credential; type one and your system has a **separate** admin
  password from day one.

The text installer states this plainly while collecting your account:

> This user is uid 0 — the first user. LoricaOS has no separate root;
> admin actions are authorized by the admin password (sudo-style).

### Changing the admin password later

On any installed (or live) system, rotate the admin credential with:

```bash
adminpw
```

It asks for the **current** admin password (verified by the same trusted
authenticator that performs elevation), then for the new one twice. Your
account/login password is unaffected — `adminpw` changes only the elevation
credential.

### Pick a disk and confirm

The installer lists eligible target disks (raw disks; ramdisks and partitions
are filtered out) and flags any that already contain a LoricaOS install.

!!! danger "The target disk is erased"
    Installing writes a fresh GPT and wipes the selected disk. **All existing
    data on it is destroyed.** Credentials are collected *before* the destructive
    write, so cancelling at any earlier point is safe and leaves the disk
    untouched.

Once you confirm, the installer writes the partition table, copies the root
filesystem, installs the bootloader, and writes your account into
`/etc/passwd`. When it finishes, remove the ISO and reboot — LoricaOS now starts
from disk, and you log in with the account you just created.

## Write to USB (bare metal) {#write-to-usb-bare-metal}

To boot LoricaOS on a physical machine, write the ISO to a USB stick. On Linux
or macOS:

```bash
# Replace /dev/sdX (Linux) or /dev/diskN (macOS) with your USB device.
# Double-check the device — this overwrites it entirely.
sudo dd if=loricaos-desktop.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

Boot the target machine from that USB device, and you land in the same live
session as in QEMU — log in as `live` / `live`, then run the installer to put
LoricaOS on the machine's internal disk.

## Next steps

- Explore the capability model — why there's no root and how `/etc/aegis/caps.d`
  governs what each program may do.
- On the desktop, poke around the Lumen desktop and the Citadel shell.
- On the server, you have a text console with the LoricaOS coreutils and net
  tooling.

!!! warning "Maturity: this is a v1 system"
    LoricaOS is young and built from scratch. The security model is real and
    deliberate, but the implementation has not been through years of hardening —
    it may contain real vulnerabilities and rough edges. Run it in a VM or on
    hardware you don't mind reinstalling. **Do not put data you can't afford to
    lose on a LoricaOS install, and don't rely on it to protect anything yet.**
