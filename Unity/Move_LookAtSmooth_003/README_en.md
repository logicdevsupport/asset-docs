# LogicDevLookAtSmooth - Smooth Look-At Component Plus

A LookAt component that smoothly tracks a target object. Comes with two variants: a full-body rotation version (`LogicDevLookAtSmooth`) and an IK head-tracking version (`LogicDevLookAtSmoothIK`). This version (Plus) adds Predictive LookAt: it automatically estimates the target's velocity and leads its future position.

---

## Features

- **Two LookAt types included** - Switch between full-body Y-axis rotation and Humanoid IK head tracking
- **Predictive LookAt support (new)** - Automatically estimates the target's velocity and leads its future position. Just enable it to reduce the visible tracking lag on a moving target. When disabled (the default), the aim point always uses Target.position directly, and the final tracking math (Yaw/Pitch) is the same as in versions without this feature (a lightweight internal check for target-reference changes still runs even when disabled)
- **Automatically follows vehicle/drone tilt** - Just attach it: even when the object itself (or its parent) banks or tilts via physics or animation, it keeps tracking the target with the correct sense of "up" - no setup required
- **Real-time Inspector adjustment** - Tweak parameters while in Play Mode and see the result instantly
- **Automatic Humanoid Animator support** - Uses `OnAnimatorIK` for natural head movement on animated characters
- **Angle clamping** - Full-body version clamps the Y axis (left/right); the IK head-tracking version clamps both X and Y axes independently
- **Flexible update timing** - Choose between `Update`, `LateUpdate`, and `FixedUpdate`

---

## Requirements

| Item | Details |
|------|---------|
| Unity Version | Unity 2022.3 LTS / Unity 6.3 LTS |
| Render Pipeline | URP (Universal Render Pipeline) |
| Input System | No dependency on Input System |

---

## How to Use

### LogicDevLookAtSmooth (Full Body Rotation)

1. Add the `LogicDevLookAtSmooth` component to the GameObject you want to rotate
2. Assign the target Transform in the **Target** field in the Inspector
3. Adjust **RotationSpeed**, **EnableYAxis**, and angle clamp settings as needed (tilt from a vehicle/drone is followed automatically, so no extra setup is usually needed)
4. Press Play - the object will smoothly follow the target

### LogicDevLookAtSmoothIK (IK Head Tracking)

1. Add `LogicDevLookAtSmoothIK` to the root GameObject of a character with a Humanoid Animator
2. Assign the target Transform in the **Target** field
3. **For non-Humanoid rigs**, assign the head bone Transform to the **HeadBone** field
4. Adjust **IKWeight** (0 to 1) to control the intensity of the head movement

### Using Predictive LookAt (new)

1. Check **Enable Prediction** in the Inspector (default: off. Leaving it off keeps the original behavior)
2. Adjust **Prediction Time** (how far ahead to lead, in seconds), **Velocity Smooth Time** (the time constant for smoothing the estimated velocity), and **Max Prediction Distance** (the cap on the lead offset) as needed. See the Parameter Reference below for details on each
3. The target's velocity is estimated automatically. Internally, if the Target has a Rigidbody (and it is not kinematic), its velocity is used; otherwise, velocity is estimated from the change in position over time. There is no setting to choose between the two - it is detected automatically
4. When you switch targets, whether by calling `SetTarget()` or by assigning the Target field directly, the internal velocity-estimation state resets automatically (the estimate from the previous target never carries over to the new one)
5. The effect of prediction is easiest to see with a fast-moving target

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
| Target | Transform | null | The object to look at. Stops tracking when null |
| RotationSpeed | float | 5.0 | Tracking speed (Slerp coefficient) |
| EnableYAxis | bool | true | Enable/disable Y-axis (left/right) rotation |
| ClampAngleY | bool | false | Enable/disable Y-axis angle clamping |
| MinAngleY | float | -90 | Minimum Y-axis angle (degrees). Used when ClampAngleY is on |
| MaxAngleY | float | 90 | Maximum Y-axis angle (degrees). Used when ClampAngleY is on |
| UpAxis | Vector3 | (0, 1, 0) | "Up" direction in local space. **Hidden from the Inspector** (script-only, advanced use - see the Note below) |
| ReferenceForward | Vector3 | (0, 0, 0) | World-space direction used as Yaw=0. When left at (0,0,0) (unset), automatically falls back to this object's own facing direction (transform.forward) at start. Set a non-zero value to use that direction instead. To change it at runtime from code, call SetReferenceForward() (directly assigning the field only takes effect before Awake or when edited via the Inspector) |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

> Note: UpAxis follows the object's own tilt automatically the moment you attach the component, so it normally needs no changes. It's intentionally hidden from the Inspector to prevent the visual "rolling" symptom that manual edits can cause. You can still set it from a script if needed, but manually setting anything other than the default `(0, 1, 0)` makes the object yaw around that axis, which can make it look like it's rotating/rolling.

### LogicDevLookAtSmoothIK (IK Head Tracking)

