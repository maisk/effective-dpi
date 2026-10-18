# effective-dpi

A command-line tool for Linux (X11, and GNOME on Wayland) that prints the **effective DPI** of a
monitor: how many device pixels are drawn per real inch of glass. A web page needs this number
to show a document at its physical size (10 cm on paper = 10 cm on screen), and the browser
cannot get it by itself: no web API exposes the physical size of the screen.

Next to it, `effective-dpi-calibrate` measures the real size of the screen with a bank card,
because the size the monitor reports about itself can be wrong.

Python 3, standard library only, one file each. No build, no install, no root.

## The idea

```
effective DPI = drawn pixels / physical size in inches
```

From it the rest follows:

```
CSS ppi        = effective DPI / devicePixelRatio
CSS px for L   = L(cm) / 2.54 × CSS ppi          (what goes into CSS)
device px      = L(cm) / 2.54 × effective DPI    (pixels the image file needs)
panel ppi      = panel pixels / physical inches  (sharpness of the glass, informative)
```

The effective DPI does not change with browser zoom (zoom changes the DPR and the CSS ppi, not
the device pixels per inch), so it is the number to store.

### Drawn pixels are not panel pixels

With fractional scaling, what an application draws can differ from what the panel shows, and
the effective DPI is about what the application draws. Take a 2560 × 1440 panel set to 125% in
GNOME:

| Session | What the browser draws | Effective DPI on a 23.8″ panel | DPR |
|---|---|---|---|
| X11 | 4096 × 2304, then downscaled ×1.6 to the panel | 197 | 2 |
| Wayland | 2560 × 1440, the panel pixels themselves | 123 | 1.25 |

The CSS ppi is the same in both (about 99), so the page looks the same size; the device pixels
per inch are not.

- **X11**: the drawn pixels are the monitor's area in the X screen (its CRTC geometry in
  `xrandr`). Every X11 application, the browser included, draws into exactly these pixels.
  X11 has one UI scale for the whole screen (`Xft.dpi / 96`), so GNOME draws everything at an
  integer scale (×2 when a monitor is at 125%) and shrinks each monitor with an xrandr
  transform. A monitor with a fractional scale therefore makes *every* monitor draw at that
  integer scale, a 100% monitor next to it included. Fractional scaling on GNOME X11 needs
  Mutter's experimental feature `x11-randr-fractional-scaling` (on Ubuntu, the "Fractional
  Scaling" switch in Settings → Displays); without it X11 offers only integer scales.
- **Wayland** (GNOME): each monitor has its own scale, and the drawn pixels depend on the
  client. The tool reads Mutter's `GetCurrentState` over D-Bus and shows every case:
  - `native` (the default): a client with fractional scaling (`wp_fractional_scale_v1`, as
    current Firefox and Chrome) draws **the panel pixels**; DPR = the monitor's scale.
  - `integer`: a client without fractional scaling draws at `ceil(scale)` and the compositor
    downscales, as on X11; DPR = `ceil(scale)`. Only for a fractional scale.
  - `xwayland`: an X11 application draws what XWayland's `xrandr` shows for the output
    (on GNOME 46, 2048 × 1152 for the panel above: drawn smaller, then upscaled); DPR =
    XWayland's `Xft.dpi / 96`.

### Physical size: the EDID can be wrong

The monitor reports its size in its EDID. That is often right, sometimes rounded to whole
centimetres, and sometimes wrong. So a real size takes precedence when one is known:
`--size-mm` (from the spec sheet), else a bank-card calibration saved by
`effective-dpi-calibrate`, else the EDID. The output always says which one it used.

## Example

An LG 24MP57VQ-P, a 23.8″ monitor whose EDID says 480 × 270 mm (a 21.7″ panel). With the EDID
alone:

```
$ effective-dpi --all --no-calibration
HDMI-2  LG IPS FULLHD  [primary]
  physical size   480 × 270 mm  (21.7″ diagonal, EDID)
  panel           1920 × 1080 px → 101.6 ppi
  drawn surface   1920 × 1080 px
  effective DPI   101.6
  DPR             1  (Xft.dpi 96; the browser at 100% zoom)
  CSS ppi         101.6
  10 cm           = 400.0 CSS px = 400 device px
  ⚠ the size is in whole centimetres, so it may be rounded (up to ±0.5 cm, about ±1%)
```

After measuring it with a bank card in `effective-dpi-calibrate`:

