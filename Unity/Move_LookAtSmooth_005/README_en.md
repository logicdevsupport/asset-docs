# LogicDevLookAtSmooth - Smooth Look-At Component Complete Edition

A LookAt component that smoothly tracks a target object. Comes with two variants: a full-body rotation version (`LogicDevLookAtSmooth`) and an IK head-tracking version (`LogicDevLookAtSmoothIK`). It also includes a component that automatically picks which target to look at from a list of candidates and switches between them (`LogicDevLookAtTargetSelector`). This version (Complete Edition, the final version of the series) adds a smooth fade in / out of the head look, a "most in front" selection mode, field-of-view / distance / line-of-sight limits for target selection, Scene-view gizmos, and development-only tools: a Game-view debug display, a runtime tuning panel and a way to keep values changed in Play Mode.

---

## Features

- **Two LookAt types included** - Switch between full-body Y-axis rotation and IK head tracking
- **Automatic multi-target switching** - Picks "the nearest candidate", "the highest-priority candidate (the nearest one among equal priorities)" or **"the candidate most in front" (new)** from a candidate list and switches the look-at target automatically. Includes two settings that prevent switching too often (hold time and switch margin) and an event that tells you when the target changes
- **Field of view, distance and line of sight (new)** - Candidates outside a field of view, outside a distance range, or hidden behind a collider can be ignored. All three are off by default
- **Smooth fade of the head look (new)** - `FadeOut()` / `FadeIn()` and **WeightFadeTime** blend the IK head look out and back in over a set time instead of snapping (0 = immediately, as before)
- **Scene-view gizmos (new)** - When an object is selected, the Scene view shows its angle limits, the field of view / distance limits, every candidate (selectable or not), the selected target and the predicted point
- **Debug display, tuning panel and keeping Play Mode values (new, development only)** - A Game-view display of the same information, a small panel to change the main settings while playing, and a "Record values" button that lets you keep chosen values after Play Mode ends. These exist only in the Editor and in Development Builds; a release build contains none of their code
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
| Input System | The components do not depend on any input system. Only the keys of the included demo (M, V, F) and of the debug tools (G, T) use the New Input System (if it is not installed, only those keys are disabled) |

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

### Fading the Head Look In and Out (new)

Stops (or restarts) the IK head look smoothly instead of snapping - for example when a conversation ends or a cutscene starts.

```csharp
var ik = GetComponent<LogicDevLookAtSmoothIK>();
ik.FadeOut(0.5f);   // the head returns to its animated pose over 0.5 s and stays there
ik.FadeIn(0.5f);    // the head blends back into the look over 0.5 s
ik.FadeOut();       // without an argument: uses WeightFadeTime
```

- On a Humanoid the head blends to its animated pose; on a non-Humanoid rig it blends to the pose the head bone had when the component started
- Tracking keeps running while faded out, so `FadeIn()` blends straight into the current look direction. Calling `FadeIn()` in the middle of a fade-out (or the other way round) turns back smoothly from where it is
- **WeightFadeTime** (default 0) is also used when **IKWeight** is changed on a Humanoid: the head moves to the new weight over that time. At 0 everything happens within one frame, exactly as in the previous version
- `IsFadedOut` (true after `FadeOut()` until `FadeIn()`) and `CurrentWeight` (the weight in use right now, 0 to 1) tell you the state

The fade exists on the IK head-tracking component only. The full-body component turns the whole object, so there is no "animated pose" to blend back to; use `Pause()` / `Resume()` (it keeps its direction while paused) or change the Target.

### Using Automatic Multi-Target Switching

