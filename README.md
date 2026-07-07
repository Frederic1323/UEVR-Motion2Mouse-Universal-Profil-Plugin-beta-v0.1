# Motion2Mouse for UEVR — Beta 0.1

**Turn any Unreal Engine RTS / strategy game into a native-feeling VR strategy game.**

Motion2Mouse is a UEVR plugin that maps your right VR controller to a laser-pointer mouse — but it goes far beyond simple cursor emulation. It hooks directly into the Unreal Engine input system, lets you select buildings and units in the 3D world with your laser, and attaches the game UI to your left hand like a holographic wrist panel.

> RTS-Strategy-2-VR. Point. Click. Build. ⚡

---

## What's new in Beta 0.1

Beta 0.1 is a complete rewrite compared to Alpha 0.2. The old version moved the Windows cursor with hardcoded bindings. The new version is a full VR interaction layer:

| Area | Alpha 0.2 | Beta 0.1 |
|---|---|---|
| Cursor math | Angle approximation (tan yaw/pitch) | True ray-plane intersection + engine-side projection |
| World selection | Menus only | **Full 3D world interaction** (buildings, units, roads) |
| Cursor target | Windows cursor only | Windows cursor **+ Unreal engine cursor** (`SetMouseLocation`) |
| Game UI | Fixed panel in front of you | **Hand-attached panel** (wrist gesture to open/close) |
| Bindings | Hardcoded for Manor Lords | **Fully configurable** per game & per controller |
| Game support | Manor Lords tuned | **Universal** — runtime reflection, no SDK dumps needed, UE4 & UE5 |
| Haptics | None | Panel open/close pulse + optional laser-tip rumble (Lua) |
| Performance | — | Static panel, pixel-change gating, throttled updates |

---

## Core features

### 🎯 Laser-locked cursor
The cursor is computed as the exact intersection of your controller's aim ray with the target surface — the same math UEVR uses for its own laser. The cursor cannot drift away from your beam, at any angle, at any framerate.

### 🌍 True 3D world interaction
In world mode the plugin asks the game engine itself where your laser points:
`PlayerCameraManager` → aim direction composed onto the camera basis → `ProjectWorldLocationToScreen()`. Because the game's own projection is used for both directions (laser → pixel and pixel → click), selection always lands where your laser points — independent of camera height, zoom, FOV or pitch.

All engine functions are resolved at runtime via reflection (`find_function` / `find_property`), including automatic detection of UE4 (float) vs UE5 (double) layouts. **No SDK dump is ever required.** This is what makes the plugin universal.

### 🖱️ Engine cursor (bypassing Windows)
The plugin sets the mouse position **inside Unreal's input system** via `APlayerController::SetMouseLocation()` every frame — the central hook that all hover/click interactions in the game read from. The Windows cursor runs in parallel by default (needed for UMG/UI hit-testing); clicks can optionally be delivered as window messages with coordinates (`click_postmessage=1`) for a complete OS-cursor bypass.

### ✋ Hand-attached UI panel
The game UI (UEVR slate) is detached from the world and bound to your **left hand**:

- Rotate your left wrist toward your face (like checking a watch) → the panel appears.
- Rotate back → the panel disappears and your laser works in the 3D world again.
- Switching between UI mode and world mode is **fully automatic** — no button needed.
- Hysteresis + minimum dwell time prevent flickering open/close.
- The panel is placed **once** when opened and stays static → zero per-frame overhead.

Two panel styles (`ui_hand_mode`):
- `view` *(default)*: the panel hangs slightly below your gaze and is **always tilted toward your face** — ergonomic, like a drafting table.
- `hand`: an upright panel floating above your left hand.

### 🧭 Decoupled pitch & flat world mapping
The plugin auto-enables UEVR's Decoupled Pitch (`decoupled_pitch=1`) so your view stays level while the RTS camera tilts down at the map. The world mapping takes **only the camera yaw** and keeps your controller pitch 1:1 (`world_flat=1`) — this removes the height-dependent laser offset entirely.

### 📳 Haptics
- Left controller pulses when the hand panel opens/closes (`haptics=1`).
- Optional: laser-tip rumble on the right controller when the beam hits an object, and again when it jumps to a *different* object — delivered as a small patch for the community `interaction.lua` laser-pointer framework (see `scripts_patched/`).

### ⚙️ Profiles: games & controllers
One config file, three layers, later layers override earlier ones:

```
Defaults  →  [global]  →  [controller:NAME]  →  [game:EXENAME]
```

- **Game profiles** are selected automatically by the running EXE name. Launch an unknown game once and the plugin **auto-creates an empty `[game:...]` section** for you to fill in.
- **Controller presets** included: `quest`, `index`, `pimax`, `vive`, `wmr` (deadzone & trigger threshold tuned per device).

