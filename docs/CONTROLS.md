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

Head position uses one continuous ease-in power curve with no deadzone. Its exponent is 1.5 and the torso mapping has a 1:1 floor: the desired torso rotation is never less than the physical head rotation unless the mech has reached its mechanical limit. The curve reaches the active mech's full directional torso range at 80° of head yaw and 30° of head pitch:

```text
H = absolute head angle relative to playspace centre
I = maximum input head angle (80° yaw, 30° pitch)
M = torso travel from neutral to the limit in H's direction

R(H) = (H / I)^1.5
F(H) = min(max(H, R(H) × M), M)
```

`TorsoTwistComponent.TorsoStats` exposes the values required to adapt this mapping to each mech:

- `Torso.BoundsLow.x`: left yaw limit
- `Torso.BoundsHigh.x`: right yaw limit
- `Torso.IsYawUnbound`: continuous-yaw capability
- `Torso.TwistRate.x`: native torso yaw speed

The two directions are calculated independently in case the bounds are asymmetric. For a 360°-capable mech, use a virtual range of -180° to +180°.

The upper and lower pitch directions are calculated independently from `Torso.BoundsLow.y` and `Torso.BoundsHigh.y`; an unbound pitch axis uses −90° and +90° as virtual limits. The curve saturates at the relevant mechanical limit.

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

The settled cockpit-relative view uses the same unbounded curve as torso position:

```text
cockpit offset = 15° × R(H)
```

It reaches 15° at 80° of physical yaw and at 30° of physical pitch. Unlike torso position, the curve is not clamped at that point.

As in the original `head aim torso v1` implementation, the full difference between desired and actual torso position is added to the cockpit-relative view while the mech catches up. It has no separate blend threshold and is not clamped to the 15° directional offset. Once the torso catches up, the error disappears and the view settles at the curve offset.

Physical head movement beyond 80° yaw or 30° pitch continues along the same power curve. This keeps head rotation unrestricted when the mech is at its mechanical limit without switching to a separate linear overflow response. Lag error and the continuing curve can both carry the cockpit-relative view past the nominal 15° endpoint.

The yaw and pitch adjustment is applied identically to both stereo eyes before UEVR composes the HMD pose. Before applying it, the plugin calculates the roll that the untouched cockpit and physical HMD pose would produce. After stereo composition, only that original roll component is restored; the completed cockpit/world transform, position, yaw, pitch, and stereo separation are not reconstructed or replaced. Eye targets receive the same yaw/pitch offsets so the gaze reticle stays aligned; the underlying HMD and OpenXR gaze poses remain unmodified.

### Throttle and leg controls

UEVR plugins can inspect and modify XInput through `on_xinput_get_state`, so the simplified controls can continue through MW5's existing input system:

- The transformed left-stick Y component supplies forward/reverse throttle.
- The transformed left-stick X component is discarded.
- Right-stick X supplies leg rotation exclusively.
- Game-facing right-stick X/Y are supplied by the head-driven torso controller.

Use standard Mech controls with Throttle Decay enabled. The plugin supplies player intent through normal inputs, preserving MW5's acceleration, deceleration, and leg-turn behaviour.
