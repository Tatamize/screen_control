# screen_control

Runs two Chrome kiosk windows (one per monitor) as systemd user services, and
works around a ghost/phantom monitor issue on OnLogic devices where a
disconnected output (e.g. `DP-3`) is still detected and steals a browser
window.

## Files

Three unit files live in `systemd/user/`:

- `xrandr-fix.service` — disables the ghost monitor before the kiosk windows start
- `chrome-kiosk-screen1.service` — Chrome kiosk window for monitor 1
- `chrome-kiosk-screen2.service` — Chrome kiosk window for monitor 2

## Installation

1. Copy all three files into `~/.config/systemd/user/`.
2. Reload systemd and enable the services:

   ```bash
   systemctl --user daemon-reload
   systemctl --user enable xrandr-fix.service chrome-kiosk-screen1.service chrome-kiosk-screen2.service
   systemctl --user restart xrandr-fix.service chrome-kiosk-screen1.service chrome-kiosk-screen2.service
   ```

## First-time setup on a new device

Monitor names (`DP-1`, `DP-2`, ...) and their physical layout can differ
between devices, so **don't assume the existing `--window-position` values
are correct**. Check them on every new device before relying on the kiosk
services.

### 1. Check which output is the ghost monitor

After connecting both monitors, run:

```bash
xrandr --query
```

Look for an output that reports `connected` but doesn't correspond to a real
screen (no image, or a resolution that doesn't match either monitor). That's
the ghost monitor. Update the `--output` value in `xrandr-fix.service` if it
isn't `DP-3` on this device.

### 2. Check real resolution and position of each monitor

Run `xrandr --query` again — the ghost monitor should now be excluded once
`xrandr-fix.service` has run. Each real, connected monitor shows a line like:

```
DP-1 connected 1920x1080+0+0 ...
DP-2 connected 1920x1080+1920+0 ...
```

The `WxH+X+Y` part gives you the values to use:

- `WxH` → `--window-size`
- `+X+Y` → `--window-position`

So in the example above: `DP-1` → `--window-position=0,0 --window-size=1920,1080`,
and `DP-2` → `--window-position=1920,0 --window-size=1920,1080`.

### 3. Identify which output is which physical monitor

Output names don't tell you left vs. right. To check, dim one monitor and see
which one physically changes:

```bash
xrandr --output DP-1 --brightness 0.3
```

Restore it afterward:

```bash
xrandr --output DP-1 --brightness 1.0
```

Match each output name to "screen1" (left) or "screen2" (right), then set
`--window-position` and `--window-size` in the corresponding service file to
the values found in step 2.

### 4. Apply changes

After editing the service files:

```bash
systemctl --user daemon-reload
systemctl --user restart chrome-kiosk-screen1.service chrome-kiosk-screen2.service
```

## Notes

- `chrome-kiosk-screen1.service` and `chrome-kiosk-screen2.service` each use
  a separate `--user-data-dir` (`.chrome-kiosk1` / `.chrome-kiosk2`). Chrome
  locks its profile directory, so both windows must not share one.
- Both kiosk services use `Restart=on-failure` with `RestartSec=5` to recover
  automatically if Chrome crashes, and `StartLimitIntervalSec=300` /
  `StartLimitBurst=10` to stop retrying if it keeps failing. If a service
  hits this limit, clear it and restart manually:

  ```bash
  systemctl --user reset-failed chrome-kiosk-screen2.service
  systemctl --user restart chrome-kiosk-screen2.service
  ```