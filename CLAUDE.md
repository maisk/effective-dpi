# CLAUDE.md

This file guides Claude Code (claude.ai/code) when it works in this repository.

## What this is

`effective-dpi` is a single-file Python 3 CLI (stdlib only, no build, no dependencies). It
prints a monitor's **effective DPI**: the device pixels drawn per real inch of glass, i.e. the
drawn surface divided by the physical size. On X11 the drawn surface is the monitor's CRTC
geometry from `xrandr`; on GNOME Wayland it depends on the client (see below). The physical
size comes from `--size-mm`, else a saved card calibration, else the EDID. Next to it is
`effective-dpi-calibrate`, a Tk window for measuring the real size with a bank card. The
README explains the idea, the formulas and the limits; `prior-art.md` covers what exists.

Terms and formulas: effective DPI = CSS ppi × DPR. Store the effective DPI (it does not change
with browser zoom), not the CSS ppi.

## Running / checking

There are no tests. To check a change, run it against the live session:

```
./effective-dpi                      # monitor under the pointer (Wayland: the primary)
./effective-dpi --all                # every monitor, saved calibrations applied
./effective-dpi --all --no-calibration
./effective-dpi --output HDMI-2 --size-mm 527.04x296.46
./effective-dpi --all --json
./effective-dpi --all --client xwayland        # Wayland only; also: integer
python3 -m py_compile effective-dpi effective-dpi-calibrate && rm -rf __pycache__
```

A change to the output format must also be made in the README ("Example", "Usage", the JSON
field list). The README's examples are real output; keep them so.

Parsers can be checked without the matching session by calling them directly: load the script
with `importlib.machinery.SourceFileLoader`, then call `parse_xrandr()` on saved
`xrandr --verbose` text, or `parse_mutter()` on the `data` of saved `busctl --json=short`
`GetCurrentState` output plus EDID hex. `describe()` takes the resulting dicts.

## Structure of `effective-dpi` (one file, top to bottom)

- `parse_xrandr`: parses `xrandr --verbose`. Only *connected and active* outputs, meaning those
  with a geometry in their header line, become monitors. It also reads the current and
  preferred modes and gathers the EDID hex lines.
- `edid_info`: decodes the maker id, product code, serial (tag 0xFF text, else the number as
  `0x%08x`, as Mutter does), the product name (tag 0xFC), the size in cm and the first DTD's
  size in mm, from the 128-byte base block.
- Wayland: `sysfs_edids` reads `/sys/class/drm/card*-*/edid`. `match_edid` matches one to a
  Mutter monitor by maker/product/serial, never by connector name (sysfs says `HDMI-A-1`,
  Mutter `HDMI-1`). `parse_mutter` turns the `GetCurrentState` data into the same dicts as
  `parse_xrandr`, plus `scale`. Its `geom` is the panel px at the layout position, and `mm`
  comes from the DTD (Mutter gives no mm). `wayland_clients` gives each client case as
  (geom, dpr): `native` (panel px, DPR = scale), `integer` (`ceil(scale)`, only when the scale
  is fractional), and `xwayland` (from XWayland's `xrandr` and `Xft.dpi`).
- `load_calibration`: reads the calibrator's JSON. Missing or broken means `{}`.
- `describe`: picks the size (`--size-mm` > calibration > EDID), does the arithmetic and builds
  the result dict, which is also the `--json` output. `oriented()` swaps mm and mode for
  rotated outputs. Warnings: no size, diagonal outside 7–120″, whole-cm size (EDID only), and
  the EDID more than 2.5% off the real size.
  - A calibration is used as a **size** (`size_mm`), not a DPI, so it survives scaling changes
    and the switch between X11 and Wayland. Records without `size_mm` fall back, in
    `calibrated_size`, to the record's `drawn_px / effective_dpi`, else to EDID mm ×
    `edid_dpi / effective_dpi`, else to the current drawn px.
- `print_row`: prints the human-readable output.
- `main`: handles the arguments and picks a monitor: `--output` first, then `--all`, then the
  monitor under the pointer (`xdotool`), then the primary. On Wayland there is no pointer
  position, so it takes the primary. `--size-mm` requires `--output`. On Wayland it describes
  every client case: the `--client` one (default `native`) is the row, and the rest go into
  `other_clients`. Every row gets `session` (`x11`/`wayland`).

## `effective-dpi-calibrate` (Tk card calibration)

- It does not import `effective-dpi`. It runs `effective-dpi --all --json --no-calibration
  --client xwayland` as a subprocess (the script next to it, else the one in `PATH`). On
  Wayland Tk runs under XWayland, so it needs XWayland's pixels; X11 ignores `--client`. The
  `--no-calibration` matters: its "EDID" baseline must not be its own earlier result. The JSON
  fields (`output`, `maker`, `name`, `drawn_px`, `position`, `effective_dpi` [h, v], `dpr`) are
  its interface. Changing them breaks the calibrator.
- The card is drawn in canvas pixels, which are X device pixels: the same surface the effective
  DPI describes. `tk scaling` (from the DPR) only sizes the UI text. The card's top-left corner
  is fixed (`self.origin`). The canvas is packed `anchor='w'` and only ever grows, so the corner
  does not move on screen.
- Saved data lives in `$XDG_CONFIG_HOME/effective-dpi/calibration.json`, keyed by
  `output|maker|name`, with `effective_dpi`, `drawn_px` and the implied `size_mm`.
- Testing: when started in the background, GNOME's focus-stealing prevention means the window
  gets no keyboard focus, so `xdotool key` and `event_generate` keys do not arrive. **Do not
  send global keys with xdotool**: they land in whatever window has focus. Test by loading the
  script with `importlib.machinery.SourceFileLoader` and calling `nudge()`, `save()` and
  `root.geometry()` directly, with `XDG_CONFIG_HOME` pointed at a temp directory. Some keyboard
  layouts change the keysym (Greek: Ctrl+S arrives as `Greek_sigma`), so both are bound.

## Conventions

- Keep each tool stdlib-only and a single file. The optional tools (`xdotool`, `xrdb`,
  `xrandr` on Wayland) must fail softly through `run()`, which returns `None`.
- The effective DPI must not depend on the DPR. Only the CSS-ppi and worked-example lines use
  the DPR (`Xft.dpi / 96`, the Wayland scale, or `--dpr`).
- Always say where the size came from (`size_source`). Never present an EDID guess as measured.
- Wayland (`XDG_SESSION_TYPE == wayland`) means GNOME/Mutter only. Read monitors from
  XWayland's `xrandr` only for the `xwayland` case: it has no EDID and rounds mm to cm.