1. Add `LogicDevLookAtTargetSelector` to the GameObject that has `LogicDevLookAtSmooth` or `LogicDevLookAtSmoothIK` (Add Component > LogicDevSupport > Move > LookAt Target Selector). If both **LookAtComponent** and **LookAtIKComponent** are left empty, the components on the same GameObject are used automatically
2. Add the Transforms that may be looked at to **Candidates**. If you plan to use priority-based selection, also set each candidate's **Priority** (0 to 100, higher wins)
3. Choose how to pick with **SelectionMode**. `Nearest` picks the closest candidate (distance is measured from the position of the GameObject that has `LogicDevLookAtTargetSelector`). `Priority` picks the candidate with the highest Priority value; if several candidates share that value, the closest one among them is picked. `Facing` (new) picks the candidate at the smallest angle from the direction the character faces (the LookAt component's reference forward - the direction its angle limits are measured from), seen from **ViewPoint**
4. If it switches too often, adjust **MinHoldTime** (how many seconds to keep a target after switching to it) and **SwitchMargin** (how much better a candidate must be before switching to it: closer in `Nearest` / `Priority`, a smaller angle in `Facing`). Setting either one to 0 turns it off
5. (new) To ignore candidates the character should not be able to see, set **ViewPoint** (for a character, the head) and turn on any of **UseViewAngle** (field of view: left/right and up/down from the reference forward), **UseDistanceLimit** (MinDistance to MaxDistance from ViewPoint) and **CheckLineOfSight** (a ray from ViewPoint to the candidate; colliders on **ObstacleLayers** block it). A candidate that fails any of them is treated like an absent one. All are off by default, so nothing changes until you turn them on
6. The selected target is assigned to the LookAt component's **Target** automatically. You can leave the LookAt component's own Target field empty (even if it is set, it is overwritten with the target chosen by `LogicDevLookAtTargetSelector`)

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

### Gizmos, Debug Display, Tuning Panel and Keeping Play Mode Values (new)

**Scene-view gizmos (Editor)**: select an object with a LookAt component or `LogicDevLookAtTargetSelector` to see its angle limits (cyan), the field of view and distance limits (white), every candidate (green = can be selected, grey = outside the view / distance limits, red = line of sight blocked), the selected target (yellow), the current look direction (orange) and the predicted point (magenta). **ShowGizmos** on each component chooses when: `Always` (default: all the time), `Selected` (only while the object is selected) or `Off`. Gizmos appear only in the Scene view, and only while the Scene view's **Gizmos** button (top right of the Scene view) is on; turn that button off to hide all gizmos at once. They are never drawn in the Game view (even with the Game view's Gizmos button on) or in a build; for the Game view use the LookAt Debug View below.

The following tools are in the `Debug` folder (`Assets/LogicDevSupport/Move_LookAtSmooth_005/Debug/`). **They exist only in the Editor and in Development Builds: in a release (non-development) build their code is compiled out completely** (the two components stay as empty placeholders so that scenes containing them load without a missing-script warning; they draw nothing). The LookAt components do not depend on them, so you can also delete the whole `Debug` folder.

**Adding and removing them**: one step each. **Tools > LogicDevSupport > LookAt > Add Debug Tools to Scene** creates one GameObject ("LookAt Debug Tools") with both components; they work for every LookAt object in the scene. **Remove Debug Tools from Scene** deletes it again (both can be undone with Ctrl+Z). Or delete the whole `Debug` folder. In a release build nothing has to be removed: the code is not included.

