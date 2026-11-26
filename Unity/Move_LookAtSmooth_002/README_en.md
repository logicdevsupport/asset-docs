# LogicDevLookAtSmooth - Smooth Look-At Component Plus

A LookAt component that smoothly tracks a target object. Comes with two variants: a full-body rotation version (`LogicDevLookAtSmooth`) and an IK head-tracking version (`LogicDevLookAtSmoothIK`). This version (Plus) adds Predictive LookAt: it automatically estimates the target's velocity and leads its future position.

---

## Features

- **Two LookAt types included** - Switch between full-body Y-axis rotation and Humanoid IK head tracking
- **Predictive LookAt support (new)** - Automatically estimates the target's velocity and leads its future position. Just enable it to reduce the visible tracking lag on a moving target. When disabled (the default), the aim point always uses Target.position directly, and the final tracking math (Yaw/Pitch) is the same as in versions without this feature (a lightweight internal check for target-reference changes still runs even when disabled)
- **Automatically follows vehicle/drone tilt** - Just attach it: even when the object itself (or its parent) banks or tilts via physics or animation, it keeps tracking the target with the correct sense of "up" - no setup required
- **Real-time Inspector adjustment** - Tweak parameters while in Play Mode and see the result instantly
- **Detects a Humanoid Animator automatically** - On a Humanoid, the head is turned through `OnAnimatorIK` (the Animator Controller's IK Pass must be enabled - see the setup steps below). On a non-Humanoid rig, the bone you assign is rotated directly
- **Angle clamping** - Full-body version clamps the Y axis (left/right); the IK head-tracking version clamps both X and Y axes independently
- **Flexible update timing** - Choose between `Update`, `LateUpdate`, and `FixedUpdate`

---

## Requirements

| Item | Details |
|------|---------|
| Unity Version | Unity 2022.3 LTS / Unity 6.3 LTS |
| Render Pipeline | URP (Universal Render Pipeline). The materials in the included demo scene are URP materials. The component scripts themselves do not use any render-pipeline-specific API |
| Input System | No dependency on Input System |

---

## How to Use

### LogicDevLookAtSmooth (Full Body Rotation)

1. Add the `LogicDevLookAtSmooth` component to the GameObject you want to rotate
2. Assign the target Transform in the **Target** field in the Inspector
3. Adjust **RotationSpeed**, **EnableYAxis**, and angle clamp settings as needed (tilt from a vehicle/drone is followed automatically, so no extra setup is usually needed)
4. Press Play - the object will smoothly follow the target

### LogicDevLookAtSmoothIK (IK Head Tracking)

**Humanoid rigs**

1. Add `LogicDevLookAtSmoothIK` to the GameObject that has the Humanoid Animator (usually the character's root). **It must be on the same GameObject as the Animator.** Unity only calls `OnAnimatorIK` on scripts attached to the Animator's own GameObject, so the head does not move if the component is on a parent or child GameObject (in that case a warning is logged to the Console once at startup)
2. **Enable "IK Pass" on the Animator Controller layer** (Animator window > Layers > gear icon of the layer > check IK Pass). Unity only calls `OnAnimatorIK` for layers that have IK Pass enabled, so the head does not move while it is off (this component does not inspect the IK Pass setting, so no warning is shown)
3. Assign the target Transform in the **Target** field
4. Adjust **RotationSpeed** (how fast the head follows) and **IKWeight** (intensity of the head movement, 0 to 1) as needed

> Note: If the Animator's Culling Mode is anything other than "Always Animate", Unity does not run the Animator's IK step while the character is off screen, so the head direction is not updated during that time.

**Non-Humanoid rigs**

1. Add `LogicDevLookAtSmoothIK` to the character's GameObject
2. Assign the bone to rotate (the head Transform) to the **HeadBone** field
3. Assign the target Transform in the **Target** field

### Using Predictive LookAt (new)

1. Check **Enable Prediction** in the Inspector (default: off. Leaving it off keeps the original behavior)
2. Adjust **Prediction Time** (how far ahead to lead, in seconds), **Velocity Smooth Time** (the time constant for smoothing the estimated velocity), and **Max Prediction Distance** (the cap on the lead offset) as needed. See the Parameter Reference below for details on each
3. The target's velocity is estimated automatically. Internally, if the Target has a Rigidbody (and it is not kinematic), its velocity is used; otherwise, velocity is estimated from the change in position over time. There is no setting to choose between the two - it is detected automatically
4. When you switch targets, whether by calling `SetTarget()` or by assigning the Target field directly, the internal velocity-estimation state resets automatically (the estimate from the previous target never carries over to the new one)
5. The effect of prediction is easiest to see with a fast-moving target
6. **To aim ahead of the target (like leading a shot):** raise **RotationSpeed** so that the facing keeps up with the aim point, and set **Prediction Time** to how many seconds ahead you want to aim (for example, the time your projectile takes to reach the target). Verified reference point: with RotationSpeed=20 (the slider's maximum) and Prediction Time=0.25, on a target passing about 3 m in front and moving back and forth sideways, the facing stayed on average about 3° (at most about 5°) from Predicted Position on the full-body version (on flat ground), and on average about 2.5° (at most about 3.5°) on the IK head-tracking version on a Humanoid (the target also moving up and down). To only cancel out the tracking lag instead, see Prediction Time in the Parameter Reference below. The lead is a linear extrapolation from the target's current velocity (see Known Limitations below)

### Controlling via Code

```csharp
// Change the target at runtime
GetComponent<LogicDevLookAtSmooth>().SetTarget(newTarget);

// Pause tracking
GetComponent<LogicDevLookAtSmooth>().Pause();

// Resume tracking
GetComponent<LogicDevLookAtSmooth>().Resume();

// Read the Predictive LookAt internal state (for verification/debugging, read-only)
Vector3 velocity = GetComponent<LogicDevLookAtSmooth>().EstimatedVelocity;
Vector3 aimPoint = GetComponent<LogicDevLookAtSmooth>().PredictedPosition;
```

The same API works with the IK variant.

---

## Parameter Reference

### LogicDevLookAtSmooth (Full Body Rotation)

| Parameter | Type | Default | Description |
|---|---|---|---|
| Target | Transform | null | The object to look at. When null, rotation stops and the object keeps the direction it had at that moment |
| RotationSpeed | float | 5.0 | Tracking speed (Slerp coefficient) |
| EnableYAxis | bool | true | Enable/disable Y-axis (left/right) rotation |
| ClampAngleY | bool | false | Enable/disable Y-axis angle clamping |
| MinAngleY | float | -90 | Minimum Y-axis angle (degrees). Used when ClampAngleY is on |
| MaxAngleY | float | 90 | Maximum Y-axis angle (degrees). Used when ClampAngleY is on |
| UpAxis | Vector3 | (0, 1, 0) | "Up" direction in local space. **Hidden from the Inspector** (script-only, advanced use - see the Note below) |
| ReferenceForward | Vector3 | (0, 0, 0) | World-space direction used as Yaw=0. When left at (0,0,0) (unset), automatically falls back to this object's own facing direction (transform.forward) at start. Set a non-zero value to use that direction instead. To change it at runtime from code, call SetReferenceForward() (directly assigning the field only takes effect before Awake or when edited via the Inspector) |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

> Note: The full-body component follows the target left/right (Y axis) only. It does not rotate up/down. The up/down rotation setting of the free version (Move_LookAtSmooth_001, EnableXAxis) is not carried over: scenes and prefabs saved in 001 with up/down rotation turned on still follow left/right only in this version. If you also need the up/down direction, use the IK head-tracking component (it turns only the head, both up/down and left/right).

> Note: Keep the width from MinAngleY to MaxAngleY (MaxAngleY - MinAngleY) at 180° or less. The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). When ClampAngleY is on and the width is over 180°, a Warning is shown once in the Console at runtime.

> Note: The full-body component sets the Transform rotation directly every frame. An object whose rotation is also driven by a (non-kinematic) Rigidbody, a CharacterController or another script would fight over the rotation and is out of scope. On such an object, stop the other system from driving the rotation, put the full-body component on a visual child object, or use the IK head-tracking component instead.

> Note: With ClampAngleY on, while the target is outside the angle limits the rotation stays fixed at the limit on the side where the target first left the range. When the target changes to a different one - through `SetTarget()` or by assigning the Target field directly - this is released, and the limit nearest to the new target is chosen again (setting the same target again does not release it; setting it to null first and then assigning a target does, even if it is the same one).

> Note: UpAxis follows the object's own tilt automatically the moment you attach the component, so it normally needs no changes. It's intentionally hidden from the Inspector to prevent the visual "rolling" symptom that manual edits can cause. You can still set it from a script if needed, but manually setting anything other than the default `(0, 1, 0)` makes the object yaw around that axis, which can make it look like it's rotating/rolling.

### LogicDevLookAtSmoothIK (IK Head Tracking)

| Parameter | Type | Default | Description |
|---|---|---|---|
| Target | Transform | null | The object to look at. When null, the head stops turning and keeps its last direction (Humanoid and non-Humanoid alike) |
| HeadBone | Transform | null | Head bone Transform for non-Humanoid rigs |
| RotationSpeed | float | 5.0 | Tracking speed (Slerp coefficient). Applies to both Humanoid and non-Humanoid rigs |
| ClampAngleX | bool | true | Enable/disable X-axis angle clamping |
| MinAngleX | float | -30 | Minimum X-axis angle (degrees) |
| MaxAngleX | float | 30 | Maximum X-axis angle (degrees) |
| ClampAngleY | bool | true | Enable/disable Y-axis angle clamping |
| MinAngleY | float | -60 | Minimum Y-axis angle (degrees) |
| MaxAngleY | float | 60 | Maximum Y-axis angle (degrees) |
| IKWeight | float | 1.0 | IK weight for a Humanoid Animator (0 to 1). Not used on non-Humanoid rigs. Passed straight to Unity's `Animator.SetLookAtWeight`. At 0 the head is in its animated pose. A change in this value is not interpolated; it takes effect within one frame (example from our measurement: with the target about 50° off the character's forward, the head pointed about 48° at 1.0, about 32° at 0.5, and 0° at 0) |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate (used on non-Humanoid rigs. On a Humanoid the component runs at the Animator's IK step instead) |

> Note: Keep the width from MinAngleY to MaxAngleY (MaxAngleY - MinAngleY) at 180° or less (Humanoid and non-Humanoid alike). The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). On a Humanoid the whole body can appear to twist sharply at that moment. When ClampAngleY is on and the width is over 180°, a Warning is shown once in the Console at runtime.

> Note: On a Humanoid, the same thing happens when ClampAngleY is turned off (there is no range at all, so the look direction can swing round past the back - for example when switching to a target on the opposite side - and the head and body can appear to twist sharply). On a Humanoid, keep ClampAngleY on with a width of 180° or less. When ClampAngleY is off on a Humanoid, a Warning is shown once in the Console at runtime.

> Note: With ClampAngleY on, while the target is outside the angle limits the rotation stays fixed at the limit on the side where the target first left the range. When the target changes to a different one - through `SetTarget()` or by assigning the Target field directly - this is released, and the limit nearest to the new target is chosen again (setting the same target again does not release it; setting it to null first and then assigning a target does, even if it is the same one).

> Note: On a Humanoid, the head holds its last direction while `Pause()` is in effect and while Target is null (it does not return to the animated pose). To return to the animated pose, set IKWeight to 0 or disable the component (enabled = false). Either way, the head returns to the animated pose within one frame (to return smoothly, change IKWeight gradually from a script). When it is enabled again, the head turns from the character's forward direction toward the target at RotationSpeed.

> Note: When used on a Humanoid character, setting a very wide angle range (Min/Max close to 180°) may not be fully achievable in practice - the character's own neck/torso muscle definition limits how far it can actually turn, independent of these settings. Making a character naturally look directly behind itself using only head/torso IK (without turning the body) is a structural limitation of Unity's built-in Humanoid LookAt IK system. Normal usage ranges (front to side) work reliably.

### Predictive LookAt Parameters (new, shared by both components)

Both the full-body version and the IK head-tracking version support the same set of parameters below.

| Parameter | Type | Default | Description |
|---|---|---|---|
| Enable Prediction | bool | false | Enables/disables predictive lead. When false, the aim point always uses Target.position directly, and the final tracking math behaves the same as a version without this feature (a lightweight internal check for target-reference changes still runs even when disabled) |
| Prediction Time | float | 0.25 | How far ahead to lead, in seconds. The actual aim point is "current target position + estimated velocity x Prediction Time". **Verified reference point**: with RotationSpeed=5 and Velocity Smooth Time=0.1, on the full-body version (a target moving in a circle) tracking lag is nearly eliminated around Prediction Time=0.25 (the difference from the target's direction was under 1°). On the IK head-tracking version on a Humanoid (a target moving back and forth sideways, two different motions), an average lag of 7 to 10° with prediction off became an average of 0.1 to 2.4° (slightly ahead of the target) at Prediction Time 0.25. Other RotationSpeed values have not been tested. **If you change RotationSpeed, re-tune Prediction Time by watching where Predicted Position (below) actually points**. Which way to adjust: Prediction Time exists to cancel out the tracking lag caused by RotationSpeed, so **if you raise RotationSpeed (faster tracking), lower Prediction Time; if you lower RotationSpeed (slower tracking), raise Prediction Time** |
| Velocity Smooth Time | float | 0.1 | Time constant, in seconds, for smoothing the estimated velocity. Larger values are more resistant to noise but react more slowly to real speed changes. 0 or less disables smoothing |
| Max Prediction Distance | float | 10 | Caps the lead offset (estimated velocity x Prediction Time), in world units. **This value has a second role**: if the target's position moves further than this distance in a single step, it is treated as an instant reposition (teleport), and velocity estimation resets once (re-converging from zero). **Setting this value too small can cause a target that is genuinely moving fast to be misidentified as having teleported.** 0 or less disables both the offset cap and teleport detection |
| Estimated Velocity | Vector3 (read-only) | - | The currently estimated velocity of the target. Exposed for verification/debugging purposes |
| Predicted Position | Vector3 (read-only) | - | The actual position currently being aimed at. Exposed for verification/debugging purposes |

---

## Known Limitations (Predictive LookAt)

- Predictive LookAt is a simple linear extrapolation based on the target's **current** velocity. Error increases when the target is accelerating, decelerating, or turning (its velocity direction is changing)
- It does not include ballistic calculations or an interception solution that accounts for projectile speed
- The parameters above work identically on both the full-body version (`LogicDevLookAtSmooth`) and the IK head-tracking version (`LogicDevLookAtSmoothIK`)
- For a target with no Rigidbody whose position is updated in FixedUpdate, Estimated Velocity can fluctuate due to the mismatch between the render frame rate and the physics update rate (in our measurements, up to about 10% relative to the true value). Two things help: (1) attach a non-kinematic Rigidbody to the target and move it via physics (or by assigning velocity directly) - the component then reads the Rigidbody's velocity directly and this fluctuation goes away; (2) set Velocity Smooth Time higher (this trades slower reaction to real speed changes for stronger smoothing)

---

## Demo Scene

You can check the component behavior immediately using the included demo scene `Demo_LookAtSmooth` (`Assets/LogicDevSupport/Move_LookAtSmooth_002/Demo/`).

- **Left side:** Full-body rotation demo - a blue capsule smoothly tracks a yellow sphere, mounted on a tilting platform to demonstrate automatic tracking through vehicle/drone-style banking
- **Right side:** IK head tracking demo - a character rotates only its head to follow a red sphere. The red sphere moves up and down as well as side to side, so you can also see the head tracking vertically
- **Predictive LookAt (new):** both demos automatically switch prediction OFF / ON every 4.5 seconds (starting with OFF). A label above each character shows the current state (`Prediction: OFF` / `Prediction: ON`), and while prediction is ON, a small magenta marker shows the position actually being aimed at (Predicted Position). Comparing OFF and ON on the same character and the same target makes the difference in tracking lag easy to see

> Note: the automatic OFF / ON switching in the demo is done by a demo-only script, `DemoPredictionToggle` (in `Demo/Scripts/`). You do not need this script to use Predictive LookAt in your own scenes - just set the component's **Enable Prediction** directly.

> Note: The demo scene uses primitive shapes only. A demo using Starter Assets is available on the Asset Store page (not included in this package).

---

## Before You Import (Important)

If the free version (`Move_LookAtSmooth_001`) is in your project, please delete it before importing this package. It uses the same class names, so importing both causes compile errors (duplicate class definitions).

**Updating this package**: if you already have an earlier release of this package (it was sold as Smooth LookAt Component Standard; folder `Move_LookAtSmooth_002`), you do not need to delete anything. Update it from the Package Manager (My Assets): the files are replaced inside the same `Move_LookAtSmooth_002` folder, and the components in your scenes and prefabs keep working with their settings.

If you have edited any files inside the package folder (such as the demo scene), please back them up before updating or deleting it.

---

## Updating from Smooth LookAt Component Standard

This package used to be sold as Smooth LookAt Component Standard. It is now Smooth LookAt Component Plus: everything in Standard, plus Predictive LookAt.

- Predictive LookAt is off by default, so unless you explicitly enable it, the components behave as before
- Your existing component settings on scenes and prefabs (RotationSpeed, ClampAngleY, UpAxis, etc.) are kept when you update. The new prediction settings start at their documented defaults

---

## Upgrade Path

The final and most complete version of the series, **Smooth LookAt Component Complete Edition**, is available on the Unity Asset Store. In addition to everything in this version, it automatically picks which target to look at from a list of candidates (nearest, highest priority or most in front) and switches between them, can ignore targets outside a field of view or behind walls, fades the head look in and out, and includes Scene-view gizmos and development tools (a Game-view debug display and a runtime tuning panel).

https://assetstore.unity.com/packages/slug/410220

The free version (`Move_LookAtSmooth_001`) is available here:

https://assetstore.unity.com/packages/slug/389610

Videos: overview https://youtu.be/eowEC9rYavM / short https://youtu.be/1GorMFFqxgQ / channel https://www.youtube.com/@LogicDevSupport

---

If you find this asset useful, please consider leaving a review - it really helps!

You can leave a review from the "Reviews" tab on the store page below.

https://assetstore.unity.com/packages/slug/404560
