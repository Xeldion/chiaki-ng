# Mouse-to-Right-Stick Aim & Mouse Button Remapping

This fork adds two features to chiaki-ng, both controlled from **Settings → Keys**:

1. **Mouse → right analog stick** ("Enable Mouse To Right Stick") — mouse movement
   drives the camera/aim stick, with a sensitivity slider and auto-recenter.
2. **Mouse button remapping** ("Mouse Button Mapping") — bind Left / Right / Middle /
   Back / Forward mouse buttons to any controller action. Defaults to **unbound**.

## How mouse-aim works (the part that's easy to get wrong)

The right stick is a *position* (±0x7fff), not a velocity, and Qt only hands us
absolute cursor positions — never raw relative deltas. So the aim path is:

1. On each mouse-move, measure the cursor's offset from the window **center**.
2. Feed `dx * sensitivity` / `dy * sensitivity` (clamped to ±0x7fff) into the right
   stick and send the controller state.
3. **Warp the cursor back to the window center** (`QCursor::setPos`), so the next
   event is another small relative delta.
4. A 50 ms single-shot timer zeroes the stick after movement stops — without it the
   camera spins forever.

The warp step is the crux. `QCursor::setPos` is a **no-op on native Wayland**, so on
a Wayland session this fork forces the `xcb` (XWayland) platform at startup —
**only when** "Enable Mouse To Right Stick" is on, and only when you haven't set
`QT_QPA_PLATFORM` yourself. Normal native-Wayland use is unaffected.

## Settings reference

| Setting | QSettings key | Default |
|---|---|---|
| Enable mouse-aim | `settings/mouse_to_stick_enabled` | `false` |
| Sensitivity | `settings/mouse_stick_sensitivity` | `500` |
| Mouse button → action | `mousemap/<left\|right\|middle\|back\|forward>` | (unset = unbound) |

Sensitivity formula: `right_x = clamp(round(dx * sensitivity), -0x7fff, 0x7fff)`.
Per-event `dx` is small (the cursor is warped back each frame), so useful values are
in the hundreds–thousands; the slider spans 1–2000. **Expect to tune this to your
mouse/DPI** — start around 500 and adjust up for slower mice, down for twitchy ones.

While mouse-aim is enabled, mouse movement and clicks stop driving the touchpad
("Enable Mouse Touchpad" is effectively superseded). Keyboard-as-controller is
unaffected.

## Build (native, Qt 6.10)

```bash
git submodule update --init --recursive
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build --target chiaki -j"$(nproc)"
# binary at build/gui/chiaki
```

Dependencies on Debian (all present on the dev machine):
`qt6-base-dev qt6-declarative-dev qt6-tools-dev libsdl2-dev libavcodec-dev
libavformat-dev libswscale-dev libavutil-dev libswresample-dev libssl-dev
protobuf-compiler libprotobuf-dev libplacebo-dev libvulkan-dev libshaderc-dev
pkg-config cmake ninja-build`.

## Test it

```bash
# run the native binary (not the Flatpak!) — this is what carries the patch
./build/gui/chiaki
```

1. Settings → **Keys** tab.
2. Tick **"Enable Mouse To Right Stick"**, pick a sensitivity.
3. In **Mouse Button Mapping**, set e.g. Left = R2, Right = L2.
4. Stream a game, make the window fullscreen, and move the mouse.

Because the aim mode forces XWayland, you can confirm it took effect with:
`QT_DEBUG_PLUGINS=1 ./build/gui/chiaki 2>&1 | grep 'libqxcb'`

> The Flatpak install (`io.github.streetpea.Chiaki4deck`) is a *separate* binary and
> does **not** contain this patch — run `build/gui/chiaki` to test the feature.

## Files changed

- `gui/include/settings.h` / `gui/src/settings.cpp` — settings + mouse-button map storage.
- `gui/include/streamsession.h` / `gui/src/streamsession.cpp` — aim handling, button
  mapping, reset timer; extracted the keyboard switch into `SetControllerButton`.
- `gui/include/qmlsettings.h` / `gui/src/qmlsettings.cpp` — QML exposure.
- `gui/src/qml/SettingsDialog.qml` — the Keys-tab controls.
- `gui/src/qmlmainwindow.cpp` — warp-to-center relative mouse routing.
- `gui/src/main.cpp` — Wayland → xcb platform forcing when mouse-aim is enabled.