- **LookAt Debug View** (Add Component > LogicDevSupport > Move > LookAt Debug View): draws the same information as the gizmos in the Game view for every LookAt component and selector in the scene, with the same colors, plus a status table in the top-left corner with one row per object and fixed columns (object, mode, target, blocked, out of range, prediction, head fade; a cell that does not apply stays empty, so the table does not change size). Rows follow the order of the Hierarchy window (in a Development Build, Unity does not keep the order of the root objects, so rows under different root objects can come in another order). With many objects, **MaxRows** (default 8) limits the table and the last row says "+N more"; **OnScreenOnly** (default off) hides the objects the camera cannot see (no lines, no row). Works with the Built-in Render Pipeline, URP and HDRP. **G** shows / hides it (New Input System)
- **LookAt Tuning Panel** (Add Component > LogicDevSupport > Move > LookAt Tuning Panel): a small panel in the bottom-right corner of the Game view for the main settings of one object at a time (`<` `>` choose the object, `-` / `+` fold / open): selection mode, view angle, max distance, line of sight, prediction time, fade time, and a Fade out / Fade in button. It also switches the parts of the Debug View (Angles, Lines, Prediction, Text) and can limit it to the object chosen in the panel ("This object only") or to the objects on screen ("On screen only"). `<` `>` step through the objects in the same order as the table. It changes the same values as the Inspector, so both always show the same values. **T** shows / hides it (while it is folded or hidden, a small "T: Tuning" hint stays next to it). Drag the title bar to move it: it always stays inside the screen and opens away from the nearest corner (upward in the lower half, downward in the upper half). The position is remembered. While the panel is open, the mouse cursor is shown and unlocked so you can use the panel even if the game locks it (`FreeCursorWhileOpen`, on by default); folding or hiding the panel restores the previous cursor state
- **Keeping Play Mode values (Editor only)**: press **Record values** (on the panel, or at the bottom of the Inspector of the three components during Play) to record that component's current values. You can press it as often as you like; the last recording is kept. When Play Mode ends, a window lists only the settings that differ from the scene as "object / setting / before -> after". Tick the ones to keep and press **Apply** (one Undo step: Ctrl+Z reverts it), or **Discard**. Save the scene afterwards. Recorded: the components' numbers, toggles, choices and vectors. Not recorded: object references (Target, ViewPoint, ...) and the Candidates list

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
| ShowGizmos | enum | Always | (new) When to draw the angle limits, the facing direction and the line to the target in the Scene view: Always / Selected (while selected) / Off (Scene view only, while its Gizmos button is on; Editor only) |
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
| WeightFadeTime | float | 0 | (new) Seconds over which the head blends when `FadeIn()` / `FadeOut()` is called without a duration, and when IKWeight is changed on a Humanoid (0 to 5). 0 = within one frame, as in the previous version |
| ShowGizmos | enum | Always | (new) When to draw the angle limits, the look direction, the line to the target and the predicted point in the Scene view: Always / Selected (while selected) / Off (Scene view only, while its Gizmos button is on; Editor only) |

> Note: On a Humanoid, the head holds its last direction while `Pause()` is in effect and while Target is null (it does not return to the animated pose). To return to the animated pose smoothly, call `FadeOut()` (see "Fading the Head Look In and Out"). Setting IKWeight to 0 (with WeightFadeTime 0) or disabling the component (enabled = false) returns it within one frame. When the component is enabled again, the head turns from the character's forward direction toward the target at RotationSpeed. When it is enabled again, the head turns from the character's forward direction toward the target at RotationSpeed.

> Note: Keep the width from MinAngleY to MaxAngleY (MaxAngleY - MinAngleY) at 180° or less (Humanoid and non-Humanoid alike). The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). On a Humanoid the whole body can appear to twist sharply at that moment. When ClampAngleY is on and the width is over 180°, a Warning is shown once in the Console at runtime.

> Note: On a Humanoid, the same thing happens when ClampAngleY is turned off (there is no range at all, so the look direction can swing round past the back - for example when switching to a target on the opposite side - and the head and body can appear to twist sharply). On a Humanoid, keep ClampAngleY on with a width of 180° or less. When ClampAngleY is off on a Humanoid, a Warning is shown once in the Console at runtime.

> Note: When used on a Humanoid character, setting a very wide angle range (Min/Max close to 180°) may not be fully achievable in practice - the character's own neck/torso muscle definition limits how far it can actually turn, independent of these settings. Making a character naturally look directly behind itself using only head/torso IK (without turning the body) is a structural limitation of Unity's built-in Humanoid LookAt IK system.

### LogicDevLookAtTargetSelector (Automatic Multi-Target Switching)

