# The Desktop

LoricaOS ships a complete graphical desktop on the desktop ISO. It is built from
four cooperating pieces — the **Bastion** greeter, the **Lumen** compositor, the
**Glyph** toolkit, and the **Citadel** dock — plus a set of small applications,
each one its own program shipped as a signed [herald](packages.md) package.

Nothing here is a monolith. Lumen owns the screen and draws the desktop; every
application is a *separate process* that connects to the compositor over an
`AF_UNIX` socket to ask for a window. There is no in-process plugin model and no
shared display authority — the same boundary the rest of LoricaOS is built on
runs straight through the GUI.

---

## Bastion — the login greeter

The first graphical thing you see on a desktop boot is **Bastion**, the display
manager and greeter. It draws the LoricaOS logo and a username/password prompt
directly to the framebuffer — before any compositor exists yet — and
authenticates you against the system credential store (`/etc/passwd` +
`/etc/shadow`).

On a successful login, Bastion *elevates the session* (binding your verified
uid/gid so the session can only ever assume that identity) and launches the Lumen
compositor as you. On a text-console boot it is never started at all.

!!! tip "Logging in on the live ISO"

    The live image ships a single account. At the Bastion prompt, log in with:

    - **Username:** `live`
    - **Password:** `live`

    Privileged actions (the installer, Settings' admin panes) prompt for the
    live system's **admin password**: `administrator`. (On an installed system
    this is the admin password you chose at install time — change it any time
    with `adminpw`.)

    The live session also auto-starts the graphical installer's dock icon, so you
    can install to disk from inside the desktop.

---

## Lumen — the compositor and display server

**Lumen** is the heart of the desktop: the compositing display server. It maps
the framebuffer, draws the desktop background, and is the single process every
graphical application talks to in order to get a window. The visible furniture
Lumen draws itself is:

- **The top bar** — spanning the top of the screen, with the **LoricaOS menu** at
  the top-left (power, restart, "About", and quick actions), and a clock and
  volume control at the right.
- **Frosted-glass windows** — application windows are composited with soft
  shadows, rounded corners, and a translucent frosted-glass chrome that blurs the
  desktop behind them. An elevated/admin window gets an unmistakable red titlebar
  tint so you always know when you're running something privileged.
- **The Citadel dock** — the launcher bar pinned to the bottom-centre (see below).
- **A dropdown terminal** — a quake-style terminal that slides down over whatever
  you're doing. Toggle it any time with **Ctrl+Alt+T**.

### Apps are external clients, not plugins

Each GUI application is an independent program that connects to Lumen's socket at
`/run/lumen.sock`. The handshake is simple: the app asks for a window, Lumen
hands back a shared-memory buffer (`memfd`) over the socket, the app draws its
pixels into that buffer and tells Lumen which regions changed, and Lumen
composites them onto the screen. Input — keyboard, mouse, focus, drag-and-drop —
flows back to the focused app over the same socket.

This means a misbehaving app cannot scribble on the framebuffer, read another
app's input, or take down the desktop. It only ever sees its own window. In
[capability](security-model.md) terms, a pure GUI client is granted `service`
and nothing more; Lumen itself holds only `FB` (framebuffer), `THREAD_CREATE`,
`PROC_READ`, and `POWER`.

!!! note "Glyph — the toolkit underneath"

    Every graphical component links **Glyph**, the LoricaOS GUI toolkit. Glyph
    provides the software drawing primitives, the TTF text rendering (Inter for
    UI, JetBrains Mono for the terminal), the runtime theme system, procedural
    icons, the terminal-emulator core, and the *client* side of Lumen's window
    protocol. It is a build-time library baked into each app — there is no shared
    runtime to install. It's also why the desktop has a single consistent look
    and a live theme: light/dark mode and the accent colour are read from
    `/etc/aegis/theme.conf` and `~/.config/aegis/theme.conf` and applied without
    a restart.

---

## Citadel — the dock

The **Citadel dock** is the persistent launcher bar at the bottom-centre of the
screen. Like every other app it's an external client of the compositor: it draws
a row of frosted-glass app icons into a panel Lumen gives it, and when you click
an icon it asks Lumen to launch that app (the compositor resolves the name
against the `/apps` bundle registry and spawns it). The dock holds *no*
capabilities of its own — it launches apps by asking the compositor, not by
spawning them itself. It also can't be closed from the UI.

For the full list of installed apps, click the **Applications** icon to open the
full-screen, Launchpad-style application grid.

---

## Window controls

Every application window has a **titlebar** along its top. At the **top-left**
sit the traffic-light controls:

- **Close** — closes the window.
- **Minimize** — hides the window.
- **Maximize** — fills the screen.

**Drag the titlebar** to move a window around the desktop. Click anywhere in a
window to focus and raise it to the top of the stack.

---

## The apps

The desktop image ships the applications below. Each is a standalone herald
package installed as an `/apps` bundle — launch them from the dock or the
Applications grid.

| App | What it does |
|-----|--------------|
| **Terminal** | A full terminal emulator window (`/bin/terminal`) — separate from the built-in Ctrl+Alt+T dropdown. |
| **Files** | Graphical file manager for browsing the filesystem, with drag-and-drop. |
| **Text Editor** | A simple plain-text editor for viewing and editing files. |
| **Settings** | System settings — theme (light/dark + accent), clock format, wallpaper, night light, pointer, and more. |
| **Calculator** | A basic calculator. |
| **Run** | A "Run…" launcher: type a command or app name to start it. |
| **Tunes** | A minimal music player; streams WAV/MP3 to the audio device. |
| **Calendar** | A month-view calendar. |
| **Image Viewer** | Opens a single image with fit / zoom / pan, and can drag the file back out. |
| **System Monitor** | A live task and resource monitor — memory bar, uptime, process count, and a sortable process list. |
| **Network Manager** | Shows live `eth0` configuration — link state, IPv4 address, subnet, gateway, DNS, and MAC — refreshed automatically. |

!!! tip "Adding more apps"

    These are just the apps that ship on the desktop image — the desktop is not a
    fixed set. Every app is a signed `.hpkg` package, so you can install more with
    the **herald** package manager (`herald install <name>`). See
    [Packages](packages.md) for how packaging and installation work.