```
$ effective-dpi --all
HDMI-2  LG IPS FULLHD  [primary]
  physical size   523.7 × 294.6 mm  (23.6″ diagonal, card calibration 2026-10-18)
  EDID size       480 × 270 mm  (with it: 101.6 dpi)
  panel           1920 × 1080 px → 93.1 ppi
  drawn surface   1920 × 1080 px
  effective DPI   93.1
  DPR             1  (Xft.dpi 96; the browser at 100% zoom)
  CSS ppi         93.1
  10 cm           = 366.7 CSS px = 367 device px
  ⚠ with the EDID size everything would come out 9.1% too large: the EDID is wrong for this monitor
```

With the EDID size, "10 cm" would have been drawn as 400 px, which is 10.9 cm on this glass.
Nothing in the EDID gives this away; only a card or the spec sheet does.

On Wayland the output adds the monitor's scale, the client case it describes, and the other
cases. For the 2560 × 1440 panel at 125% from the table above:

```
  Wayland scale   1.25
  client          native Wayland with fractional scaling: draws the panel pixels
  ...
  other clients   integer: 4096 × 2304 px → 197.4 DPI, DPR 2, CSS ppi 98.8  (--client integer)
                  xwayland: 2048 × 1152 px → 98.7 DPI, DPR 1, CSS ppi 98.8  (--client xwayland)
```

## Requirements

| Need | Why | Package (Debian/Ubuntu) |
|---|---|---|
| **X11 session, or GNOME on Wayland** | Other Wayland compositors (KDE, wlroots) are not read yet. | – |
| **Python ≥ 3.8** | the tools; standard library only | `python3` |
| **`xrandr`** | X11: geometry (drawn pixels), current/preferred mode (panel pixels), physical mm, EDID hex. Wayland: XWayland's view of each output, for the `xwayland` case only. | `x11-xserver-utils` |
| **`busctl`** *(Wayland)* | Mutter's `org.gnome.Mutter.DisplayConfig.GetCurrentState` as JSON | `systemd` |
| **`/sys/class/drm/*/edid`** *(Wayland)* | the EDID, matched to Mutter's monitors by maker/product/serial; readable without root | – |
| `xdotool` *(optional, X11)* | pointer position → which monitor is "current". Without it, and always on Wayland: the primary. | `xdotool` |
| `xrdb` *(optional)* | `Xft.dpi / 96` → the browser's `devicePixelRatio` at 100% zoom, for the CSS ppi line. On Wayland only for the `xwayland` case. | `x11-xserver-utils` |
| `python3-tk` *(calibrator only)* | the calibration window | `python3-tk` |

`edid-decode /sys/class/drm/card1-HDMI-A-1/edid` is handy for checking by hand, but the tool
decodes the few EDID bytes it needs itself.

## Usage

```
effective-dpi                 # the monitor under the pointer (else, and on Wayland, the primary)
effective-dpi --all           # every active monitor
effective-dpi --output HDMI-2 # one output by name
effective-dpi --client xwayland               # Wayland: the pixels of an X11 app (also: integer)
effective-dpi --dpr 1.5       # assume another devicePixelRatio for the CSS lines
effective-dpi --cm 21         # worked example length (default 10 cm)
effective-dpi --json          # machine-readable
effective-dpi --output HDMI-2 --size-mm 527x296   # the real size, e.g. from the spec sheet
effective-dpi --no-calibration                    # ignore saved card calibrations (EDID only)
```

The `--json` rows carry the same information: `session` (`x11` or `wayland`), `output`,
`primary`, `name`, `maker`, `drawn_px`, `position`, `panel_px`, `size_mm`, `size_source`,
`warnings`, `diagonal_in`, `effective_dpi` [h, v], `panel_ppi`, `scale` (drawn / panel), `dpr`,
`css_ppi` and `example`. When the size does not come from the EDID, also `edid_size_mm` and
`edid_dpi`. On Wayland also `client`, `wayland_scale` and `other_clients` (`client`,
`drawn_px`, `position`, `effective_dpi`, `dpr`, `css_ppi` for each other case). `position` is
in Mutter's layout coordinates, except for `xwayland`, where it is XWayland's.

## Calibration with a bank card (`effective-dpi-calibrate`)

`effective-dpi-calibrate` is a small Tk window. It runs `effective-dpi --all --json
--no-calibration` and draws an ID-1 card outline (85.60 × 53.98 mm, 3.18 mm corners) at the
size the EDID predicts, in device pixels, on the monitor that holds the window. On Wayland Tk
runs under XWayland, so it asks for `--client xwayland` and works in XWayland's pixels. Hold a
real bank card against the glass and grow or shrink the outline until it matches:

- the outline's top-left corner stays fixed (put the card's corner on it once) and only the
  right and bottom edges move
