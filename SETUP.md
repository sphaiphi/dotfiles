# Sway config — setup summary

Fedora `sway-config-fedora` split-file layout as the base, with a curated
set of Ubuntu-Sway's good ideas layered on top — deliberately without
the background-daemon sprawl either project ships by default. Verified
against actual sway/Waybar source and both Fedora and Ubuntu 26.04 LTS
package listings, not just written from memory. Colors are a swappable
theme (Nord and Catppuccin Mocha both available, switchable with one
command) applied consistently across sway, Waybar, foot, and wofi.

## File tree and install locations

```
~/.config/sway/
├── config                        # entrypoint — includes everything below
├── themes/
│   ├── nord.conf                 # ALL sway/client color values live here
│   └── catppuccin.conf           # same variable names, different values
└── config.d/
    ├── 00-variables.conf         # includes active theme, then $mod/$term/
    │                             # $menu/$lock/$file_manager/workspaces
    ├── 10-appearance.conf        # fonts, borders, gaps, titlebars, client.* colors
    ├── 20-input.conf             # keyboard (us+th, Ctrl+Shift toggle), touchpad, gestures
    ├── 20-output.conf            # display defaults ($bg-themed), wallpaper, lid switch
    ├── 30-autostart.conf         # dbus-update-activation-environment, mako, polkit,
    │                             # swayidle, cliphist — see "Known fixes" below
    ├── 40-bindings-core.conf     # launch/focus/move/workspace/media keys
    ├── 50-modes.conf             # resize mode, screenshot mode
    ├── 60-window-rules.conf      # floating dialogs, shortcuts inhibitor
    └── 70-bar.conf               # waybar launch (+ native fallback)

~/.config/environment.d/
└── 10-sway.conf                  # session env vars — NOT included from sway config;
                                   # read by systemd at LOGIN time only (see below)

~/.config/waybar/
├── config.jsonc                  # modules: workspaces, mode, scratchpad, layout,
│                                  # idle_inhibitor, backlight, tray, language,
│                                  # network, volume, battery, clock
├── style.css                     # structure only — @imports active theme,
│                                  # zero raw hex values
└── themes/
    ├── nord.css                  # ALL waybar color values live here
    └── catppuccin.css

~/.config/foot/
├── foot.ini                      # [main] include=themes/<name>.ini
└── themes/
    ├── nord.ini                  # official Nord 16-color ANSI mapping
    └── catppuccin.ini            # official Catppuccin Mocha ANSI mapping

~/.config/wofi/
├── style.css                     # structure only — @imports active theme
└── themes/
    ├── nord.css
    └── catppuccin.css

~/.local/bin/
└── sway-theme                    # switches all four apps' active theme at once
```

`config.d.local/*.conf` (referenced but not created) is an optional
empty slot for host-specific overrides — safe to leave absent.

`lf` (the file manager, see "File manager" below) has no entry here
deliberately — it needs no theme file of its own; see that section.

## Install steps

```bash
# 1. Create the directories
mkdir -p ~/.config/sway/config.d ~/.config/sway/themes
mkdir -p ~/.config/environment.d
mkdir -p ~/.config/waybar/themes ~/.config/foot/themes ~/.config/wofi/themes
mkdir -p ~/.local/bin ~/Pictures/Screenshots

# 2. Copy each file to its path above
chmod +x ~/.local/bin/sway-theme

# 3. Log out and back in
#    (required once, specifically for environment.d — a sway reload
#    does NOT pick up environment.d changes, only a fresh login does)
```

## Switching themes

