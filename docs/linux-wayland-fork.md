# Linux / Wayland fixes carried by this fork

This fork (`anisf/openscreen`, branch `fix/linux-wayland-hud-scaling`) is upstream
`getopenscreen/openscreen` plus a few fixes for Linux on Wayland, found on Omarchy
(Hyprland 0.56, NVIDIA, a 5120x2160 monitor at scale 1). This page says what each fix
does, why it is needed, and how to rebuild and install the app from this branch.

## The problems and the fixes

### 1. The HUD is drawn smaller than its window (gap on the right and bottom)

**Cause.** Omarchy's `omarchy display text size` sets GNOME's `text-scaling-factor`
(0.9091 for text size 11). Chromium/Electron multiplies that factor into its device
scale factor, so the renderer draws at `devicePixelRatio = 0.906` while Hyprland gives
the Wayland surface its full logical size. The content fills only ~90% of the window,
anchored top-left, leaving a transparent strip on the right and bottom.

This is not OpenScreen-specific: it is the Omarchy issue
[omacom/omarchy#8716](https://github.com/omacom/omarchy/issues/8716) (any Electron app,
text size below default), with the same root cause as
[#8574](https://github.com/omacom/omarchy/issues/8574) and
[#12228](https://github.com/omacom/omarchy/issues/12228), where Omarchy pins
`--force-device-scale-factor=1` for 1Password.

**Fix.** `electron/wayland.ts::configureWaylandSupport` now appends
`--force-device-scale-factor=1` in a Wayland session, unless the switch was already
given on the command line (an explicit value is kept). Tests: `electron/wayland.test.ts`.

Equivalent workaround for an unpatched build: start the stock binary with
`--force-device-scale-factor=1`.

### 2. The HUD window is much bigger than the HUD (large transparent frame)

**Cause.** On Linux the HUD window is not click-through, so every transparent pixel
around the bar still eats clicks. Upstream sized the Linux window with reserves for
panels, tooltips and edge slack (stock window ~615x576 for a 567x55 bar), plus native
minimums of 120x80 and a 220px minimum width.

**Fix (Linux only, Windows/macOS unchanged):**

- `src/components/launch/hudGeometry.ts`: `computeHudLinuxStack` keeps no idle
  tooltip/panel reserve; the window grows only by what is actually open (menu, settings
  panel, toast) and shrinks back. `computeHudWindowSize` drops the 220px width floor
  when there is no growth reserve, and the Linux bar has no bottom margin.
- `src/components/launch/LaunchWindow.tsx`: measures the visible stack and toasts and
  requests exactly that size.
- `electron/windows.ts`: Linux HUD window starts at 600x190 and has a 1x1 native
  minimum instead of 120x80.
- `electron/hudWindowBounds.ts`: the work-area clamp uses the same 1x1 minimum on Linux.
- Tests: `src/components/launch/hudGeometry.test.ts`,
  `src/components/launch/LaunchWindow.test.tsx`.

Result measured on Hyprland: window 567x54 for a 566x54 bar.

Trade-offs: tooltips have no reserved room above the idle bar, so they can be clipped;
on Wayland, opening a panel resizes the toplevel and the compositor decides where the
grown window sits, so the bar can shift while a panel is open.

### 3. Rounded corners

- `src/components/launch/LaunchWindow.module.css`: bar radius 20px, `overflow: hidden`,
  and the outer drop shadow replaced by an inset highlight (an outer shadow is clipped
  by a window that hugs the bar, which looked like a thin gutter).
- `electron/windows.ts`: `roundedCorners: true` on Linux (still `false` elsewhere).

On Hyprland the compositor also draws its own border and corners. Add this rule to
`~/.config/hypr/looknfeel.lua` (Omarchy Lua config) so they match the bar:

```lua
-- OpenScreen's frameless HUD sizes itself to its rounded bar.
o.window({ class = "^openscreen$", title = "^openscreen$" }, {
  float = true,
  border_size = 0,
  rounding = 20,
})
```

Then `hyprctl reload`. The title match also catches any other window titled
`openscreen`.

### 4. Full-screen recording at 5120x2160 failed to open an H.264 encoder

**Cause.** The native PipeWire helper tried VAAPI, Vulkan and software at the full
5120x2160 size. H.264 tops out at 4096 per axis in most hardware encoders, and on an
NVIDIA host VAAPI/Vulkan were not usable, so every attempt failed with
`no H.264 encoder could be opened at 5120x2160`.

**Fix** (`electron/native/pipewire-capture/src/{encoder,capture,dmabuf_import,main}.rs`):
try NVENC first (then VAAPI, Vulkan, software), cap the H.264 output to 4096 on the
long axis keeping the aspect (5120x2160 becomes 4096x1728), track source, crop and
output dimensions separately, and crop DMA-BUF frames before scaling.

Status: builds; a full 5120x2160 recording has not been re-tested end to end yet.

## Rebuilding and installing from this branch (Arch / Omarchy, pacman)

```bash
git clone -b fix/linux-wayland-hud-scaling https://github.com/anisf/openscreen.git
cd openscreen
npm ci

# Native helpers (Rust + FFmpeg SDK). Needed for the encoder fix; the prebuilt
# electron/native/bin/linux-x64 payload is gitignored, so build it on a fresh clone.
npm run build:native:linux
npm run build:native:compositor:linux

# App + pacman package only. OPENSCREEN_SYMBOL_FLOOR=host accepts the host's newer
# glibc instead of the pinned Ubuntu floor; such a package is for this machine only.
OPENSCREEN_SYMBOL_FLOOR=host sh -c \
  'npx tsc && npx vite build && npx electron-builder --linux pacman --config.npmRebuild=false'

sudo pacman -U release/*/Openscreen-Linux-*.pacman
```

Notes:

- Use npm (see `AGENTS.md`). Node 22 is the pinned engine; on Node 26 the jsdom tests
  in `LaunchWindow.test.tsx` fail during setup, Node 22/24 run them.
- `npm run build` builds every Linux target; its AppImage update hook can fail on a
  local build. The pacman-only command above avoids it.
- If the STT native build fails on a missing `Vulkan_INCLUDE_DIR`, install
  `vulkan-headers` or copy the whisper/STT libraries from an installed OpenScreen
  (`/opt/Openscreen/resources/electron/native/bin/linux-x64`).
- `patchelf` must be on `PATH` for packaging.

Focused checks for these changes:

```bash
npx vitest --run electron/wayland.test.ts electron/windows.test.ts electron/hudWindowBounds.test.ts
npx vitest --run src/components/launch/hudGeometry.test.ts src/components/launch/LaunchWindow.test.tsx
npx tsc --noEmit && npx tsc -p tsconfig.test.json --noEmit
```

## Launching on Omarchy / Hyprland + NVIDIA

The packaged desktop entry runs `/opt/Openscreen/openscreen %U`. On this host the
binary also needs Wayland/GL flags to start; a small wrapper in `~/.local/bin/openscreen`
used by a personal `~/.local/share/applications/openscreen.desktop` does it:

```bash
#!/usr/bin/env bash
exec /opt/Openscreen/openscreen \
  --no-sandbox \
  --ozone-platform=wayland \
  --use-gl=angle --use-angle=vulkan --use-vulkan-swiftshader \
  --enable-features=UseOzonePlatform,WaylandWindowDrag \
  --force-device-scale-factor=1 \
  "$@"
```

`--force-device-scale-factor=1` there is redundant with fix 1 but keeps an unpatched
upstream build correct after an upgrade.

## Keeping the fork current

```bash
git remote add upstream https://github.com/getopenscreen/openscreen.git
git fetch upstream
git rebase upstream/main     # resolve conflicts in the files listed above
```
