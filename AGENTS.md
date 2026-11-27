# Notes for AI coding agents

Linux tools for the ASUS ROG Pelta headset (2.4 GHz dongle `0b05:1b84`, USB-C `0b05:1b82`). The README is the user documentation and has the full protocol tables under **How it works → Protocol**; read it before changing anything that talks to the headset.

## Layout

- `bin/pelta-ctl`: command-line tool **and** the shared library. Every headset command lives here.
- `bin/pelta-tray`: PyQt6 tray icon and settings window. Loads `pelta-ctl` from its own folder and calls its functions; it never builds HID reports itself.
- `bin/pelta-wake-hid`: user daemon (`systemd/pelta-wake.service`). Wake/reconnect volume fix, battery, charging, connection mode. Writes state to `$XDG_RUNTIME_DIR/pelta.json`, which the tray reads.
- `bin/pelta-recover`: re-syncs the hardware volume (nudge, then restore the exact raw value).
- `udev/70-pelta.rules`: `/dev/pelta-hid-dongle` and `/dev/pelta-hid-usb` (HID interface 3), user access via `uaccess`.
- `install.sh`: installs to `~/.local/bin`, the user service, the udev rule and desktop entries; `--uninstall` reverses it.
- `tools/pelta-probe`: sends the status queries and prints raw replies. Not installed.

## Protocol essentials

- Report ID `0xCC`, 64 bytes each way. Byte 1 = command (`12` get, `51` set, `41` audio set, `61` audio level), byte 2 = setting key, bytes 3–4 = index, data from byte 5.
- A get is answered with the same first bytes. `ff aa` in bytes 5–6 means the headset can't answer (asleep or off).
- Multi-byte versions are least-significant first (`00 07 00 04` = 4.0.7.0).
- 2.4 GHz quality (`52`) is a dongle setting: send it to the dongle even when the cable is preferred.
- Switching the headset off/on sends no report; the daemon detects it by polling `cc 12 00 00 01` on the dongle.

## Rules

- **Only send commands that are known**: from Armoury Crate captures or the command table in ASUS Gear Link. Don't guess bytes on the set commands (`51`, `41`, `61`); an unknown set can change an unrelated setting.
- **Never add** the reset commands (`cc 50 40`, `cc 50 60`) or firmware flashing.
- After a write that can be read back, read it back and report a mismatch (see `write_power`, `write_quality`).
- Keep everything as standalone Python 3 with the standard library only (PyQt6 for the tray). No new dependencies, no build step.
- New headset commands go into `pelta-ctl` first; the tray calls them through `run()`.
- Update the README with every user-facing change: the command list, the settings-window table, and the protocol tables.

## Testing

There's usually no headset in an agent's environment. Test logic by loading a script with `importlib` and replacing `get`/`send` (or the daemon's `handle` inputs) with a fake headset built from real replies, as the tables in the README show. State plainly what wasn't tested on hardware.

Before merging, a person should check on a real headset: wake from sleep, switching off and on, USB ↔ 2.4 GHz switching, and that the headset still goes to sleep on its own.

## Workflow

- `main` is protected (no force-push or deletion). Work on a branch and open a pull request.
- Commit messages: a short summary line, then what changed and why.