---

## Installation

1. Download `Motion2Mouse.dll` from [Releases](../../releases).
2. Place it in your UEVR game profile's plugin folder:
   `%APPDATA%\UnrealVRMod\<GameExeName>\plugins\`
3. Start the game through UEVR. The plugin initializes automatically and writes a default `motion2mouse_config.txt` (in the game's working directory, usually next to the game EXE / in `Binaries\Win64`).
4. Optional (laser-tip haptics): if you use the community Lua laser-pointer framework, replace `scripts/libs/interaction.lua` with the patched version from `scripts_patched/libs/`.

### Recommended setup

- Keep a **physical mouse and keyboard connected**.
- Windows mouse settings: disable **"Enhance pointer precision"**.
- UEVR menu: enable **"Always Show Cursor"**; slate **Overlay Type = Quad** (default).
- In-game (Manor Lords): **Borderless Fullscreen**, disable *"Move mouse outside the window"* and *"Scroll at the edge of the screen"*.
- Tip for a wide, flat hand panel with big buttons: run the game at an ultrawide resolution (e.g. 2560×720 / 3440×1440) and raise the in-game **UI scale**. The panel's aspect ratio follows the game's render resolution.

---

## Controls (default)

| Input | Action |
|---|---|
| Right controller aim | Move cursor (laser-locked) |
| Right trigger | Left mouse button |
| Right grip | Right mouse button |
| Left grip | Middle mouse button (camera drag in Manor Lords) |
| Left trigger | Shift |
| Left stick | WASD (camera pan) |
| Right stick ↑↓ | Mouse wheel (zoom) |
| Right stick ←→ | G / F (rotate) |
| A | Space (pause) |
| B | Escape |
| Left wrist rotate | Open/close hand UI panel |
| R3 (right stick click) | Manual slate/world toggle (only if `ui_hand=0`) |

> **Note on trigger/grip slots:** Depending on your UEVR version and controller profile, the physical trigger and grip may arrive on swapped XInput slots. If your buttons feel reversed, simply swap the `bind_trigger_r` / `bind_grip_r` values in the config — see below.

---

## How to change controller bindings (step by step)

**1. Find the config file.**
Launch the game once with the plugin installed. The plugin creates `motion2mouse_config.txt` in the game's working directory (usually the folder containing the game EXE, e.g. `...\ManorLords\Binaries\Win64\`). Open it with any text editor.

**2. Understand the layout.**
The file has sections. Later sections override earlier ones:

```ini
[global]                          # applies to every game
bind_grip_r=lmb

[controller:quest]                # active preset (chosen via controller=quest)
deadzone=8000
trigger_threshold=30