| Parameter | Type | Default | Description |
|---|---|---|---|
| LookAtComponent | LogicDevLookAtSmooth | null | The full-body component that receives the selected target. If both this and LookAtIKComponent are empty, the components on the same GameObject are found automatically |
| LookAtIKComponent | LogicDevLookAtSmoothIK | null | The IK head-tracking component that receives the selected target (same rule as above). If both are set, the same target is given to both |
| Candidates | List | empty | The candidate list. Each entry has a **Target** (Transform) and a **Priority** (int, 0 to 100 in the Inspector). Empty, destroyed, or inactive candidates are never selected |
| SelectionMode | enum | Nearest | `Nearest`: the closest candidate. `Priority`: the candidate with the highest Priority (the closer one if the values are equal). `Facing` (new): the candidate at the smallest angle from the reference forward, seen from ViewPoint. Priority values are used in Priority mode only |
| MinHoldTime | float | 0.5 | After switching targets, the component does not switch again for this many seconds, even if a better candidate appears (0 to 5; 0 = off). When the current target becomes unavailable, and right after SelectionMode is changed, it picks again without waiting for this time |
| SwitchMargin | float | 0.1 | How much better a candidate must be before switching between candidates of the same Priority (0 to 0.9; 0 = off). At 0.1, it switches only when the new candidate is more than 10% closer than the current target (in Facing mode: when its angle is more than 10% smaller - for example under 18° while the current target is at 20°). In Priority mode, a candidate with a higher Priority is switched to without this margin (MinHoldTime still applies) |
| ViewPoint | Transform | null | (new) The point the view limits and the Facing angle are measured from. Empty = this GameObject. For a character, assign the head (or an eye-level child) |
| UseViewAngle | bool | false | (new) Ignore candidates outside the field of view around the reference forward (the driven LookAt component's reference forward - the direction its angle limits are measured from; this GameObject's forward if it drives none) |
| ViewAngleY | float | 60 | (new) Field of view to the left and to the right, in degrees (half width, 0 to 180) |
| ViewAngleX | float | 45 | (new) Field of view above and below, in degrees (half height, 0 to 90) |
| UseDistanceLimit | bool | false | (new) Ignore candidates closer than MinDistance or farther than MaxDistance (measured from ViewPoint) |
| MinDistance | float | 0 | (new) World units |
| MaxDistance | float | 20 | (new) World units (never smaller than MinDistance) |
| CheckLineOfSight | bool | false | (new) Ignore candidates hidden behind a collider: one ray is cast from ViewPoint to the candidate's position. Colliders of this object, of the driven LookAt objects, of ViewPoint and of the candidate itself (and their children) never block; trigger colliders are ignored |
| ObstacleLayers | LayerMask | Default raycast layers | (new) Layers whose colliders can block the line of sight |
| ShowGizmos | enum | Always | (new) When to draw the view limits, the candidates and the selected target in the Scene view: Always / Selected (while selected) / Off (Scene view only, while its Gizmos button is on; Editor only) |
| OnTargetChanged | `UnityEvent<Transform, Transform>` | - | Raised whenever the selected target changes. The arguments are (previous target, new target); either can be null. The previous target can also be an object that has already been destroyed |

**Script API**

| Member | Description |
|---|---|
| `void AddCandidate(Transform target, int priority = 0)` | Adds a candidate. If the Transform is already a candidate, only its priority is updated. Passing null logs a warning and is ignored |
| `bool RemoveCandidate(Transform target)` | Removes a candidate. Returns false if it was not in the list |
| `bool SetCandidatePriority(Transform target, int priority)` | Changes a candidate's priority. If it is not in the list, logs a warning and returns false |
| `Transform CurrentTarget { get; }` | The currently selected target (read-only). null when no candidate is available |
| `bool IsSelectable(Transform target)` | (new) True if the target could be selected right now: it exists, is active and passes the view limits that are on. It does not have to be in the candidate list. Selection uses exactly this test |
| `LookAtCandidateStatus GetCandidateStatus(Transform target, out Vector3 blockPoint)` | (new) The same test broken down: `Selectable`, `Unavailable` (null / destroyed / inactive), `OutOfDistance`, `OutOfViewAngle` or `Blocked` (blockPoint = where the ray hits the obstacle). Read-only |
| `void GetViewFrame(out Vector3 origin, out Vector3 forward, out Vector3 up)` | (new) The point and the directions the view limits and the Facing angle are measured from, right now |

`Candidates`, `SelectionMode`, `MinHoldTime`, `SwitchMargin` and the view-limit settings are public fields, so you can also change them directly from a script at runtime.

Both LookAt components also have `void ResetClampState()` (see "Angle clamp side" under "Behavior" below) and the read-only `CurrentReferenceForward` / `CurrentReferenceUp` (the direction their angle limits are measured from). The IK head-tracking component also has `FadeIn()` / `FadeOut()` (with or without a duration), `IsFadedOut`, `CurrentWeight`, and the read-only `CurrentHeadPosition` / `CurrentAimDirection` (new).

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
| (new) The current target leaves the field of view or the distance range, or becomes hidden (view limits on) | Same as above: it counts as absent, and a new target is picked without waiting for MinHoldTime. When it can be seen again it is a normal candidate (MinHoldTime and SwitchMargin apply) |
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
- For `Nearest` and `Priority`, distance is the straight-line 3D distance measured from the position (transform.position) of the GameObject that has `LogicDevLookAtTargetSelector`, as in the previous version (also when it drives the IK head-tracking version). The Facing angle and the view limits (angle, distance, line of sight) are measured from **ViewPoint** instead (this GameObject when ViewPoint is empty)

