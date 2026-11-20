# LogicDevLookAtSmooth - Smooth Look-At Component

A LookAt component that smoothly tracks a target object. Comes with two variants: a full-body rotation version (`LogicDevLookAtSmooth`) and an IK head-tracking version (`LogicDevLookAtSmoothIK`).

---

## Features

- **Two LookAt types included** — Switch between full-body Y-axis rotation and Humanoid IK head tracking
- **Real-time Inspector adjustment** — Tweak parameters while in Play Mode and see the result instantly
- **Detects a Humanoid Animator automatically** — On a Humanoid, the head is turned through `OnAnimatorIK` (the Animator Controller's IK Pass must be enabled - see the setup steps below). On a non-Humanoid rig, the bone you assign is rotated directly
- **Axis limits and angle clamping** — Set minimum and maximum angles independently for X and Y axes
- **Flexible update timing** — Choose between `Update`, `LateUpdate`, and `FixedUpdate`

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
3. Adjust **RotationSpeed**, **EnableXAxis**, **EnableYAxis**, and angle clamp settings as needed
4. Press Play — the object will smoothly follow the target

### LogicDevLookAtSmoothIK (IK Head Tracking)

**Humanoid rigs**

1. Add `LogicDevLookAtSmoothIK` to the GameObject that has the Humanoid Animator (usually the character's root). **It must be on the same GameObject as the Animator.** Unity only calls `OnAnimatorIK` on scripts attached to the Animator's own GameObject, so the head does not move if the component is on a parent or child GameObject (in that case a warning is logged to the Console once at startup)
2. **Enable "IK Pass" on the Animator Controller layer** (Animator window > Layers > gear icon of the layer > check IK Pass). Unity only calls `OnAnimatorIK` for layers that have IK Pass enabled, so the head does not move while it is off (this component does not inspect the IK Pass setting, so no warning is shown)
3. Assign the target Transform in the **Target** field
4. Adjust **RotationSpeed** (how fast the head follows) and **IKWeight** (intensity of the head movement, 0 to 1) as needed

> **Note:** If the Animator's Culling Mode is anything other than "Always Animate", Unity does not run the Animator's IK step while the character is off screen, so the head direction is not updated during that time.

**Non-Humanoid rigs**

1. Add `LogicDevLookAtSmoothIK` to the character's GameObject
2. Assign the bone to rotate (the head Transform) to the **HeadBone** field
3. Assign the target Transform in the **Target** field

### Controlling via Code

```csharp
// Change the target at runtime
GetComponent<LogicDevLookAtSmooth>().SetTarget(newTarget);

// Pause tracking
GetComponent<LogicDevLookAtSmooth>().Pause();

// Resume tracking
GetComponent<LogicDevLookAtSmooth>().Resume();
```

The same API works with the IK variant.

---

## Parameter Reference

### LogicDevLookAtSmooth (Full Body Rotation)

| Parameter | Type | Default | Description |
|---|---|---|---|
| Target | Transform | null | The object to look at. When null, rotation stops and the object keeps the direction it had at that moment |
| RotationSpeed | float | 5.0 | Tracking speed (Slerp coefficient) |
| EnableXAxis | bool | true | Enable/disable X-axis (up/down) rotation |
| EnableYAxis | bool | true | Enable/disable Y-axis (left/right) rotation |
| ClampAngleX | bool | false | Enable/disable X-axis angle clamping |
| MinAngleX | float | -60 | Minimum X-axis angle (degrees). Used when ClampAngleX is on |
| MaxAngleX | float | 60 | Maximum X-axis angle (degrees). Used when ClampAngleX is on |
| ClampAngleY | bool | false | Enable/disable Y-axis angle clamping |
| MinAngleY | float | -90 | Minimum Y-axis angle (degrees). Used when ClampAngleY is on |
| MaxAngleY | float | 90 | Maximum Y-axis angle (degrees). Used when ClampAngleY is on |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

> **Note:** Keep the width from MinAngleY to MaxAngleY (MaxAngleY - MinAngleY) at 180° or less. The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). When ClampAngleY is on and the width is over 180°, a Warning is shown once in the Console at runtime.

> **Note:** The full-body component sets the Transform rotation directly every frame. An object whose rotation is also driven by a (non-kinematic) Rigidbody, a CharacterController or another script would fight over the rotation and is out of scope. On such an object, stop the other system from driving the rotation, put the full-body component on a visual child object, or use the IK head-tracking component instead.

> **Note:** With ClampAngleY on, while the target is outside the angle limits the rotation stays fixed at the limit on the side where the target first left the range. When the target changes to a different one - through `SetTarget()` or by assigning the Target field directly - this is released, and the limit nearest to the new target is chosen again (setting the same target again does not release it; setting it to null first and then assigning a target does, even if it is the same one).

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

> **Note:** Keep the width from MinAngleY to MaxAngleY (MaxAngleY - MinAngleY) at 180° or less (Humanoid and non-Humanoid alike). The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). On a Humanoid the whole body can appear to twist sharply at that moment. When ClampAngleY is on and the width is over 180°, a Warning is shown once in the Console at runtime.

> **Note:** On a Humanoid, the same thing happens when ClampAngleY is turned off (there is no range at all, so the look direction can swing round past the back - for example when switching to a target on the opposite side - and the head and body can appear to twist sharply). On a Humanoid, keep ClampAngleY on with a width of 180° or less. When ClampAngleY is off on a Humanoid, a Warning is shown once in the Console at runtime.

> **Note:** With ClampAngleY on, while the target is outside the angle limits the rotation stays fixed at the limit on the side where the target first left the range. When the target changes to a different one - through `SetTarget()` or by assigning the Target field directly - this is released, and the limit nearest to the new target is chosen again (setting the same target again does not release it; setting it to null first and then assigning a target does, even if it is the same one).

> **Note:** On a Humanoid, the head holds its last direction while `Pause()` is in effect and while Target is null (it does not return to the animated pose). To return to the animated pose, set IKWeight to 0 or disable the component (enabled = false). Either way, the head returns to the animated pose within one frame (to return smoothly, change IKWeight gradually from a script). When it is enabled again, the head turns from the character's forward direction toward the target at RotationSpeed.

---

## Demo Scene

You can check the component behavior immediately using the included demo scene `Demo_LookAtSmooth` (`Assets/LogicDevSupport/Move_LookAtSmooth_001/Demo/`).

- **Left side:** Full-body rotation demo — a blue capsule smoothly tracks a yellow sphere
- **Right side:** IK head tracking demo — a character rotates only its head to follow a red sphere

> **Note:** The demo scene uses primitive shapes only. A demo using Starter Assets is available on the Asset Store page (not included in this package).

---

## Upgrade Path

Paid upgrades are now available on the Unity Asset Store:

- Smooth LookAt Component Plus (adds tilting-object support and Predictive LookAt): https://assetstore.unity.com/packages/slug/404560
- Smooth LookAt Component Complete Edition (the final version: everything in Plus, plus automatic multi-target selection, view limits, head fade, gizmos and development tools): https://assetstore.unity.com/packages/slug/410220

Videos: https://www.youtube.com/@LogicDevSupport

※ If you upgrade, please delete this free version (`Move_LookAtSmooth_001`) from your project before importing the upgrade. Importing both at the same time will cause compile errors due to duplicate class definitions.

---

If you find this asset useful, please consider leaving a review — it really helps!

You can leave a review from the "Reviews" tab on the store page below.

https://assetstore.unity.com/packages/tools/utilities/smooth-lookat-component-389610
