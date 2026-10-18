# Prior art: what already exists

Researched in October 2026 by searching the web. Question: is `effective-dpi` reinventing the
wheel?

**Short answer: no tool found does the same job.** The building blocks are well known (EDID
size via `xrandr`, `px × 25.4 / mm`). No tool found does these two things:

1. Measure the **drawn surface** (CRTC geometry on X11, the client's buffer on Wayland) instead
   of the panel mode. For a 2560 × 1440 panel at 125% on GNOME X11 this is the difference
   between 197 and 123 DPI.
2. Turn the result into what a web page needs: DPR, CSS ppi, and "L cm = N CSS px".

## Command-line tools (Linux)

| Tool | What it gives | Gap vs effective-dpi |
|---|---|---|
| `xrandr` | Per-output geometry, modes, physical mm (from the EDID DTD), EDID hex | Raw data only, no DPI. This is the X11 input. |
| `edid-decode` | Full EDID decode: base-block size (cm), DTD sizes (mm), product name | No DPI, no link to the X screen. Useful for checking by hand. |
| `inxi -Gaz` (`--vs` on some versions) | Per-monitor `res`, `dpi`, `size` (mm and in), `diag`, plus a screen-wide `s-dpi` | Its `dpi` appears to be **panel ppi** (mode / mm), so it would not show the fractional-scaling drawn surface. *Not verified.* Some older versions reject `--vs`. |
| `xdpyinfo` | One DPI for the whole X screen | Global, usually 96 or `Xft.dpi`. Not per monitor, not physical. |
| `wlr-randr --json` | Output state on wlroots compositors (Sway etc.) | wlroots only, not GNOME/KDE. Unclear whether it includes mm. |
| `kscreen-doctor -j` | KDE output state (mm, mode, scale) | KDE only. No DPI. A possible data source for KDE support. |
| `gdbus` / `busctl` → `org.gnome.Mutter.DisplayConfig.GetCurrentState` | GNOME monitors, modes, logical scale | Data source only, no DPI and no mm. `effective-dpi` uses it on GNOME Wayland. |

Sources: [Arch forum: xrandr mm vs EDID](https://bbs.archlinux.org/viewtopic.php?id=256884),
[autorandr #33: EDID bytes 21–22, per-display DPI in X](https://github.com/phillipberndt/autorandr/issues/33),
[NVIDIA README: DPI from EDID, EDID can be unreliable](https://http.download.nvidia.com/solaris/470.103.01/README/dpi.html),
[Trinity list: `inxi -Gaz --vs`](https://mail.trinitydesktop.org/mailman3/hyperkitty/list/users@trinitydesktop.org/message/OXNN3H3K7HWR4PNPVZKZBU45ZREDO7XW),
[Manjaro forum: inxi monitor lines, bogus EDID sizes](https://forum.manjaro.org/t/testers-needed-manjaro-data-donor/170163/77),
[openSUSE forum: inxi `s-dpi` vs monitor `dpi`](https://forums.opensuse.org/t/kde-super-size-fonts-and-objects-in-2560x1440-laptop-screen/141937?page=2),
[wlr-randr man page](https://man.archlinux.org/man/wlr-randr.1.en).

## Fractional-scaling helpers (X11)

These **set** scaling (`xrandr --scale`, framebuffer tricks). None of them **measure** the
resulting DPI.

- [hidpi-fixer](https://github.com/alex-spataru/hidpi-fixer): generates xrandr scaling scripts.
- [Ubuntu Budgie fractional HiDPI on X11](https://discourse.ubuntubudgie.org/t/ubuntu-budgie-20-04-fractional-hidpi-for-x11/4777):
  scale ×2, then `xrandr --scale 1.5x1.5`. This is the same mechanism GNOME uses.
- [xorg-fractional-scaling COPR](https://copr.fedorainfracloud.org/coprs/lukajan/xorg-fractional-scaling/).
- [tych0's gist](https://gist.github.com/tych0): an xrandr script that matches DPI across
  monitors. Reference code only.

## Physical size in the browser

- **Calibration with a card or ruler** ([Screen Ruler](https://screen-ruler.utils.com/) and
  many similar pages): the user matches an on-screen bar to an 85.60 mm ID-1 card. Manual,
  needs redoing after zoom changes, no OS data.
- **[lifesized](https://npmjs.com/package/lifesized)** ([source](https://github.com/francoislaberge/lifesized)):
  an Electron module with the same goal (objects at real size, `ppi()` and `scale()`). Its
  README says "Only accurate on OSX"; on Linux and Windows it returns a 72 ppi default. Small
  and apparently inactive.
- **`chrome.system.display`**: `dpiX`/`dpiY` are 96 × the scaling factor on Windows and unset
  on Linux/macOS (Chromium source,
  `chrome/browser/extensions/system_display/display_info_provider_win.cc`), so not usable.
- **Standards**: there is no web API for physical screen size. Discussions stall because the
  OS often does not know the exact size, and because of fingerprinting concerns.
  [W3C CSS list (2020)](https://lists.w3.org/Archives/Public/public-css-archive/2020Apr/0125.html),
  [WICG "One inch is not an inch"](https://discourse.wicg.io/t/one-inch-is-not-an-inch/1228),
  [WHATWG (2010) on zoom / DPR](https://lists.whatwg.org/pipermail/whatwg-whatwg.org/2010-November/029072.html).

No browser extension with a native-messaging host that exposes the OS display size was found.

## Wayland

- On Wayland, GTK's `gdk_monitor_get_width_mm()` was reported to return 0, while XWayland still
  reported a size ([GNOME Discourse](https://discourse.gnome.org/t/gdk-monitor-get-width-mm-failure-wayland/5412)).
- Mutter has had reports of wrong physical sizes on Wayland, and the EDID DTD (mm) is more
  precise than the base block (cm) ([mutter #2812](https://gitlab.gnome.org/GNOME/mutter/-/issues/2812)).
  Mutter's `GetCurrentState` gives no mm at all, and XWayland's `xrandr` rounds them to whole
  centimetres. So `effective-dpi` reads the EDID from `/sys/class/drm/*/edid` (readable without
  root) and takes the DTD size.
- `xrandr` on X11 already reports the DTD size, not the base block's centimetres.

## Why nothing is built in

- CSS 2.1 (around 2010) fixed `1in = 96px`. The proposed `truemm`/`truept` units were rejected
  on www-style. Firefox's `mozmm` shipped in 2010 and was removed in Firefox 59 (2018).
- Screen properties are a fingerprinting vector, so browsers expose less and less. A 2019
  request to expose EDID data went nowhere.

The OS knows the size, and native apps get it through an API (Windows
[`GetDpiForMonitor(MDT_RAW_DPI)`](https://learn.microsoft.com/en-us/windows/desktop/api/ShellScalingAPI/nf-shellscalingapi-getdpiformonitor),
Qt `QScreen::physicalDotsPerInch`, GTK `gdk_monitor_get_width_mm`, macOS
`CGDisplayScreenSize`). A web page cannot get it. So the gap is exactly "OS knowledge → web
page". PDF viewers rely on the same OS data or on a manual setting: Evince and Okular follow
xrandr's EDID size ([Linux Mint forum](https://forums.linuxmint.com/viewtopic.php?t=344638)),
and Acrobat has a manual resolution preference
([Adobe community](https://community.adobe.com/t5/acrobat-discussions/how-does-acrobat-decide-what-is-default-resolution/m-p/12696451)).

## Where effective-dpi fits

A page can solve the problem **inside the browser**, with a card or ruler calibration that it
stores (for example in `localStorage`). `effective-dpi` solves the same problem **from the OS
side** (EDID + drawn surface), with a card calibration only where the EDID is wrong. The two
are independent measurements of the same number, so each checks the other.

Its value lies in two things: it measures the drawn surface instead of the panel, and it
converts the result to CSS units. If `inxi` turns out to report the drawn surface already,
that belongs in the README.