**Example of what MinHoldTime and SwitchMargin do (our measurement)**: with two candidates swaying back and forth at nearly the same distance (so that "the nearer one" changes 46 times in 10 seconds) and the full-body version selecting between them, the number of switches was 46 with both set to 0, 15 with only MinHoldTime 0.5, 25 with only SwitchMargin 0.1, and 11 with both (the default values).

---

## Known Limitations

**Automatic multi-target switching**

- The line of sight is one ray from ViewPoint to the candidate's position (its pivot). A candidate whose pivot is hidden counts as hidden even if part of it is visible, and the other way round. The candidate itself does not need a collider
- The field of view is measured as two angles (left/right and up/down separately), not as a circular cone
- Every candidate is checked at every evaluation (no frames are skipped). In our measurement in the Editor, one evaluation took about 30 µs for 100 candidates with no limits, about 90 µs with all limits on, and about 0.4 ms for 500 candidates with line of sight
- When Predictive LookAt is enabled, velocity estimation starts over from zero every time the target switches. Frequent switching leaves less time with an effective lead, so use MinHoldTime and SwitchMargin to keep the switching rate down
- When driving the IK head-tracking version on a Humanoid (Animator Update Mode: Normal), selection is evaluated early in Update (execution order -100). If your candidates are moved in Update at the default execution order, the evaluation uses their positions from the previous frame (a change in which one is nearer is detected one frame late)

**Predictive LookAt**

- Predictive LookAt is a simple linear extrapolation based on the target's **current** velocity. Error increases when the target is accelerating, decelerating, or turning (its velocity direction is changing)
- It does not include ballistic calculations or an interception solution that accounts for projectile speed
- For a target with no Rigidbody whose position is updated in FixedUpdate, Estimated Velocity can fluctuate due to the mismatch between the render frame rate and the physics update rate (in our measurements, up to about 10% relative to the true value). Two things help: (1) attach a non-kinematic Rigidbody to the target and move it via physics (or by assigning velocity directly) - the component then reads the Rigidbody's velocity directly and this fluctuation goes away; (2) set Velocity Smooth Time higher (this trades slower reaction to real speed changes for stronger smoothing)

---

## Demo Scene

You can check the component behavior immediately using the included demo scene `Demo_LookAtSmooth` (`Assets/LogicDevSupport/Move_LookAtSmooth_005/Demo/`).

- **Left side:** Full-body rotation demo - a blue capsule on a tilting platform tracks either the yellow sphere `A` or the green sphere `A2`. You can also see that tracking continues while the platform tilts
- **Right side:** IK head tracking demo - a character rotates only its head to follow either the red sphere `B` or the purple sphere `B2`. The red sphere moves up and down as well as side to side, so you can also see the head tracking vertically
- **Automatic multi-target switching (new):** the demo starts in `Nearest` mode. The two spheres move back and forth sideways, and each time the nearer one changes - roughly every 2 to 3.5 seconds - the character switches the target it looks at
- **Switching the mode:** press the **M key** during Play to cycle through `Nearest`, `Priority` and `Facing` (when the New Input System is active). In the demo, `A2` and `B2` have Priority 2 and `A` and `B` have Priority 1, so in `Priority` mode the characters keep looking at `A2` / `B2` even when they are farther away. To switch without the key, change **Mode** on the `Demo_SelectionMode` object in the Inspector (this also works during Play)
- **Reading the labels:** a two-line label is shown above each character. The top line shows the current mode and the name of the target being looked at (for example `Nearest: A2` or `Priority: B2`); the target's name is shown in a color close to that sphere's color. The bottom line shows the Predictive LookAt state (`Prediction: OFF` / `Prediction: ON`)
- **Predictive LookAt:** both demos automatically switch prediction OFF / ON every 4.5 seconds (starting with OFF). While prediction is ON, a small magenta marker shows the position actually being aimed at (Predicted Position) for the target currently being looked at
- **View limits (new):** a wall stands on the tilting platform and the full-body side checks the line of sight, so `A2` cannot be selected while it is behind the wall (the label adds `(A2 hidden)`). The IK side has a field of view of ±70° left/right and ±45° up/down measured from the head, so a sphere outside it cannot be selected (`(B2 out of view)`). Press the **V key** to turn the view limits off and on
- **Head fade (new):** the IK character fades its head look out and back in on a cycle (8 s on, 1 s fading out, 2 s off, 1 s fading in); the label above it shows `Look: ON` / `fading out` / `OFF` / `fading in`. Press the **F key** to fade out / in by hand (this stops the cycle)
- **Debug display and tuning panel (new):** the `Demo_Debug` object has the LookAt Debug View (on from the start, **G** key) and the LookAt Tuning Panel (folded in the bottom-right corner at the start, **T** key; press `+` to open it). See "Gizmos, Debug Display, Tuning Panel and Keeping Play Mode Values"

