# Packages (herald)

LoricaOS distributes software as signed **`.hpkg`** packages, installed by **herald**, the system package manager. The desktop image is itself assembled from herald packages, and more are installable on demand from a **Chancery**-signed repository.

Like everything else in LoricaOS, herald assumes no ambient authority. It does not hold the network capability; it cannot mutate the system app tree on its own; and it trusts nothing it has not cryptographically verified. The transport is explicitly *outside* the trusted computing base — every artifact is checked against an offline signing key after it is fetched, so plain HTTP and a MITM'd HTTPS are equally safe.

## What herald is

`herald` is a command-line package manager. A herald "bears trusted, signed software from beyond the walls": it verifies a package's signature (or its repository-pinned hash), extracts the payload into place, installs the package's capability policy, records it in a local database, and asks the kernel to reload its policy — with no reboot.

### Commands

```bash
herald sync                    # fetch + verify repository metadata
herald search <term>           # search the synced package lists
herald install <name>          # install from a repository (resolves dependencies)
herald install <file.hpkg>     # install a local package file
herald verify  <file.hpkg>     # check a package's signature only (no install)
herald info    <file.hpkg|id>  # show package or installed-package info
herald list                    # list installed packages
herald remove  <id>            # uninstall a package
```

A typical session installs a desktop app by name. herald resolves its dependencies, downloads each from the repository, verifies every payload against the hash pinned by the signed repository metadata, and installs dependencies first:

```bash
herald sync
herald install lumen-terminal
```

`herald install` distinguishes its two modes by the argument: anything ending in `.hpkg` is treated as a **local file** (verified against its detached `.hpkg.sig` signature, then installed), and anything else is a **repository package name** resolved through the cached, signature-verified index.

```bash
herald install ./lumen-calculator.hpkg   # local file: checks lumen-calculator.hpkg.sig
herald install lumen-calculator          # repo name: resolves + downloads + verifies
```

`herald list` and `herald info <id>` read the local install database (`/var/lib/herald/`); `herald info <file.hpkg>` reads and verifies a package file directly, printing its manifest (`id`, `name`, `version`, `exec`, `caps`, `depends`).

!!! note "herald has no network capability of its own"

    herald does not open sockets. To download, it `fork`/`exec`s `/bin/curl`,
    which carries the `NET_SOCKET` capability via its own
    `/etc/aegis/caps.d/curl` policy — herald does not. The network surface stays
    in curl. Because integrity never depends on the transport, curl is invoked
    with `-k` (no TLS certificate validation); the signature and hash checks
    after the fetch are what make the install trustworthy.

## The `.hpkg` format

A `.hpkg` is a **manifest-first, uncompressed POSIX `ustar` archive**, accompanied by a detached **ECDSA-P256 / SHA-256** signature in a sibling `.hpkg.sig` file.

- **Manifest first.** The first member of the archive is named `manifest` — a small `key=value` INI (`id`, `name`, `version`, `exec`, `class`, `caps`, `depends`, optional `paths` and per-binary `caps.<binary>=` lines). herald reads it before touching the payload.
- **Uncompressed ustar.** The payload is a plain tar of the install tree. There is no compression layer and no scripting hooks — an install is an extraction, not an executed program.
- **Detached signature.** The signature covers the whole archive. herald verifies it against a trusted public key baked into the binary before extracting anything.

