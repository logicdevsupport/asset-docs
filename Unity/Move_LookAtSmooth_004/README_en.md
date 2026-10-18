# LogicDevLookAtSmooth - Smooth Look-At Component Advanced

A LookAt component that smoothly tracks a target object. Comes with two variants: a full-body rotation version (`LogicDevLookAtSmooth`) and an IK head-tracking version (`LogicDevLookAtSmoothIK`). This version (Advanced) adds a component that automatically picks which target to look at from a list of candidates and switches between them (`LogicDevLookAtTargetSelector`).

---

## Features

- **Two LookAt types included** - Switch between full-body Y-axis rotation and IK head tracking
- **Automatic multi-target switching (new)** - Picks "the nearest candidate" or "the highest-priority candidate (the nearest one among equal priorities)" from a candidate list and switches the look-at target automatically. Includes two settings that prevent switching too often (hold time and switch margin) and an event that tells you when the target changes
- **Predictive LookAt support** - Automatically estimates the target's velocity and leads its future position. When disabled (the default), the component aims at Target.position directly
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
| Input System | The components do not depend on any input system. Only the M-key mode switch in the included demo uses the New Input System (if it is not installed, only that key switch is disabled) |

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

### Using Automatic Multi-Target Switching (new)

1. Add `LogicDevLookAtTargetSelector` to the GameObject that has `LogicDevLookAtSmooth` or `LogicDevLookAtSmoothIK` (Add Component > LogicDevSupport > Move > LookAt Target Selector). If both **LookAtComponent** and **LookAtIKComponent** are left empty, the components on the same GameObject are used automatically
2. Add the Transforms that may be looked at to **Candidates**. If you plan to use priority-based selection, also set each candidate's **Priority** (0 to 100, higher wins)
3. Choose how to pick with **SelectionMode**. `Nearest` picks the closest candidate (distance is measured from the position of the GameObject that has `LogicDevLookAtTargetSelector`). `Priority` picks the candidate with the highest Priority value; if several candidates share that value, the closest one among them is picked
4. If it switches too often, adjust **MinHoldTime** (how many seconds to keep a target after switching to it) and **SwitchMargin** (how much closer a candidate must be before switching to it). Setting either one to 0 turns it off
5. The selected target is assigned to the LookAt component's **Target** automatically. You can leave the LookAt component's own Target field empty (even if it is set, it is overwritten with the target chosen by `LogicDevLookAtTargetSelector`)

Candidates can also be added and removed from a script.

```csharp
var selector = GetComponent<LogicDevLookAtTargetSelector>();

// Add a candidate (the second argument is its priority; 0 if omitted).
// Passing a Transform that is already a candidate only updates its priority
selector.AddCandidate(enemy.transform, 5);

// Change a candidate's priority
selector.SetCandidatePriority(enemy.transform, 10);

// Remove a candidate
selector.RemoveCandidate(enemy.transform);

// The currently selected target (null when no candidate is available)
Transform current = selector.CurrentTarget;

// Get notified when the target changes (previous target, new target)
selector.OnTargetChanged.AddListener((previous, next) =>
{
    Debug.Log("Target changed: " + (next != null ? next.name : "none"));
});
```

`OnTargetChanged` can also be wired up in the Inspector.

### Using Predictive LookAt

1. Check **Enable Prediction** in the Inspector (default: off)
2. Adjust **Prediction Time** (how far ahead to lead, in seconds), **Velocity Smooth Time** (the time constant for smoothing the estimated velocity), and **Max Prediction Distance** (the cap on the lead offset) as needed. See the Parameter Reference below for details on each
3. The target's velocity is estimated automatically. Internally, if the Target has a Rigidbody (and it is not kinematic), its velocity is used; otherwise, velocity is estimated from the change in position over time. There is no setting to choose between the two - it is detected automatically
4. Whenever the target changes - by calling `SetTarget()`, by assigning the Target field directly, or through `LogicDevLookAtTargetSelector` - the internal velocity-estimation state resets automatically (the estimate from the previous target never carries over to the new one). Right after a switch the component starts with no lead (the new target's current position) and then estimates the velocity again
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

> Note: While `LogicDevLookAtTargetSelector` is enabled, a target set manually with `SetTarget()` is overwritten on the next update. To choose the target manually for a while, disable `LogicDevLookAtTargetSelector` (enabled = false).

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

> Note: The full-body component follows the target left/right (Y axis) only. It does not rotate up/down. The up/down rotation setting of the free version (Move_LookAtSmooth_001, EnableXAxis) is not carried over: scenes and prefabs saved in 001 with up/down rotation turned on still follow left/right only in this version.

> Note: Keep the width from MinAngleY to MaxAngleY (MaxAngleY - MinAngleY) at 180° or less. The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). When ClampAngleY is on and the width is over 180°, a Warning is shown once in the Console at runtime.

