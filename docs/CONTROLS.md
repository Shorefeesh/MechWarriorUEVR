# Controls

The standard control scheme is awkward in VR. Legs use tank controls, the torso is mapped to the right joystick, and aiming is driven by head look. The proposed control scheme separates those responsibilities:

- Eye gaze -> weapon/arm aim
- Head rotation -> in-cockpit view
- Controller inputs -> Use mouse and keyboard / HOTAS instead

### Eye aiming

Your Quest Pro runtime advertises XR_EXT_eye_gaze_interaction in C:\Users\samon\AppData\Roaming\UnrealVRMod\MechWarrior-Win64-Shipping\log.txt:328, so the headset/runtime side is capable.

The integration uses a small UEVR patch to enable the gaze extension, create `/user/eyes_ext/input/gaze_ext/pose`, and expose the resulting pose to `HeadAim.dll`. This must happen while UEVR creates its OpenXR instance; `HeadAim.dll` cannot enable the extension afterward by itself.

The HeadAim plugin keeps weapon aiming separate from physical torso control: both `HeadTarget` (torso-mounted weapons) and `ArmsTarget` use eye gaze, while raw head yaw drives the torso. If gaze is unavailable, both weapon targets fall back to head aiming. Arm aiming continues to respect arm bounds and maximum weapon movement speed. See `MW5-UEVR-Plugins/src/head_aim/HeadAim.cpp`.

The original mod's head-aim pitch offset applies only to head-based fallback. Eye gaze uses the dedicated UEVR weapon-aim yaw and pitch offsets and does not inherit the legacy head offset.

While the player is in a mech, the plugin selects this control scheme automatically; it does not depend on the mod exposing its internal `ArmsOnly`/`TorsoAndArms` enum in the UI.

### Eye-directed zoom

While zoom is active, the HUD plugin redirects the camera at the end of the HUD manager's tick and moves the zoom-display mesh, outline, and captured weapon-target overlay with the eye-driven arm-target reticle. It bypasses the original torso-centered zoom compensation while calculating the reticle position. The torso reticle remains in its normal position. The plugin also raises both zoom scene-capture textures from the mod's 320x215 default to 640x430; it preserves any already-higher resolution.