Building one is mechanical (see a component's `tools/pack.sh`): stage the payload tree, write the `manifest`, then `tar --format=ustar` the manifest first followed by the payload roots, and sign the result with `openssl dgst -sha256 -sign`.

### `class=system` packages

Every first-party LoricaOS component (the whole Lumen desktop stack) is **`class=system`**: a signature-trusted package whose herald `id` may differ from its bundle/exec name and which may install **across multiple trees** — `/bin`, `/apps`, `/etc/aegis/caps.d`, `/etc/vigil`, `/usr/share`, and so on. herald installs its **whole payload tree verbatim** to `/`, skipping the constraints applied to third-party apps (the capability allow-list, the `exec == id` rule, the single-bundle-directory restriction).

The trust justification is the signature itself: only the first-party key signs the repository, so a `class=system` package is exactly as trusted as the rootfs that shipped herald. For example, `lumen-calculator` (id) installs the `calculator` (exec) bundle plus its capability policy:

```
/apps/calculator/calculator     the app binary
/apps/calculator/app.ini        the bundle descriptor
/etc/aegis/caps.d/calculator    its capability policy
```

Two lines of defense are **not** bypassed even for system packages: extraction still rejects absolute paths and `..` traversal, and the kernel still gates every write to the protected trees (`/bin`, `/sbin`, `/apps`, `/etc/aegis`) on herald's unforgeable `CAP_KIND_INSTALL` capability — which is held only in an **authenticated admin session**. Without it, the install fails with a permission error.

Third-party (`class=app`) packages stay fully constrained: their requested capabilities are checked against an allow-list (escalation caps like `INSTALL`, `SETUID`, `NET_ADMIN`, `POWER` are refused), they may only own a capability policy whose name matches their own bundle id, and they may not overwrite a policy they do not already own.

## Chancery and the trust model

Packages come from a **Chancery**-signed repository, served in a Debian-style layout: a signed top-level **Release** index pins the per-architecture **Packages** list, which pins each individual **package** hash.

The trust flows down a single chain:

1. herald fetches `Release` and `Release.sig` and verifies the signature against the embedded trusted key. If it fails, the repository is rejected outright.
2. The verified `Release` contains the **SHA-256 of the `Packages` list**. herald fetches `Packages` and confirms its hash matches. A mismatch is treated as tampering and refused.
3. Each stanza in the verified `Packages` list carries the **SHA-256 of its package file**. On `install`, herald downloads the `.hpkg` and confirms its hash against that pinned value.

!!! info "Trust chain: sign once, verify everything"

    A single signature over **Release** vouches for the **Packages** hash, which
    in turn vouches for **each package** hash:

    ```
    Release.sig  →  Release  →  Packages hash  →  package hash
    (signed)        (pins)      (pins)            (verified on install)
    ```

    Individual packages are therefore **not** signed one by one — the chain
    validates them. This is the Debian signed-repository model: tampering with
    any link (the index, the package list, or a package file) breaks a hash and
    herald refuses to proceed. Because integrity rests entirely on this chain
    and the offline signing key, the download transport itself is untrusted and
    need not be secured.

### Sources

Repositories are configured in `/etc/herald/sources.list`, one per line as `<base-url> <suite> <component>`:

```
https://herald.byexec.com stable main
```

For each source, herald fetches `<base-url>/dists/<suite>/Release` (and `.sig`) and `<base-url>/dists/<suite>/<component>/binary-<arch>/Packages`, caching the verified lists under `/var/lib/herald/lists/`.

!!! note "Repository host is moving"

    The repository is currently served at **`herald.byexec.com`** — a temporary
    host. It is moving to **`herald.loricaos.org`**; the `sources.list` entry
    will change accordingly. The trust model is unaffected: the host name is not
    trusted, only the signing key is.

## Capabilities for installed apps

Installing software on LoricaOS also declares **what that software is allowed to do**. A package ships its capability policy as part of its payload, under `/etc/aegis/caps.d/`, keyed by the executable's basename. A pure GUI client ships a `service` policy and nothing more; anything beyond that (`NET_SOCKET`, and so on) must be declared explicitly and, for third-party packages, must pass herald's escalation allow-list.

When herald finishes writing a package's bundle and its `caps.d` files, it issues a single syscall asking the kernel to **reload its capability policy and trusted-path anchors** — so the new policy takes effect immediately, with no reboot. A package that installs an engine library outside `/apps` additionally registers that prefix as a trusted-path anchor, preserving the kernel invariant that a tree trusted to grant capability policy is also write-protected.

The upshot: a herald install is not just "drop files on disk." It is a verified payload plus an explicit, kernel-enforced statement of authority — declared at install time, granted at exec time, and never ambient. See [The Security Model](security-model.md) for how `/etc/aegis/caps.d/<exec>`, capability tokens, and the no-root rule fit together.