Run `sway-theme` with no arguments to see the current theme and what's
available; `sway-theme catppuccin` to switch. It edits the one
"pointer" line each app has (sway's `include`, Waybar/wofi's
`@import`, foot's `include=`) and applies the change live where
possible: sway reloads immediately, Waybar is killed and relaunched
(no hot-reload for `style.css` exists), foot/wofi pick it up
naturally since they aren't long-running daemons the same way — foot
specifically can't retheme windows already open, only new ones.

It fails closed: if a theme is missing even one of the four files it
needs, nothing gets switched.

To add a new theme: copy each `themes/nord.*` file to
`themes/<name>.*` in all four locations, change the values, keep
every variable/`@define-color` name identical. `sway-theme` picks it
up automatically — no script changes needed.

## File manager: `lf`

Migrated from `ranger` (Python, slower startup, heavier) to `lf`
(single static Go binary, async I/O, no runtime dependencies besides
a terminfo database). Set via `$file_manager` in `00-variables.conf`.

**No separate theme file needed or even possible in the useful
sense** — `lf`'s `~/.config/lf/colors` file (if you add one) uses
`dircolors`/`LS_COLORS`-style ANSI index codes (`di 01;34` for bold
blue directories, etc.), not raw hex. It inherits whatever the
*terminal's* ANSI palette currently is — the same mechanism `ls` and
`git diff` use — so switching foot's theme via `sway-theme`
automatically recolors `lf` too, with zero extra config. `lf`'s
built-in defaults are already sane ANSI-indexed values, so no colors
file is required at all unless you want to customize specific file
types beyond the defaults.

**Package availability gap, worth knowing about**: `lf` is in
Ubuntu's official `universe` repo (`apt install lf`) but is **not** in
Fedora's official repos — only via a third-party COPR
(`sudo dnf copr enable lsevcik/lf && sudo dnf install lf`). This is
the first package in this whole config that isn't from an official
repo on both distros; weigh that against the performance win if it
matters to you.

## Packages

**Fedora (`dnf`):**
```
sway foot wofi swaybg mako swayidle swaylock wl-clipboard \
cliphist waybar brightnessctl wireplumber pulseaudio-utils playerctl \
grim slurp jq polkit-gnome xdg-desktop-portal xdg-desktop-portal-wlr \
xdg-desktop-portal-gtk
# lf: not in official repos — see "File manager" above
sudo dnf copr enable lsevcik/lf && sudo dnf install lf
```

**Ubuntu/Debian (`apt`):**
```
sway foot wofi lf swaybg mako-notifier swayidle swaylock \
wl-clipboard cliphist waybar brightnessctl wireplumber \
pulseaudio-utils playerctl grim slurp jq policykit-1-gnome \
xdg-desktop-portal xdg-desktop-portal-wlr xdg-desktop-portal-gtk
```

Optional, only if you use these apps (referenced in `60-window-rules.conf`):
`pavucontrol`, `blueman`, `network-manager-gnome` (Ubuntu) /
`network-manager-applet` (Fedora).

## Known fixes already baked in (so you don't hit them again)

- **`lf` isn't in Fedora's official repos** (only Ubuntu's) — see
  "File manager" above.
- **`mako` (Fedora) vs `mako-notifier` (Ubuntu)** — same `mako` binary
  once installed, Debian just names the package differently.
- **polkit-gnome binary path** differs by distro (`/usr/libexec/...` vs
  `/usr/lib/policykit-1-gnome/...`) — `30-autostart.conf` tries both
  via a shell fallback.
- **`exec`, not `exec_always`, for long-running daemons** (mako,
  polkit agent) in `30-autostart.conf` — `exec_always` would spawn a
  duplicate on every config reload.
- **Environment variables live in `~/.config/environment.d/10-sway.conf`,
  not inside the sway config.** Sway's `include` only parses sway
  command syntax — plain `KEY=VALUE` lines there produce a config
  error on every line. `environment.d` is read by systemd at login,
  before Sway even starts; changes need a fresh login, not a reload.
- **`dbus-update-activation-environment --systemd WAYLAND_DISPLAY
  XDG_CURRENT_DESKTOP SWAYSOCK`** is the first `exec` in
  `30-autostart.conf`. Without it, anything requesting
  `org.freedesktop.portal.Desktop` (screen share, GTK file pickers,
  dark-mode detection) hangs for the full D-Bus activation timeout —
  setting the vars via `environment.d` alone isn't enough, since
  D-Bus's service-activation environment is separate from the
  session's process environment.
- **`NO_AT_BRIDGE=1`** in `environment.d/10-sway.conf` silences GTK's
  accessibility-bus warning on a session with no full desktop
  environment running an AT-SPI launcher. Remove it (and install
  `at-spi2-core`) if you ever need a screen reader.
- **Known live Ubuntu 26.04 bug**: `xdg-desktop-portal` 1.21.0+ds-1ubuntu3/4
  can fail a systemd dependency and time out under Sway
  (launchpad.net/bugs/2142255). Check
  `systemctl --user status xdg-desktop-portal.service` if the portal
  fix above doesn't fully resolve a timeout — `Failed with result
  'dependency'` means this packaging bug, not a config issue; pinning
  back to `1.20.3+ds-1ubuntu2` is the workaround until it's fixed.
- **`smart_borders on` removed** from `10-appearance.conf` — it
  conflicts with `hide_edge_borders --i3 smart` (swaywm/sway#4698);
  the latter alone covers single-window border hiding plus hides the
  titlebar on a lone tabbed/stacked child.
- **Volume/mic bindings try `wpctl` (native PipeWire) first, fall back
  to `pactl`** — matters specifically for Ubuntu 22.04 LTS, which
  still defaults to PulseAudio for audio (PipeWire there only handles
  video/screen-share). Ubuntu 22.10+ and 26.04 LTS take the `wpctl`
  branch exclusively (26.04 ships no PulseAudio at all).
- **foot's `[colors]` section will need renaming to `[colors-dark]`**
  if you ever upgrade past foot 1.26 — both target distros currently
  ship 1.25.0, where `[colors]` is correct; `[colors-dark]` wouldn't
  be recognized yet on that version.

## Keybinding cheat sheet

| Key | Action |
|---|---|
| `$mod` = Super | modifier for everything below |
| `$mod+Return` | terminal (foot) |
| `$mod+d` | launcher (wofi) |
| `$mod+e` | file manager (lf) |
| `$mod+Shift+q` | kill focused window |
| `$mod+Shift+c` | reload sway config |
| `$mod+Shift+e` | exit sway (confirm dialog) |
| `$mod+Escape` | lock screen |
| `$mod` + arrows | move focus |
| `$mod+Shift` + arrows | move window |
| `$mod` + 1–0 | switch workspace |
| `$mod+Shift` + 1–0 | move window to workspace |
| `$mod+b` / `v` | split horizontal / vertical |
| `$mod+s` / `w` | stacking / tabbed layout |
| `$mod+f` | fullscreen |
| `$mod+Shift+space` | floating toggle |
| `$mod+space` | focus tiling/floating toggle |
| `$mod+a` | focus parent container |
| `$mod+t` | border toggle |
| `$mod+r` | resize mode |
| `$mod+minus` / `Shift+minus` | scratchpad show / move |
| `Print` | screenshot mode (`s`=region, `w`=window, `Shift+`=clipboard) |
| `Ctrl+Shift` | toggle keyboard layout (US ⇄ Thai) |
| Volume/brightness/media keys | hardware keys, wpctl→pactl fallback |

## Waybar modules

`sway/workspaces`, `sway/mode`, `sway/scratchpad`, `custom/layout`
(tiling/stacking/tabbed — polled every 1s, the one polling module in
the bar since Sway's IPC events don't cleanly cover pure layout
changes) on the left; `sway/window` centered; `idle_inhibitor`,
`backlight`, `tray`, `sway/language` (US/Thai indicator), `network`,
`pulseaudio` (wpctl-first, pactl-fallback clicks/scroll), `battery`,
`clock` on the right. Plain text throughout, no icon font dependency.

## What's intentionally left out

No `swayr`, no workspace-autoname polling scripts, no `playerctl --watch`
loop, no compositing/blur fork (`swayfx`), no theme-engine indirection
layer beyond the one described above, no `nwg-shell` (considered and
rejected — heavier GTK3/Python baseline, destructive installer, no
functional gain over the already-tuned Waybar/wofi pair).