- buttons: both / width / height, ±0.1% and ±1%
- keys: ←/→ width, ↓/↑ height, −/+ both (0.1%); Shift or PgUp/PgDn for 1%; Ctrl+S saves;
  Esc quits
- it shows the EDID DPI, the current DPI and the difference in %, the CSS ppi, and where the
  value came from (EDID / saved calibration / adjusted by hand)
- it warns when width and height disagree by more than 2.5%, when the value is more than 2.5%
  from the EDID, or when the window spans two monitors
- moving the window to another monitor switches to that monitor, and each monitor keeps its
  own value

Saving writes the value per monitor (output + EDID maker + product name) to
`~/.config/effective-dpi/calibration.json` and prints it as one JSON line. For the LG above,
on X11 at 100%, a record looks like this:

```json
{"output": "HDMI-2", "maker": "GSM", "name": "LG IPS FULLHD", "effective_dpi": [93.13, 93.13],
 "drawn_px": [1920, 1080], "size_mm": [523.7, 294.6], "edid_dpi": [101.6, 101.6], "dpr": 1.0,
 "saved": "2026-10-18T14:35:42"}
```

`size_mm` is the real size of the glass that the card measured. `effective-dpi` uses it from
then on, so the result stays right when the scaling changes or you switch between X11 and
Wayland. The DPI in the record belongs only to the surface it was measured on.

## Limits (what can make the number wrong)

- **EDID size wrong or missing.** TVs, projectors and some adapters report 0 or a nonsense
  size. The tool warns (no size; diagonal outside 7–120″). A plausible size can be wrong too,
  as in the example above. With a calibration or `--size-mm`, the tool warns when the EDID is
  more than 2.5% off.
- **EDID size rounded.** Some monitors report whole centimetres (e.g. `480 x 270`): up to
  ±0.5 cm, about ±1%. The tool warns when both values are multiples of 10 mm and the size comes
  from the EDID.
- **DPR is a guess.** On X11 it is `Xft.dpi / 96`, which GNOME sets for its UI scale and the
  browser follows by default; on Wayland it is the monitor's scale. A browser started with its
  own scale (Chrome `--force-device-scale-factor`, Firefox `layout.css.devPixelsPerPx`)
  differs: pass `--dpr`. Only the CSS-ppi lines depend on it; the effective DPI does not.
- **Wayland: the client decides.** The tool cannot see how the browser runs. Compare
  `screen.width × devicePixelRatio` in the browser with the drawn surface of each case
  (Firefox `about:support` → "Window Protocol"; Chrome `chrome://gpu` → "Ozone platform").
  GNOME only; KDE (`kscreen-doctor -j`) is not done.
- **X11 and Wayland values differ.** The same monitor has another effective DPI in each
  session (see the table above), so a stored DPI belongs to one session. The physical size and
  the CSS ppi stay the same. On X11 the monitors are also coupled: adding or removing a monitor
  with a fractional scale changes the drawn pixels of the others.
- **A calibration belongs to a port.** It is keyed by output, maker and name. Plug the monitor
  into another port and the calibration is not found, so the tool falls back to the EDID.
- **Rotated outputs**: handled by matching orientation (mm and mode are swapped to match the
  drawn geometry).

## Cross-check with the browser

- `screen.width × devicePixelRatio` in the browser must equal the tool's **drawn surface**. If
  not, the browser is on another monitor, runs as another client (Wayland), or uses its own
  scale.
- A square of N device pixels, with N = the tool's effective DPI, should measure one inch
  (2.54 cm).

[maistrelis.com/en](https://maistrelis.com/en/) has pages for both,
[Your screen resolution](https://maistrelis.com/en/tools/screen_info/) and
[Pixel test](https://maistrelis.com/en/tools/pixel_grid/), and
[Calculate your monitor's DPI](https://maistrelis.com/en/tools/monitor_dpi/) for a card
calibration in the browser.

## Prior art

No tool found does the same job: the building blocks (EDID size via `xrandr`, `px × 25.4 / mm`)
are well known, but none measures the drawn surface or turns it into CSS units. See
[`prior-art.md`](prior-art.md).

## Not done

- Wayland on KDE (`kscreen-doctor -j`) and wlroots compositors.
- A browser bridge: a native-messaging host wrapping this tool, plus an extension that gives the
  value only to allow-listed origins, so a page can size a document without a card.

## Credits

Written by [Kostas Maistrelis](https://maistrelis.com/en/). The code and the documentation
were developed in collaboration with [Claude](https://www.anthropic.com/claude), Anthropic's
AI model, using Claude Code.

## License

MIT, see [`LICENSE`](LICENSE).