> Note: The full-body component sets the Transform rotation directly every frame. An object whose rotation is also driven by a (non-kinematic) Rigidbody, a CharacterController or another script would fight over the rotation and is out of scope. On such an object, stop the other system from driving the rotation, put the full-body component on a visual child object, or use the IK head-tracking component instead.

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

> Note: On a Humanoid, the head holds its last direction while `Pause()` is in effect and while Target is null (it does not return to the animated pose). To return to the animated pose, set IKWeight to 0 or disable the component (enabled = false). Either way, the head returns to the animated pose within one frame (to return smoothly, change IKWeight gradually from a script). When it is enabled again, the head turns from the character's forward direction toward the target at RotationSpeed.

> Note: Keep the width from MinAngleY to MaxAngleY (MaxAngleY - MinAngleY) at 180° or less (Humanoid and non-Humanoid alike). The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). On a Humanoid the whole body can appear to twist sharply at that moment. When ClampAngleY is on and the width is over 180°, a Warning is shown once in the Console at runtime.

> Note: On a Humanoid, the same thing happens when ClampAngleY is turned off (there is no range at all, so the look direction can swing round past the back - for example when switching to a target on the opposite side - and the head and body can appear to twist sharply). On a Humanoid, keep ClampAngleY on with a width of 180° or less. When ClampAngleY is off on a Humanoid, a Warning is shown once in the Console at runtime.

> Note: When used on a Humanoid character, setting a very wide angle range (Min/Max close to 180°) may not be fully achievable in practice - the character's own neck/torso muscle definition limits how far it can actually turn, independent of these settings. Making a character naturally look directly behind itself using only head/torso IK (without turning the body) is a structural limitation of Unity's built-in Humanoid LookAt IK system.

### LogicDevLookAtTargetSelector (Automatic Multi-Target Switching, new)

| Parameter | Type | Default | Description |
|---|---|---|---|
| LookAtComponent | LogicDevLookAtSmooth | null | The full-body component that receives the selected target. If both this and LookAtIKComponent are empty, the components on the same GameObject are found automatically |
| LookAtIKComponent | LogicDevLookAtSmoothIK | null | The IK head-tracking component that receives the selected target (same rule as above). If both are set, the same target is given to both |
| Candidates | List | empty | The candidate list. Each entry has a **Target** (Transform) and a **Priority** (int, 0 to 100 in the Inspector). Empty, destroyed, or inactive candidates are never selected |
| SelectionMode | enum | Nearest | `Nearest`: the closest candidate. `Priority`: the candidate with the highest Priority (the closer one if the values are equal). Priority values are not used in Nearest mode |
| MinHoldTime | float | 0.5 | After switching targets, the component does not switch again for this many seconds, even if a better candidate appears (0 to 5; 0 = off). When the current target becomes unavailable, and right after SelectionMode is changed, it picks again without waiting for this time |
| SwitchMargin | float | 0.1 | How much better a candidate must be before switching between candidates of the same Priority (0 to 0.9; 0 = off). At 0.1, it switches only when the new candidate is more than 10% closer than the current target. In Priority mode, a candidate with a higher Priority is switched to without this margin (MinHoldTime still applies) |
| OnTargetChanged | `UnityEvent<Transform, Transform>` | - | Raised whenever the selected target changes. The arguments are (previous target, new target); either can be null. The previous target can also be an object that has already been destroyed |

**Script API**

| Member | Description |
|---|---|
| `void AddCandidate(Transform target, int priority = 0)` | Adds a candidate. If the Transform is already a candidate, only its priority is updated. Passing null logs a warning and is ignored |
| `bool RemoveCandidate(Transform target)` | Removes a candidate. Returns false if it was not in the list |
| `bool SetCandidatePriority(Transform target, int priority)` | Changes a candidate's priority. If it is not in the list, logs a warning and returns false |
| `Transform CurrentTarget { get; }` | The currently selected target (read-only). null when no candidate is available |