[game:ManorLords-Win64-Shipping]  # applies only to this EXE — wins over everything
bind_a=key:SPACE
```

**3. Know the input slots.** These are the physical inputs you can bind:

| Config key | Physical input |
|---|---|
| `bind_trigger_r` / `bind_trigger_l` | Right / left trigger (analog, threshold via `trigger_threshold`) |
| `bind_grip_r` / `bind_grip_l` | Right / left grip |
| `bind_a`, `bind_b`, `bind_x`, `bind_y` | Face buttons |
| `bind_dpad_up/down/left/right` | D-pad |
| `bind_back` | Back/menu button |
| `bind_l3` | Left stick click |
| `bind_lstick_up/down/left/right` | Left stick directions |
| `bind_rstick_left/right` | Right stick horizontal |

Right stick vertical is reserved for the mouse wheel, R3 for the mode toggle.

**4. Know the possible values.** Each slot can be bound to:

| Value | Meaning |
|---|---|
| `lmb` / `rmb` / `mmb` | Left / right / middle mouse button |
| `key:A` … `key:Z`, `key:0` … `key:9` | Letter/digit keys |
| `key:SPACE`, `key:ESC`, `key:ENTER`, `key:TAB`, `key:SHIFT`, `key:CTRL`, `key:ALT` | Named keys |
| `key:F1` … `key:F12`, `key:UP/DOWN/LEFT/RIGHT`, `key:PGUP/PGDOWN`, `key:HOME/END`, `key:INS/DEL` | Named keys |
| `key:0x2C` | Any raw Windows virtual-key code (hex) |
| `none` | Unbound |

**5. Example — put "build menu" on the X button for Manor Lords only:**

```ini
[game:ManorLords-Win64-Shipping]
bind_x=key:B
```

**6. Example — swap trigger and grip if they feel reversed:**

```ini
[global]
bind_trigger_r=lmb
bind_grip_r=rmb
```

**7. Save and restart the game.** The config is read once at startup.

> Don't know the EXE name? Start the game once — the plugin appends an empty `[game:<name>]` section automatically. Or check Task Manager → right-click the game → *Details*.

---

## Full configuration reference

| Key | Default | Description |
|---|---|---|
| `controller` | `quest` | Controller preset: `quest`, `index`, `pimax`, `vive`, `wmr`, `auto` |
| `smoothing_tau` | `0.04` | Cursor smoothing time constant in seconds (0 = off, framerate-independent) |
| `deadzone` | `8000` | Stick deadzone (0–32767) |
| `trigger_threshold` | `30` | Analog value (0–255) at which the trigger counts as pressed |
| `scroll_speed` | `1.5` | Mouse wheel speed multiplier |
| `cursor_engine` | `1` | Drive Unreal's internal cursor via `SetMouseLocation` |
| `cursor_windows` | `1` | Also move the Windows cursor (needed for UI hit-testing) |
| `click_postmessage` | `0` | Deliver clicks as window messages with coordinates (full OS-cursor bypass) |
| `world_engine` | `1` | Use the game's own projection for world mode (recommended) |
| `world_flat` | `1` | Take only camera yaw; controller pitch maps 1:1 (use with decoupled pitch) |
| `world_fov` | `100` | Horizontal FOV for the fallback frustum mapping (loading screens etc.) |
| `decoupled_pitch` | `1` | Auto-enable UEVR Decoupled Pitch at startup |
| `haptics` | `1` | Haptic pulse on panel open/close |
| `ui_hand` | `1` | Hand-attached UI panel with wrist gesture (0 = classic R3 toggle) |
| `ui_hand_mode` | `view` | `view` = gaze-anchored, auto-tilted panel; `hand` = upright above left hand |
| `ui_hand_size` | `0.45` | Panel size in meters |
| `ui_hand_on` / `ui_hand_off` | `0.60` / `0.30` | Gesture hysteresis thresholds (open above / close below) |
| `ui_hand_dwell` | `0.30` | Minimum seconds between panel state changes |
| `ui_view_dist` / `ui_view_y` | `0.50` / `-0.22` | Panel distance and vertical offset (view mode) |
| `ui_hand_lift` | `0.12` | Panel height above the hand (hand mode) |

---

## Compatibility

| Game | Status | Notes |
|---|---|---|
| **Manor Lords** | ✅ Great | Full profile included; world selection, building, roads |
| **Frostpunk 2** | ✅ Working | UE5; create/adjust bindings via game profile & in-game key settings |
| **HumanitZ** | 🟡 Legacy | Worked in Alpha; re-testing needed for Beta |
| **SurrounDead** | 🟡 Legacy | Worked in Alpha; re-testing needed for Beta |
| **SpaceBourne 2** | 🟡 Menus | UI interaction only |
| Any UE4/UE5 mouse-driven game | 🧪 Should work | Runtime reflection — try it and report! |

**Tested setup:** Meta Quest 3 via Virtual Desktop (VDXR/OpenXR), UEVR Nightly 1096+. Other headsets should work (controller presets included) — community reports welcome.

---

## Known limitations (Beta)

- The panel's aspect ratio is fixed to the game's render resolution (UEVR limitation) — use an ultrawide game resolution for a flat, wide panel.
- The slate panel cannot be freely rotated via the UEVR API; `view` mode approximates a tilted panel by always facing your gaze.
- Per-widget features (pop-ups floating above the selected building, radial menus) are outside what a UEVR plugin can do — they would require game-side modding.
- Trigger/grip XInput slot mapping differs between UEVR builds; use the config to swap if needed.

## Roadmap

- 🕹️ Free camera flight / god-view (world scale & camera offsets on grip + stick)
- ✋ Skeletal mesh hands (via the community `hands.lua` framework)
- 🎡 Radial quick-menu experiment (ImGui overlay)
- 🖱️ Drag & box-select helper for unit selection
- 📳 Haptic ticks when hovering panel buttons

## Building from source

- Visual Studio 2022, C++17 or newer, x64 Release.
- Depends on the [UEVR Plugin API](https://github.com/praydog/UEVR) headers (`uevr/Plugin.hpp`).
- The project uses a precompiled header (`pch.h`); optionally add `NOMINMAX` there before `<windows.h>`.
- Single translation unit: `main.cpp` → builds to `Motion2Mouse.dll` exporting `create_plugin()`.

## Credits

- [praydog](https://github.com/praydog) — UEVR, the foundation that makes all of this possible.
- The UEVR community Lua framework (`uevr_utils` / `interaction`) used for the visual laser pointer and its haptics patch.
- Everyone testing and reporting — this is a Beta, feedback drives it. Open an [Issue](../../issues)!

## License

MIT — see `LICENSE`.
