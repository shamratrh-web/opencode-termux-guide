# opencode on Termux — Full Guide

Run [opencode](https://opencode.ai) — the open-source terminal AI coding agent — **natively on
Android via Termux**. No proot, no chroot, no Linux VM, no root required.

This guide documents the working setup, powered by the bionic (Android-native) builds from
[bd-loser/opencode-bionic](https://github.com/bd-loser/opencode-bionic).

- One-command install / update
- SHA256-verified `.deb` packages
- Native aarch64 binary (runs unrooted, Android 7.0+)

> opencode upstream: [anomalyco/opencode](https://github.com/anomalyco/opencode) (MIT).
> The bionic build is maintained by [@bd-loser](https://github.com/bd-loser).

---

## Why `npm install -g opencode` doesn't work on Termux

opencode's TUI is powered by [opentui](https://opentui.com) — a native Zig library.
Upstream `@opentui/core` only ships **darwin / linux / win32**, and the Linux build is
**glibc-linked**. Android's Bionic linker rejects it outright.

The bionic build solves this with three pieces:

| Piece | What it does | Repo |
|---|---|---|
| Patched **Bun** runtime | Bun 1.3.14 patched for Bionic (FFI, TinyCC, MTE, SELinux fixes) | [bd-loser/bun-termux](https://github.com/bd-loser/bun-termux) |
| **opentui for Android** | `libopentui.so` rebuilt for `android-arm64` (Bionic ABI, 16 KiB page-size aligned) — published as `@androidtui/*` | [bd-loser/opentui](https://github.com/bd-loser/opentui) |
| **opencode delta** | Patches upstream opencode to use `@androidtui/*`, packaged as a Termux `.deb` | [bd-loser/opencode-bionic](https://github.com/bd-loser/opencode-bionic) |

The result is a single self-contained `opencode` binary compiled by Bun, installed to
`/data/data/com.termux/files/usr/lib/opencode/runtime/opencode` with a launcher at
`$PREFIX/bin/opencode`.

---

## Install

```bash
pkg update && pkg upgrade

# One command (downloads latest stable .deb, verifies SHA256, installs via dpkg)
curl -fsSL https://raw.githubusercontent.com/bd-loser/opencode-bionic/main/install.sh | bash

# Verify
opencode --version
```

Installer options (env vars):

```bash
OPENCODE_CHANNEL=prerelease curl -fsSL .../install.sh | bash   # dev builds
OPENCODE_VERSION=1.18.15 curl -fsSL .../install.sh | bash      # exact version
```

Manual install (no curl|bash):

```bash
curl -LO https://github.com/bd-loser/opencode-bionic/releases/latest/download/opencode_<version>_aarch64.deb
dpkg -i opencode_<version>_aarch64.deb
```

## Update

Same as install — the installer replaces the binary and launcher in place:

```bash
curl -fsSL https://raw.githubusercontent.com/bd-loser/opencode-bionic/main/install.sh | bash
opencode --version
```

After a version bump, also refresh the config plugin package if you use one:

```bash
cd ~/.config/opencode && npm install @opencode-ai/plugin@<new-version> --save
```

> The project polls upstream every 12h and publishes matching stable releases automatically —
> releases track upstream version-for-version (verified update path: 1.15.10 → 1.18.19).

## Uninstall

```bash
dpkg -r opencode
```

## Login / auth

```bash
opencode auth login          # interactive provider picker
opencode auth list           # show logged-in providers
```

Supported: Anthropic, OpenAI, OpenRouter, Google, local/OpenAI-compatible endpoints — same as
upstream. Credentials are stored in `~/.local/share/opencode/auth.json`.

## Configuration

Config lives in `~/.config/opencode/opencode.json` (see the [sample](./opencode.json) in this
repo). Minimal example:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-5",
  "small_model": "anthropic/claude-haiku-4-5",
  "snapshot": false
}
```

Full config reference: https://opencode.ai/docs/config

## Termux-specific tips

**Extra keys** — the TUI needs arrows, Enter, and text input. Merge the `extra-keys`
section of [`termux.properties`](./termux.properties) into `~/.termux/termux.properties`,
then `termux-reload-settings`:

- `← ↓ ↑ →` for navigation, `KEYBOARD` toggles the soft keyboard, `DRAWER` opens the
  Termux sidebar, `-` swipe-up gives `|`.

**On-screen keyboard** — keep mouse mode ON so touch taps still work (click-to-expand,
answering questions, selecting items). With mouse capture active, Termux sends taps as
clicks and does not auto-open the keyboard — pop it up with the `KEYBOARD` extra key
or `Ctrl+Alt+K`. Copy [`tui.json`](./tui.json) to `~/.config/opencode/tui.json` to make
mouse mode explicit:

```bash
mkdir -p ~/.config/opencode
cp tui.json ~/.config/opencode/tui.json
```

Disabling mouse mode instead (`"mouse": false`) makes any tap open the keyboard, but
then taps can no longer click things in the TUI.

**Storage access** — run `termux-setup-storage` to work with `~/storage` (downloads,
shared folders), then open opencode in those directories.

**Termux:API** (`pkg install termux-api`) — useful for share/paste integration with agents.

**Working directory** — opencode runs on whatever directory you launch it in. Launch it
inside your project: `cd ~/project && opencode`.

## How it works (architecture)

```
install.sh
  └─ resolves latest stable release (or pinned version/channel)
  └─ downloads opencode_<ver>_aarch64.deb
  └─ verifies SHA256SUMS
  └─ dpkg -i
       ├─ $PREFIX/bin/opencode            (bash launcher)
       └─ $PREFIX/lib/opencode/runtime/opencode  (bun-compiled bionic aarch64 binary)
```

The launcher sets `OPENCODE_DISABLE_DEFAULT_PLUGINS=1`, unsets `LD_PRELOAD`, cleans stale
locks, and restores the terminal (stty/tput) on exit. The binary is `ELF aarch64`,
`interpreter /system/bin/linker64`, built with NDK r29 — i.e. a normal Android executable,
so it runs in plain Termux with zero hacks.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `opencode: runtime not found` | reinstall: run the installer again |
| `dlopen failed: not a valid ELF` / glibc `.so` errors | you're on the npm install path — remove it (`npm rm -g opencode`) and use the bionic build |
| TUI renders garbage | `export TERM=xterm-256color`, restart |
| terminal stuck after exit | the launcher runs `stty sane`; if killed hard, run `reset` |
| `opencode auth login` fails in non-TTY | run it inside the Termux app terminal |
| Permission errors on `.git` operations | grant Termux storage permission in Android settings (Android 11+ "All files access" for `~/storage`) |
| upgrade broke something | reinstall the previous version: `OPENCODE_VERSION=<old> curl -fsSL .../install.sh \| bash` |

## Files

- [`opencode.json`](./opencode.json) — sample config
- [`tui.json`](./tui.json) — keeps mouse mode on so touch taps click; keyboard via KEYBOARD key
- [`termux.properties`](./termux.properties) — extra-keys block for the TUI

## Credits

- [opencode](https://github.com/anomalyco/opencode) — the agent itself (MIT)
- [bd-loser/opencode-bionic](https://github.com/bd-loser/opencode-bionic) — bionic builds & installer
- [bd-loser/bun-termux](https://github.com/bd-loser/bun-termux), [bd-loser/opentui](https://github.com/bd-loser/opentui) — the Termux enablement pieces