| Parameter | Type | Default | Description |
|---|---|---|---|
| Target | Transform | null | The object to look at. Stops tracking when null |
| HeadBone | Transform | null | Head bone Transform for non-Humanoid rigs |
| RotationSpeed | float | 5.0 | Tracking speed (Slerp coefficient) |
| ClampAngleX | bool | true | Enable/disable X-axis angle clamping |
| MinAngleX | float | -30 | Minimum X-axis angle (degrees) |
| MaxAngleX | float | 30 | Maximum X-axis angle (degrees) |
| ClampAngleY | bool | true | Enable/disable Y-axis angle clamping |
| MinAngleY | float | -60 | Minimum Y-axis angle (degrees) |
| MaxAngleY | float | 60 | Maximum Y-axis angle (degrees) |
| IKWeight | float | 1.0 | IK weight for Humanoid Animator (0 to 1) |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

> Note: When used on a Humanoid character, setting a very wide angle range (Min/Max close to 180°) may not be fully achievable in practice - the character's own neck/torso muscle definition limits how far it can actually turn, independent of these settings. Making a character naturally look directly behind itself using only head/torso IK (without turning the body) is a structural limitation of Unity's built-in Humanoid LookAt IK system. Normal usage ranges (front to side) work reliably.

### Predictive LookAt Parameters (new, shared by both components)

Both the full-body version and the IK head-tracking version support the same set of parameters below.

| Parameter | Type | Default | Description |
|---|---|---|---|
| Enable Prediction | bool | false | Enables/disables predictive lead. When false, the aim point always uses Target.position directly, and the final tracking math behaves the same as a version without this feature (a lightweight internal check for target-reference changes still runs even when disabled) |
| Prediction Time | float | 0.25 | How far ahead to lead, in seconds. The actual aim point is "current target position + estimated velocity x Prediction Time". **Verified reference point**: with RotationSpeed=5 and Velocity Smooth Time=0.1, testing with a target moving in a circle showed that tracking lag is nearly eliminated around Prediction Time=0.25 (other RotationSpeed values have not been tested). **If you change RotationSpeed, re-tune Prediction Time by watching where Predicted Position (below) actually points**. Which way to adjust: Prediction Time exists to cancel out the tracking lag caused by RotationSpeed, so **if you raise RotationSpeed (faster tracking), lower Prediction Time; if you lower RotationSpeed (slower tracking), raise Prediction Time** |
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

You can check the component behavior immediately using the included demo scene `Demo_LookAtSmooth` (`Assets/LogicDevSupport/Move_LookAtSmooth_003/Demo/`).

- **Left side:** Full-body rotation demo - a blue capsule smoothly tracks a yellow sphere, mounted on a tilting platform to demonstrate automatic tracking through vehicle/drone-style banking
- **Right side:** IK head tracking demo - a character rotates only its head to follow a red sphere. The red sphere moves up and down as well as side to side, so you can also see the head tracking vertically
- **Predictive LookAt (new):** both demos automatically switch prediction OFF / ON every 4.5 seconds (starting with OFF). A label above each character shows the current state (`Prediction: OFF` / `Prediction: ON`), and while prediction is ON, a small magenta marker shows the position actually being aimed at (Predicted Position). Comparing OFF and ON on the same character and the same target makes the difference in tracking lag easy to see

> Note: the automatic OFF / ON switching in the demo is done by a demo-only script, `DemoPredictionToggle` (in `Demo/Scripts/`). You do not need this script to use Predictive LookAt in your own scenes - just set the component's **Enable Prediction** directly.

> Note: The demo scene uses primitive shapes only. A demo using Starter Assets is available on the Asset Store page (not included in this package).

---

## Before You Import (Important)

Before importing this package, please delete both previous versions (Move_LookAtSmooth_001 and Move_LookAtSmooth_002) from your project. Importing without deleting them can cause shared scripts or scene files to be silently overwritten, so a folder's name may no longer match what it actually contains. Depending on which previous version you have, it may also result in compile errors.

If you have edited any files inside a previous version's folder (such as the demo scene), please back them up before deleting it.

---

## Upgrading from 002

Notes for users of Move_LookAtSmooth_002:

- Predictive LookAt is off by default, so unless you explicitly enable it, the component follows the same code path as 002
- Deleting the 002 folder before importing 003 carries over your existing component settings on scenes and prefabs (RotationSpeed, ClampAngleY, etc.) automatically (verified on Unity 2022.3, covering both the Full Body and IK components and a Prefab; newly added fields were also confirmed to initialize to their documented defaults)
- As noted above, please also delete 002 itself from your project before importing

---

## Upgrade Path

An upgraded version (`Move_LookAtSmooth_004`) is in development. Stay tuned!

The free version (`Move_LookAtSmooth_001`) is available here:

https://assetstore.unity.com/packages/slug/389610

---

If you find this asset useful, please consider leaving a review - it really helps!

You can leave a review from the "Reviews" tab on the store page below.

https://assetstore.unity.com/packages/slug/410220