> Note: switching every selector's mode at once, updating the labels, toggling prediction OFF / ON and the fade cycle are all done by demo-only scripts (`DemoSelectionModeSwitcher`, `DemoSelectionLabel`, `DemoPredictionToggle` and `DemoFadeToggle` in `Demo/Scripts/`). You do not need these scripts in your own scenes - set **SelectionMode** on `LogicDevLookAtTargetSelector` and **Enable Prediction** on the LookAt component directly.

> Note: The demo scene uses primitive shapes only. A demo using Starter Assets is available on the Asset Store page (not included in this package).

---

## Before You Import (Important)

Before importing this package, please delete all previous versions (Move_LookAtSmooth_001, Move_LookAtSmooth_002, Move_LookAtSmooth_003, and Move_LookAtSmooth_004) from your project. Importing without deleting them can cause shared scripts or scene files to be silently overwritten, so a folder's name may no longer match what it actually contains. Depending on which previous version you have, it may also result in compile errors.

If you have edited any files inside a previous version's folder (such as the demo scene), please back them up before deleting it.

---

## Upgrading from 004

Notes for users of Move_LookAtSmooth_004:

- As noted in "Before You Import" above, delete 004 from your project before importing this package
- Deleting the 004 folder before importing this package carries over your existing component settings on scenes and prefabs (verified with the full-body and the IK head-tracking components; `LogicDevLookAtTargetSelector` also keeps the same script identity as in 004). The new settings start at their defaults: SelectionMode values saved in 004 (`Nearest`, `Priority`) keep their meaning, the view limits are off and WeightFadeTime is 0
- With the new settings left at their defaults, everything behaves the same as in 004 (we ran 004 and this version on the same 3,129 test conditions, and the recorded rotations and selections matched bit for bit on every frame)

**If you are upgrading from 003 or earlier**, also note the changes made in 004: on a Humanoid the IK head-tracking version turns with RotationSpeed (up to 003 it pointed exactly at the target every frame), holds its last direction while Target is null or while paused, and Predictive LookAt cancels out that interpolation lag (re-tune Prediction Time if you tuned it for a Humanoid in 003). With ClampAngleY on, changing to a different target releases the angle clamp side.

**If you are replacing the free version (Move_LookAtSmooth_001)**

- In this version the full-body component follows the target left/right (Y axis) only. The up/down rotation setting of 001 (EnableXAxis) and its up/down angle clamp (ClampAngleX, MinAngleX, MaxAngleX) are not carried over. Scenes and prefabs saved in 001 with up/down rotation turned on do not rotate up/down in this version (these fields are not shown in the Inspector either)
- If you also need the up/down direction, use the IK head-tracking component (it turns only the head, both up/down and left/right)

---

## Upgrade Path

This is the final and most complete version of the series.

The free version (`Move_LookAtSmooth_001`) is available here:

https://assetstore.unity.com/packages/slug/389610

---

If you find this asset useful, please consider leaving a review - it really helps!

You can leave a review from the "Reviews" tab on the store page below.

https://assetstore.unity.com/packages/slug/424450
