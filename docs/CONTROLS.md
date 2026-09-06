# Controls

The standard control scheme is awkward in VR. Legs use tank controls, the torso is mapped to the right joystick, and aiming is driven by head look. The proposed control scheme separates those responsibilities:

- Eye gaze → weapon/arm aim
- Head yaw → desired torso rotation relative to the legs
- Left-stick direction → head-relative forward/reverse throttle only
- Right stick left/right → leg rotation

The full left-stick vector is rotated by the raw cockpit-relative head heading and then projected onto the forward/reverse throttle axis; its lateral component is discarded. For example, while looking 90° left, pressing the physical stick right produces forward throttle, while pressing it forward produces no movement. Right-stick vertical input is used internally for head-driven torso pitch and is not a physical control.

### Eye aiming

Your Quest Pro runtime advertises XR_EXT_eye_gaze_interaction in C:\Users\samon\AppData\Roaming\UnrealVRMod\MechWarrior-Win64-Shipping\log.txt:328, so the headset/runtime side is capable.

The integration uses a small UEVR patch to enable the gaze extension, create `/user/eyes_ext/input/gaze_ext/pose`, and expose the resulting pose to `HeadAim.dll`. This must happen while UEVR creates its OpenXR instance; `HeadAim.dll` cannot enable the extension afterward by itself.

The HeadAim plugin keeps weapon aiming separate from physical torso control: both `HeadTarget` (torso-mounted weapons) and `ArmsTarget` use eye gaze, while raw head yaw drives the torso. If gaze is unavailable, both weapon targets fall back to head aiming. Arm aiming continues to respect arm bounds and maximum weapon movement speed. See `MW5-UEVR-Plugins/src/head_aim/HeadAim.cpp`.

The original mod's head-aim pitch offset applies only to head-based fallback. Eye gaze uses the dedicated UEVR weapon-aim yaw and pitch offsets and does not inherit the legacy head offset.

While the player is in a mech, the plugin selects this control scheme automatically; it does not depend on the mod exposing its internal `ArmsOnly`/`TorsoAndArms` enum in the UI.

### Absolute head-driven torso

Head yaw and pitch represent desired torso angles rather than torso turn rates. Returning the head to playspace centre therefore returns the torso toward zero yaw and zero pitch, clamped only if the active mech's bounds exclude zero. No neutral is captured from the head or torso pose when entering a cockpit. Weapon aim and view pitch calibration do not alter this physical torso neutral.

Yaw has a 1° central deadzone. After removing that deadzone, the torso target maps 1:1 through 15°; it then expands linearly and reaches the active mech's full left/right torso range at 75°:

```text
H = absolute head yaw relative to the cockpit, minus the 1° deadzone
M = torso limit in H's direction

if H <= 15°:
    F(H) = H
else:
    F(H) = 15° + min((H - 15°) / 60°, 1) × (M - 15°)
```

`TorsoTwistComponent.TorsoStats` exposes the values required to adapt this mapping to each mech:

- `Torso.BoundsLow.x`: left yaw limit
- `Torso.BoundsHigh.x`: right yaw limit
- `Torso.IsYawUnbound`: continuous-yaw capability
- `Torso.TwistRate.x`: native torso yaw speed

The two directions are calculated independently in case the bounds are asymmetric. For a 360°-capable mech, use a virtual range of -180° to +180°.

Pitch has the same 1° central deadzone and maps 1:1 from playspace level through 15°. Between 15° and 30° of physical pitch, the torso target expands from 15° to the active directional pitch limit. The upper and lower directions are calculated independently from `Torso.BoundsLow.y` and `Torso.BoundsHigh.y`; an unbound pitch axis uses −90° and +90° as virtual limits.

`Head-Driven Torso > View Pitch Offset` on UEVR's Input page adjusts the player's settled view relative to the cockpit. It does not alter the physical torso's world-space neutral or the weapon-aim pitch offset.

`Head-Driven Torso > Maximum Torso Input` caps the magnitude of the virtual torso stick. It defaults to 0.45 so the torso remains below MW5's high-velocity combined traversal behavior; it can be tuned at runtime for the active control settings.

The mapped value is a desired position. A feedback controller compares it with `TorsoTwistComponent.TorsoTwist.x` and generates ordinary torso input:

```text
desired torso yaw = F(H)
torso error       = desired torso yaw - actual torso yaw
torso input       = controller(torso error)
```

Do not write the torso angle directly. Driving the existing controller preserves the native twist speed and mechanical behaviour of each mech.

### Cockpit-relative head view

Rendered head yaw remains 1:1 with the physical headset through 15°. Between 15° and 75°, its settled cockpit-relative angle is compressed linearly from 15° to 20°. Beyond 75°, 1:1 overflow resumes so the head does not become mechanically locked at the torso limit.

Torso-lag compensation is blended in across the 15°–75° outer range. At 15° none of the torso error affects the view; at 75° the full difference between desired and actual torso yaw is temporarily added to the cockpit view. This keeps the outer-range world view responsive while the torso catches up, then settles back toward the mapped 15°–20° cockpit angle as the error disappears.

Rendered pitch remains 1:1 through 15°. Between 15° and 30°, its settled cockpit-relative angle maps from 15° to 20°, with pitch-lag compensation blended from 0% to 100%. Beyond 30°, 1:1 overflow resumes so vertical head movement remains unrestricted.

The yaw compensation and `View Pitch Offset` are applied in render space identically to both stereo eyes. Eye targets receive the same offsets so the gaze reticle stays aligned; the underlying HMD and OpenXR gaze poses remain unmodified.

### Throttle and leg controls

UEVR plugins can inspect and modify XInput through `on_xinput_get_state`, so the simplified controls can continue through MW5's existing input system:

- The transformed left-stick Y component supplies forward/reverse throttle.
- The transformed left-stick X component is discarded.
- Right-stick X supplies leg rotation exclusively.
- Game-facing right-stick X/Y are supplied by the head-driven torso controller.

Use standard Mech controls with Throttle Decay enabled. The plugin supplies player intent through normal inputs, preserving MW5's acceleration, deceleration, and leg-turn behaviour.
