# Motion2Mouse for UEVR — Beta 0.9

**Turn any Unreal Engine RTS / strategy game into a native-feeling VR strategy game.**

Motion2Mouse is a UEVR plugin that maps your right VR controller to a laser-pointer mouse — but it goes far beyond simple cursor emulation. It hooks directly into the Unreal Engine input system, lets you select buildings and units in the 3D world with your laser, opens the game UI as a holographic panel right where you look, watches the game's own UI so windows open and close your panel automatically — and lets you rearrange the game HUD like Tetris pieces, live, while playing.

> RTS-Strategy-2-VR. Point. Click. Build. ⚡

---

## What's new in Beta 0.9

Beta 0.9 turns the interaction layer of Beta 0.1 into a full VR cockpit:

| Area | Beta 0.1 | Beta 0.9 |
|---|---|---|
| Panel placement | View / hand mode | **Gaze mode** (default): panel opens where you look, then stays world-fixed |
| Panel toggle | Wrist gesture | **X button** (default); wrist gesture still available |
| Game awareness | None | **Popup watch**: the game opens a window → panel opens automatically, game pauses, panel closes with the window |
| Solo window | — | **`popup_solo=1`**: while a game window is open, the rest of the HUD is hidden — the panel shows only that window |
| HUD | As the game renders it | **HUD Tetris**: move & scale every HUD widget freely, live via hot-reload |
| Click-to-build | — | **Click zones**: clicking a road/building in the panel closes it — laser instantly back in the world for placement |
| Target feedback | — | **Hover snap + haptics**: cursor sticks to selectable targets, controller pulses on lock-on |
| Turning | Game orbit keys | **Self-turn**: right stick rotates *you* (snap or smooth), panel stays in place |
| Scrolling | Raw wheel deltas | Clean `WHEEL_DELTA` steps — fixes zoom in Manor Lords & other UE games |
| Config | Read once at startup | **Live hot-reload** (~1 s), per-game & per-controller profiles, auto-migration, widget discovery tool |
| Robustness | — | Crash guard around the whole tick, per-container popup detection, repeated HUD restore |
| Content | — | **Pak auto-installer** for optional game-content files |

---

## Core features

### 🎯 Laser-locked cursor
The cursor is computed as the exact intersection of your controller's aim ray with the target surface — the same math UEVR uses for its own laser. The cursor cannot drift away from your beam, at any angle, at any framerate.

### 🌍 True 3D world interaction
In world mode the plugin asks the game engine itself where your laser points:
`PlayerCameraManager` → aim direction composed onto the camera basis → `ProjectWorldLocationToScreen()`. Because the game's own projection is used for both directions (laser → pixel and pixel → click), selection always lands where your laser points — independent of camera height, zoom, FOV or pitch.

All engine functions are resolved at runtime via reflection (`find_function` / `find_property`), including automatic detection of UE4 (float) vs UE5 (double) layouts. **No SDK dump is ever required.** This is what makes the plugin universal.

### 🖱️ Engine cursor (bypassing Windows)
The plugin sets the mouse position **inside Unreal's input system** via `APlayerController::SetMouseLocation()` — the central hook that all hover/click interactions in the game read from. The Windows cursor runs in parallel by default (needed for UMG/UI hit-testing); clicks can optionally be delivered as window messages with coordinates (`click_postmessage=1`) for a complete OS-cursor bypass.

### 🧲 Hover snap & lock-on haptics
The plugin doesn't guess whether your laser is over something selectable — **the game tells it** (via its own tooltip widget, watched through reflection):

- The moment the game recognizes a selectable target under your cursor, your laser controller gives a short **lock-on pulse** (`hover_haptics=1`, strength via `hover_strength`).
- While locked on, the cursor becomes **sticky** (`cursor_snap=1`): its smoothing is multiplied by `cursor_snap_factor`, so it softly snaps onto small targets instead of sliding past them. World mode only — inside the panel the cursor stays fast.
- `hover_widget` is configurable per game (default tuned for Manor Lords).

Bonus: the pulse doubles as a diagnostic. No pulse while pointing at something = the game doesn't see a selectable target there.

### 👁️ Gaze panel (the main UI)
The game UI (UEVR slate) becomes a floating panel:

- Press **X** → the panel appears **where you are looking** and stays world-fixed there.
- Press **X** again → the panel parks out of sight and your laser works in the 3D world.
- Panel modes (`ui_hand_mode`): `gaze` *(default)*, `view` (permanently follows your gaze), `hand` (upright above the left hand; the classic wrist gesture is available via `ui_hand_toggle=gesture`).
- Hysteresis + minimum dwell time prevent flickering open/close.
- The panel is placed **once** when opened and stays static → zero per-frame overhead.

### 🪟 Popup watch — the game drives the panel
The plugin observes the game's own UI containers via runtime reflection (`ui_popup_watch=1`):

- Click a building in the world → the game opens its info window → **your panel opens automatically**, showing it.
- `popup_pause=1` pauses the game while a window is open, unpauses on close.
- The panel closes only when **all** watched containers are empty — transient notifications (e.g. unit production messages) can no longer hijack the close detection.
- The X button also closes the game's window (`popup_x_closes=1`), and `popup_offset_x/y` centers the game's window inside the panel.

### 🧩 Solo window — one window, nothing else
**Set `popup_solo=1` in your config to enable this** (it ships disabled):

```ini
popup_solo=1          # while a game window is open, hide the rest of the HUD
popup_solo_enforce=1  # re-hide stubborn widgets every 0.3 s (the game re-shows some)
popup_solo_move=1     # additionally shove stubborn widgets off-screen (position sticks)
```

While a building/event window is open, the panel shows **only that window** — top bars, portrait, build bar and date/season are hidden and restored exactly when the window closes (restore is repeated for ~1 s to also catch widgets the game re-creates). `popup_hide=` lists the affected widgets.

### 🧱 HUD Tetris — rearrange the game HUD
Every widget of the game's main HUD can be **moved and scaled** — live, while the game runs. All of it lives in the **`[game:...]` section** of the respective game, right next to its bindings:

```ini
hud_move=main_tabs_HB,0,-430       # property name, x-offset, y-offset (pixels)
hud_move=TopCluster_VB,0,380
hud_move=DateTimeHB,-500,-600      # date + season widget
hud_scale=main_tabs_HB,0.85,0.85   # property name, x-scale, y-scale
hud_move_enforce=1                 # re-apply every 0.3 s (beats game animations)
```

**The recommended workflow:**
1. **Size first, in-game:** set the game's own **UI scale** (and your render resolution) until the buttons have the size you want on the panel.
2. **Then Tetris, via hot-reload:** keep `motion2mouse_config.txt` open in an editor next to the game, change `hud_move` / `hud_scale` values, hit save — the panel updates after ~1 second. Nudge in ±50 steps until the layout clicks together.

**Don't know a widget's name?** Set `hud_list=1` + save → the plugin dumps **every widget of the main UI** (name, class, visibility) into the UEVR log once, then set it back to 0. This is how the Manor Lords preset found `main_tabs_HB` (build bar) and `DateTimeHB` (date + season).

### 🚪 Click zones — click a building, panel closes
With `ui_click_close=1`, a left click inside a defined zone of the panel closes it automatically — click a road or a building card, and your laser is instantly back in the world for placement:

```ini
ui_click_log=1                       # measuring mode: logs every panel click
# play, click the road button / building cards, read the UEVR log:
#   [Motion2Mouse] Panel-Klick bei x=0.412 y=0.573
ui_click_zone=0.38,0.54,0.45,0.61    # x1,y1,x2,y2 (normalized 0..1), repeatable
ui_click_zone=0.15,0.15,0.85,0.48
ui_click_log=0                       # measuring mode off
```

Define zones only over "build" clicks (road button, building selection cards) — not over the tab buttons themselves. Everything is hot-reloadable, and `ui_click_close_delay` makes sure the click registers before the panel closes.

### 🔄 Self-turn
`rotate_self=1` makes the right stick rotate **you** (the VR view around your head) instead of sending the game's orbit keys:

- `rotate_self_mode=snap` *(default)*: comfortable snap turning, `rotate_self_snap=30` degrees per flick, with a haptic tick.
- `rotate_self_mode=smooth`: continuous turning at `rotate_self_speed` deg/s.
- The open panel is **re-anchored automatically** after every turn — it stays exactly where it was.
- Feels wrong? Try `rotate_self_order=1` (UEVR builds differ), `rotate_self_invert=1`, or toggle `rotate_self_pivot`.
- Emergency exit: `rotate_reset=1` + save restores the view captured at game start; then set it back to 0.

### 🧭 Decoupled pitch & flat world mapping
The plugin auto-enables UEVR's Decoupled Pitch (`decoupled_pitch=1`) so your view stays level while the RTS camera tilts down at the map. The world mapping takes **only the camera yaw** and keeps your controller pitch 1:1 (`world_flat=1`) — this removes the height-dependent laser offset entirely.

### 📦 Pak auto-installer
Some optional features ship as game-content files (`.pak` / `.utoc` / `.ucas`). You never install them manually:

1. Drop the pak files into your UEVR profile folder — next to the plugin DLL (`plugins\`) or one level above, in the profile root.
2. Start the game once. The plugin copies them to `<Game>\Content\Paks\~mods` automatically and logs `Pak installiert`.
3. Restart the game — the content is active from the next launch on.

Mod users only import the UEVR profile; the plugin handles the rest. Files are re-copied only when they change.

### 📳 Haptics
- Lock-on pulse on the laser controller when the cursor snaps onto a selectable target.
- Pulse on the off-hand controller when the panel opens/closes.
- Haptic tick on every snap turn.
All gated by `haptics=1`, hover feedback separately by `hover_haptics` / `hover_strength`.

### ⚙️ Profiles: games & controllers
One config file, three layers, later layers override earlier ones:

```
Defaults  →  [global]  →  [controller:NAME]  →  [game:EXENAME]
```

- **Game profiles** are selected automatically by the running EXE name. Launch an unknown game once and the plugin **auto-creates an empty `[game:...]` section** for you to fill in.
- Profiles for **Manor Lords**, **Frostpunk 2**, **Homeworld 3** and **Tempest Rising** are included.
- **Controller presets**: `quest`, `index`, `pimax`, `vive`, `wmr` (deadzone & trigger threshold tuned per device).
- **Hot-reload**: edit + save the config while playing — changes apply after ~1 second. No restart. On version bumps your previous file is kept as `motion2mouse_config_old.txt`.
- A **crash guard** wraps the whole plugin tick: a bad frame (e.g. reflection during a level change) is skipped and logged once instead of erroring out in UEVR.

---

## Installation

1. Download `Motion2Mouse.dll` from [Releases](../../releases).
2. Place it in your UEVR game profile's plugin folder:
   `%APPDATA%\UnrealVRMod\<GameExeName>\plugins\`
3. Start the game through UEVR. The plugin initializes automatically and writes a default `motion2mouse_config.txt` **next to the plugin DLL** — each game profile carries its own config. (Configs from older versions are migrated automatically.)
4. Optional pak content: drop the `.pak`/`.utoc`/`.ucas` files into the profile folder — the auto-installer does the rest (see above).

### Recommended setup

- Keep a **physical mouse and keyboard connected**.
- Windows mouse settings: disable **"Enhance pointer precision"**.
- UEVR menu: enable **"Always Show Cursor"**; slate **Overlay Type = Quad** (default).
- Virtual Desktop users: set the OpenXR runtime to **VDXR** (avoids crashes in the Oculus emulation layer).
- In-game (Manor Lords): **Borderless Fullscreen**, disable *"Move mouse outside the window"* and *"Scroll at the edge of the screen"*.
- Tip for a wide, flat panel with big buttons: run the game at an ultrawide resolution (e.g. 2560×720 / 3440×1440) and raise the in-game **UI scale**. The panel's aspect ratio follows the game's render resolution.

---

## Controls (default)

| Input | Action |
|---|---|
| Right controller aim | Move cursor (laser-locked, snaps onto targets) |
| Right trigger | Left mouse button |
| Right grip | Right mouse button |
| Left grip | Middle mouse button (camera drag in Manor Lords) |
| Left trigger | Shift |
| Left stick | WASD (camera pan) |
| Right stick ↑↓ | Mouse wheel (zoom) |
| Right stick ←→ | Rotate: game keys — or **self-turn** if `rotate_self=1` |
| **X** | **Open/close the main UI panel** |
| A | Space (pause) |
| B | Escape |
| R3 (right stick click) | Manual slate/world toggle (only if `ui_hand=0`) |

> **Note on trigger/grip slots:** Depending on your UEVR version and controller profile, the physical trigger and grip may arrive on swapped XInput slots. If your buttons feel reversed, simply swap the `bind_trigger_r` / `bind_grip_r` values in the config — see below.

---

## Setting up the main UI panel (step by step)

Everything below goes into the **`[game:<YourGameExe>]` section** of `motion2mouse_config.txt` — panel layout, HUD Tetris, click zones and bindings are per-game settings and live together.

**1. Open the config.**
It sits next to the plugin DLL: `%APPDATA%\UnrealVRMod\<GameExeName>\plugins\motion2mouse_config.txt`. Keep it open in an editor while the game runs — every save hot-reloads after ~1 second.

**2. Choose how the panel opens.**

```ini
ui_hand=1              # panel system on
ui_hand_mode=gaze      # gaze (recommended) | view | hand
ui_hand_toggle=x       # x/y/a/b/back/l3 or gesture (wrist rotation)
ui_hand_size=0.45      # panel size in meters
ui_view_dist=0.50      # distance in front of your face (gaze/view)
```

**3. Let the game drive it — and enable the solo window.**

```ini
ui_popup_watch=1       # game window opens -> panel opens
ui_popup_close=1       # all game windows closed -> panel closes
popup_pause=1          # pause while a window is open
popup_solo=1           # <- IMPORTANT: set this to 1 for the clean solo window
popup_solo_enforce=1
popup_solo_move=1
```

**4. Size first, then Tetris.**
Set widget **sizes in-game first** (the game's UI scale + render resolution). Then arrange the HUD **live via hot-reload** with `hud_move` / `hud_scale`. Unknown widget names → `hud_list=1`, read the UEVR log, set back to 0.

**5. Measure your click zones.**
`ui_click_log=1` → click the road button and the building cards → read the coordinates from the UEVR log → define `ui_click_zone=` rectangles around them → log off.

**6. Done.** Every step is reversible live. `ui_restore=1` (set briefly, then back to 0) returns the panel to UEVR's original placement; `rotate_reset=1` does the same for the view rotation.

---

## How to change controller bindings (step by step)

**1. Find the config file** (see above — next to the plugin DLL).

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
| `bind_a`, `bind_b`, `bind_x`, `bind_y` | Face buttons (the panel-toggle button is auto-excluded) |
| `bind_dpad_up/down/left/right` | D-pad |
| `bind_back` | Back/menu button |
| `bind_l3` | Left stick click |
| `bind_lstick_up/down/left/right` | Left stick directions |
| `bind_rstick_left/right` | Right stick horizontal (ignored while `rotate_self=1`) |

Right stick vertical is reserved for the mouse wheel, R3 for the mode toggle (classic mode).

**4. Know the possible values.** Each slot can be bound to:

| Value | Meaning |
|---|---|
| `lmb` / `rmb` / `mmb` | Left / right / middle mouse button |
| `key:A` … `key:Z`, `key:0` … `key:9` | Letter/digit keys |
| `key:SPACE`, `key:ESC`, `key:ENTER`, `key:TAB`, `key:SHIFT`, `key:CTRL`, `key:ALT` | Named keys |
| `key:F1` … `key:F12`, `key:UP/DOWN/LEFT/RIGHT`, `key:PGUP/PGDOWN`, `key:HOME/END`, `key:INS/DEL` | Named keys |
| `key:0x2C` | Any raw Windows virtual-key code (hex) |
| `none` | Unbound |

**5. Example — put "build menu" on the Y button for Manor Lords only:**

```ini
[game:ManorLords-Win64-Shipping]
bind_y=key:B
```

**6. Example — swap trigger and grip if they feel reversed:**

```ini
[global]
bind_trigger_r=lmb
bind_grip_r=rmb
```

**7. Save.** Hot-reload applies it after ~1 second — no restart needed.

> Don't know the EXE name? Start the game once — the plugin appends an empty `[game:<name>]` section automatically. Or check Task Manager → right-click the game → *Details*.

---

## Full configuration reference

| Key | Default | Description |
|---|---|---|
| `controller` | `quest` | Controller preset: `quest`, `index`, `pimax`, `vive`, `wmr`, `auto` |
| `smoothing_tau` | `0.04` | Cursor smoothing time constant in seconds (0 = off, framerate-independent) |
| `deadzone` | `8000` | Stick deadzone (0–32767) |
| `trigger_threshold` | `30` | Analog value (0–255) at which the trigger counts as pressed |
| `scroll_speed` / `scroll_invert` | `1.5` / `1` | Mouse wheel speed & direction |
| `scroll_mode` | `sendinput` | `postmessage` sends wheel events directly to the game window |
| `cursor_engine` | `1` | Drive Unreal's internal cursor via `SetMouseLocation` |
| `cursor_windows` | `1` | Move the Windows cursor: `1` always, `panel` only while the panel is open, `0` never |
| `click_postmessage` | `0` | Deliver clicks as window messages with coordinates (full OS-cursor bypass) |
| `world_engine` | `1` | Use the game's own projection for world mode (recommended) |
| `world_flat` | `1` | Take only camera yaw; controller pitch maps 1:1 (use with decoupled pitch) |
| `world_fov` | `100` | Horizontal FOV for the fallback frustum mapping (loading screens etc.) |
| `decoupled_pitch` | `1` | Auto-enable UEVR Decoupled Pitch at startup |
| `haptics` | `1` | Master switch for all haptic pulses |
| `hover_haptics` / `hover_strength` | `1` / `0.45` | Lock-on pulse when the cursor hits a selectable target |
| `hover_widget` | ML preset | The game's tooltip widget used as hover detector (find via `hud_list`) |
| `cursor_snap` / `cursor_snap_factor` | `1` / `3.0` | Sticky cursor over targets (smoothing multiplier) |
| `left_handed` | `0` | Swap laser hand and gesture hand |
| `ui_hand` | `1` | Panel system (0 = classic R3 slate/world toggle) |
| `ui_hand_mode` | `gaze` | `gaze` = opens where you look, then world-fixed; `view` = follows gaze; `hand` = above left hand |
| `ui_hand_toggle` | `x` | Panel toggle: `x/y/a/b/back/l3` or `gesture` |
| `ui_hand_size` | `0.45` | Panel size in meters |
| `ui_view_dist` / `ui_view_y` | `0.50` / `-0.35` | Panel distance / vertical offset |
| `ui_hand_on` / `ui_hand_off` / `ui_hand_dwell` | `0.68` / `0.40` / `0.30` | Gesture hysteresis & minimum dwell (gesture mode) |
| `ui_popup_watch` / `ui_popup_close` | `1` / `1` | Game windows open/close the panel automatically |
| `popup_mainui_class` | ML preset | Widget class of the game's main UI (runtime reflection target) |
| `popup_containers` | ML preset | UI containers watched for opening windows (comma-separated) |
| `popup_pause` / `popup_pause_key` | `1` / `SPACE` | Auto-pause while a game window is open |
| `popup_x_closes` / `popup_close_key` | `1` / `ESC` | Panel-toggle button also closes the game's window |
| `popup_offset_x/y` | `0` | Move the game's window inside the panel (pixels) |
| `popup_vec2_double` | `1` | `1` = UE5 (double vectors), `0` = UE4 |
| `popup_solo` | `0` | **Set to 1**: hide the rest of the HUD while a game window is open |
| `popup_solo_enforce` | `1` | Re-hide stubborn widgets every 0.3 s |
| `popup_solo_move` | `1` | Additionally shove stubborn widgets off-screen (restored on close) |
| `popup_hide` | ML preset | Widget list affected by solo mode |
| `hud_move` | — | `<Property>,<dx>,<dy>` move a HUD widget (repeatable, hot-reload) |
| `hud_scale` | — | `<Property>,<sx>,<sy>` scale a HUD widget (repeatable, hot-reload) |
| `hud_move_enforce` | `1` | Re-apply moves/scales every 0.3 s (beats game animations) |
| `hud_list` | `0` | Set to 1 + save: dump all main-UI widget names into the UEVR log once |
| `ui_hide_hud` / `hud_container` | `0` / ML preset | Collapse the persistent HUD while the panel is closed (experimental) |
| `ui_click_close` | `1` | Clicks inside click zones close the panel (build workflow) |
| `ui_click_zone` | — | `x1,y1,x2,y2` normalized close-zone (repeatable) |
| `ui_click_log` | `0` | Log every panel click's coordinates (for measuring zones) |
| `ui_click_close_delay` | `0.25` | Delay so the click still registers before the panel closes |
| `rotate_self` | `0` | Right stick X turns **you** instead of sending game keys |
| `rotate_self_mode` | `snap` | `snap` or `smooth` |
| `rotate_self_snap` / `rotate_self_speed` | `30` / `120` | Degrees per flick / degrees per second |
| `rotate_self_invert` / `rotate_self_order` / `rotate_self_pivot` | `0` / `0` / `1` | Direction / quaternion order / pivot around your head |
| `rotate_self_deadzone` / `rotate_self_in_panel` | `12000` / `0` | Turn deadzone / allow turning while the panel is open |
| `rotate_reset` | `0` | Set to 1 + save = restore the view captured at game start |
| `ui_restore` | `0` | Set to 1 + save = restore UEVR's original slate placement |
| `ui_auto_show` | `0` | Open the panel automatically after a left click in the world |

---

## Compatibility

| Game | Status | Notes |
|---|---|---|
| **Manor Lords** | ✅ Great | Full profile: world selection, popup watch, solo window, HUD Tetris preset, hover snap, click zones |
| **Frostpunk 2** | ✅ Working | UE5; mouse+keyboard bindings included; popup watch off until containers are mapped |
| **Homeworld 3** | 🧪 Profile included | Full 3D camera preset (`world_flat=0`) |
| **Tempest Rising** | 🧪 Profile included | UE5; control groups on the D-pad |
| Any UE4/UE5 mouse-driven game | 🧪 Should work | Runtime reflection — try it and report! |

**Tested setup:** Meta Quest 3 via Virtual Desktop (**VDXR**/OpenXR), UEVR Nightly 1096+. Other headsets should work (controller presets included) — community reports welcome.

---

## Known limitations (Beta)

- The panel's aspect ratio is fixed to the game's render resolution (UEVR limitation) — use an ultrawide game resolution for a flat, wide panel.
- The slate panel cannot be freely rotated via the UEVR API; in `gaze` mode it always faces the recenter-forward direction, so it looks slightly angled when opened while looking sideways.
- Popup watch, solo window, HUD Tetris and hover snap need the game's widget names (`popup_mainui_class`, `popup_containers`, `popup_hide`, `hover_widget`) — included for Manor Lords; other games need a one-time discovery pass with `hud_list=1`.
- Per-widget features beyond move/scale/hide (pop-ups floating above the selected building, radial menus) are outside what a UEVR plugin can do — they would require game-side modding.
- Trigger/grip XInput slot mapping differs between UEVR builds; use the config to swap if needed.

## Roadmap

- 🪟 Popup-watch container maps for Frostpunk 2, Homeworld 3, Tempest Rising
- 🕹️ Free camera flight / god-view (world scale & camera offsets on grip + stick)
- 🎡 Radial quick-menu experiment (ImGui overlay)
- 🖱️ Drag & box-select helper for unit selection
- 📳 Haptic ticks when hovering panel buttons

## Building from source

- Visual Studio 2022, C++17 or newer, x64 Release.
- Depends on the [UEVR Plugin API](https://github.com/praydog/UEVR) headers (`uevr/Plugin.hpp`).
- The project uses a precompiled header (`pch.h`); optionally add `NOMINMAX` there before `<windows.h>`.
- Single translation unit: `dllmain.cpp` → builds to `Motion2Mouse.dll` exporting `create_plugin()`.

## Credits

- [praydog](https://github.com/praydog) — UEVR, the foundation that makes all of this possible.
- The UEVR community for tooling, testing and ideas.
- Everyone testing and reporting — this is a Beta, feedback drives it. Open an [Issue](../../issues)!

## License

MIT — see `LICENSE`.