`Candidates`, `SelectionMode`, `MinHoldTime`, and `SwitchMargin` are public fields, so you can also change them directly from a script at runtime.

In addition, `void ResetClampState()` has been added to both the full-body and the IK head-tracking component (see "Angle clamp side" under "Behavior" below).

### Predictive LookAt Parameters (shared by both components)

Both the full-body version and the IK head-tracking version support the same set of parameters below.

| Parameter | Type | Default | Description |
|---|---|---|---|
| Enable Prediction | bool | false | Enables/disables predictive lead. When false, the aim point always uses Target.position directly (a lightweight internal check for target-reference changes still runs even when disabled) |
| Prediction Time | float | 0.25 | How far ahead to lead, in seconds. The actual aim point is "current target position + estimated velocity x Prediction Time". **Verified reference point**: with RotationSpeed=5 and Velocity Smooth Time=0.1, on the full-body version (a target moving in a circle) tracking lag is nearly eliminated around Prediction Time=0.25 (the difference from the target's direction was under 1°). On the IK head-tracking version on a Humanoid (a target moving back and forth sideways, two different motions), an average lag of 7 to 10° with prediction off became an average of 0.1 to 2.4° (slightly ahead of the target) at Prediction Time 0.25. Other RotationSpeed values have not been tested. **If you change RotationSpeed, re-tune Prediction Time by watching where Predicted Position (below) actually points**. Which way to adjust: Prediction Time exists to cancel out the tracking lag caused by RotationSpeed, so **if you raise RotationSpeed (faster tracking), lower Prediction Time; if you lower RotationSpeed (slower tracking), raise Prediction Time** |
| Velocity Smooth Time | float | 0.1 | Time constant, in seconds, for smoothing the estimated velocity. Larger values are more resistant to noise but react more slowly to real speed changes. 0 or less disables smoothing |
| Max Prediction Distance | float | 10 | Caps the lead offset (estimated velocity x Prediction Time), in world units. **This value has a second role**: if the target's position moves further than this distance in a single step, it is treated as an instant reposition (teleport), and velocity estimation resets once (re-converging from zero). **Setting this value too small can cause a target that is genuinely moving fast to be misidentified as having teleported.** 0 or less disables both the offset cap and teleport detection |
| Estimated Velocity | Vector3 (read-only) | - | The currently estimated velocity of the target. Exposed for verification/debugging purposes |
| Predicted Position | Vector3 (read-only) | - | The actual position currently being aimed at. Exposed for verification/debugging purposes |

---

## LogicDevLookAtTargetSelector Behavior

| Situation | What happens |
|---|---|
| The current target is destroyed, removed from the candidates, or deactivated | A new target is picked from the remaining candidates without waiting for MinHoldTime. With `Destroy()`, this happens on the next frame, when the destruction takes effect |
| No candidate is available | The LookAt component's Target is set to null. The LookAt component stops rotating and keeps its last direction. `OnTargetChanged` is raised with (previous target, null). If a candidate is added later, it is selected right away |
| A better candidate appears or is added while MinHoldTime is running | The switch happens after the hold time ends |
| A candidate's Priority is changed | Takes effect from the next evaluation. If a hold is running, the switch happens after the hold time ends |
| SelectionMode is changed | For the first evaluation after the change only, a new target is picked without waiting for MinHoldTime (SwitchMargin still applies) |
| `SetTarget()` is called manually on the LookAt component | It is overwritten with the selected target at the next evaluation. `OnTargetChanged` is not raised |
| The LookAt component is paused with `Pause()` | Selection continues, and Target updates and `OnTargetChanged` happen as usual. Rotation stays stopped; after `Resume()` the component turns toward whichever target is selected at that time |
| `LogicDevLookAtTargetSelector` is disabled | Selection stops. The LookAt component's Target stays at the last selected target. Selection starts again when it is re-enabled |
| No LookAt component is found or assigned | Only target selection and `OnTargetChanged` take place (a warning is logged at startup) |
| Angle clamp side (when ClampAngleY is on) | While the target is outside the angle limits, the LookAt component stays fixed at the limit on the side where the target first left the range. When the target changes to a different one - switched by `LogicDevLookAtTargetSelector`, through `SetTarget()`, or by assigning the Target field directly - this is released and the limit nearest to the new target is chosen again. Setting the same target again does not release it (setting it to null first and then assigning a target does, even if it is the same one). To release it without changing the target, call `ResetClampState()` |
| Predictive LookAt is enabled | Velocity estimation resets on every switch and starts again from the new target's current position |

- Selection is evaluated at the moment the LookAt component actually runs (its UpdateMode; for the IK head-tracking version on a Humanoid, this is Update when the Animator's Update Mode is Normal, and FixedUpdate when it is Animate Physics - called Fixed in Unity 6), and before the LookAt component. A switch is reflected in the rotation of that same frame
- Distance is the straight-line 3D distance measured from the position (transform.position) of the GameObject that has `LogicDevLookAtTargetSelector`. This is also true when it drives the IK head-tracking version: the reference point is the GameObject's position, not the head

**Example of what MinHoldTime and SwitchMargin do (our measurement)**: with two candidates swaying back and forth at nearly the same distance (so that "the nearer one" changes 46 times in 10 seconds) and the full-body version selecting between them, the number of switches was 46 with both set to 0, 15 with only MinHoldTime 0.5, 25 with only SwitchMargin 0.1, and 11 with both (the default values).

---

## Known Limitations

**Automatic multi-target switching**

- There are two selection modes: `Nearest` (distance) and `Priority`. Field of view, a maximum distance, and occlusion (whether the candidate is behind a wall) are not taken into account. To exclude a candidate that does not meet your conditions, remove it with `RemoveCandidate()` or deactivate its GameObject
- When Predictive LookAt is enabled, velocity estimation starts over from zero every time the target switches. Frequent switching leaves less time with an effective lead, so use MinHoldTime and SwitchMargin to keep the switching rate down
- When driving the IK head-tracking version on a Humanoid (Animator Update Mode: Normal), selection is evaluated early in Update (execution order -100). If your candidates are moved in Update at the default execution order, the evaluation uses their positions from the previous frame (a change in which one is nearer is detected one frame late)

**Predictive LookAt**

- Predictive LookAt is a simple linear extrapolation based on the target's **current** velocity. Error increases when the target is accelerating, decelerating, or turning (its velocity direction is changing)
- It does not include ballistic calculations or an interception solution that accounts for projectile speed
- For a target with no Rigidbody whose position is updated in FixedUpdate, Estimated Velocity can fluctuate due to the mismatch between the render frame rate and the physics update rate (in our measurements, up to about 10% relative to the true value). Two things help: (1) attach a non-kinematic Rigidbody to the target and move it via physics (or by assigning velocity directly) - the component then reads the Rigidbody's velocity directly and this fluctuation goes away; (2) set Velocity Smooth Time higher (this trades slower reaction to real speed changes for stronger smoothing)

---

## Demo Scene

You can check the component behavior immediately using the included demo scene `Demo_LookAtSmooth` (`Assets/LogicDevSupport/Move_LookAtSmooth_004/Demo/`).

- **Left side:** Full-body rotation demo - a blue capsule on a tilting platform tracks either the yellow sphere `A` or the green sphere `A2`. You can also see that tracking continues while the platform tilts
- **Right side:** IK head tracking demo - a character rotates only its head to follow either the red sphere `B` or the purple sphere `B2`. The red sphere moves up and down as well as side to side, so you can also see the head tracking vertically
- **Automatic multi-target switching (new):** the demo starts in `Nearest` mode. The two spheres move back and forth sideways, and each time the nearer one changes - roughly every 2 to 3.5 seconds - the character switches the target it looks at
- **Switching the mode:** press the **M key** during Play to switch between `Nearest` and `Priority` (when the New Input System is active). In the demo, `A2` and `B2` have Priority 2 and `A` and `B` have Priority 1, so in `Priority` mode the characters keep looking at `A2` / `B2` even when they are farther away. To switch without the key, change **Mode** on the `Demo_SelectionMode` object in the Inspector (this also works during Play)
- **Reading the labels:** a two-line label is shown above each character. The top line shows the current mode and the name of the target being looked at (for example `Nearest: A2` or `Priority: B2`); the target's name is shown in a color close to that sphere's color. The bottom line shows the Predictive LookAt state (`Prediction: OFF` / `Prediction: ON`)
- **Predictive LookAt:** both demos automatically switch prediction OFF / ON every 4.5 seconds (starting with OFF). While prediction is ON, a small magenta marker shows the position actually being aimed at (Predicted Position) for the target currently being looked at

> Note: switching every selector's mode at once, updating the labels, and toggling prediction OFF / ON are all done by demo-only scripts (`DemoSelectionModeSwitcher`, `DemoSelectionLabel`, and `DemoPredictionToggle` in `Demo/Scripts/`). You do not need these scripts in your own scenes - set **SelectionMode** on `LogicDevLookAtTargetSelector` and **Enable Prediction** on the LookAt component directly.

> Note: The demo scene uses primitive shapes only. A demo using Starter Assets is available on the Asset Store page (not included in this package).

---

## Before You Import (Important)

Before importing this package, please delete all previous versions (Move_LookAtSmooth_001, Move_LookAtSmooth_002, and Move_LookAtSmooth_003) from your project. Importing without deleting them can cause shared scripts or scene files to be silently overwritten, so a folder's name may no longer match what it actually contains. Depending on which previous version you have, it may also result in compile errors.

If you have edited any files inside a previous version's folder (such as the demo scene), please back them up before deleting it.

---

## Upgrading from 003

Notes for users of Move_LookAtSmooth_003:

- As noted in "Before You Import" above, delete 003 from your project before importing this package
- Deleting the 003 folder before importing this package carries over your existing component settings on scenes and prefabs (RotationSpeed, angle clamps, Predictive LookAt settings, etc.) automatically (verified on Unity 2022.3 with non-default values set on the full-body component, the IK component, and a Prefab; no Missing Script occurs)
- Unless you add `LogicDevLookAtTargetSelector`, the full-body version and the IK head-tracking version on non-Humanoid rigs behave the same as in 003 (we compared the included demo's motion against 003 and confirmed that the rotations match on every frame). There is one exception: with ClampAngleY on, changing to a different target (through `SetTarget()` or by assigning the Target field directly) now releases the angle clamp side (up to 003 the rotation stayed fixed at the previous limit until the target came back inside the range; see "Angle clamp side" under "Behavior")

**The IK head-tracking version on a Humanoid behaves differently from 003 in the following three ways.**

- **It now interpolates with RotationSpeed.** Up to 003, RotationSpeed was not used on a Humanoid, and the head pointed exactly at the target every frame (when the target was switched, the head changed direction within one frame). In this version it turns smoothly according to RotationSpeed, the same way as on non-Humanoid rigs. At the default value of 5, it takes about 0.3 to 0.5 seconds after a target switch for the head to cover 90% of the turn. To make the head follow faster, raise RotationSpeed (maximum 20)
- **While Target is null and while `Pause()` is in effect, the head now holds its last direction.** Up to 003, it returned to the animated pose within one frame. To return to the animated pose, set IKWeight to 0 or disable the component
- **If you use Predictive LookAt:** up to 003 there was no interpolation lag on a Humanoid, so the head pointed ahead of the target by the amount of the lead. In this version the lead cancels out the lag caused by RotationSpeed (in our measurement with RotationSpeed=5 and Prediction Time=0.25, the average difference from the target's direction was 0.1 to 2.4°, slightly ahead of the target). If you tuned Prediction Time for a Humanoid in 003, please re-tune it

**If you are replacing the free version (Move_LookAtSmooth_001)**

- In this version the full-body component follows the target left/right (Y axis) only. The up/down rotation setting of 001 (EnableXAxis) and its up/down angle clamp (ClampAngleX, MinAngleX, MaxAngleX) are not carried over. Scenes and prefabs saved in 001 with up/down rotation turned on do not rotate up/down in this version (these fields are not shown in the Inspector either)
- If you also need the up/down direction, use the IK head-tracking component (it turns only the head, both up/down and left/right)

---

## Upgrade Path

An upgraded version (`Move_LookAtSmooth_005`) is in development. Stay tuned!

The free version (`Move_LookAtSmooth_001`) is available here:

https://assetstore.unity.com/packages/slug/389610

---

If you find this asset useful, please consider leaving a review - it really helps!

You can leave a review from the "Reviews" tab on the store page below.

https://assetstore.unity.com/packages/slug/417124
