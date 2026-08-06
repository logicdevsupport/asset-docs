# LogicDevLookAtSmooth - Smooth Look-At Component

A LookAt component that smoothly tracks a target object. Comes with two variants: a full-body rotation version (`LogicDevLookAtSmooth`) and an IK head-tracking version (`LogicDevLookAtSmoothIK`).

---

## Features

- **Two LookAt types included** — Switch between full-body Y-axis rotation and Humanoid IK head tracking
- **Real-time Inspector adjustment** — Tweak parameters while in Play Mode and see the result instantly
- **Automatic Humanoid Animator support** — Uses `OnAnimatorIK` for natural head movement on animated characters
- **Axis limits and angle clamping** — Set minimum and maximum angles independently for X and Y axes
- **Flexible update timing** — Choose between `Update`, `LateUpdate`, and `FixedUpdate`

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
3. Adjust **RotationSpeed**, **EnableXAxis**, **EnableYAxis**, and angle clamp settings as needed
4. Press Play — the object will smoothly follow the target

### LogicDevLookAtSmoothIK (IK Head Tracking)

1. Add `LogicDevLookAtSmoothIK` to the root GameObject of a character with a Humanoid Animator
2. Assign the target Transform in the **Target** field
3. **For non-Humanoid rigs**, assign the head bone Transform to the **HeadBone** field
4. Adjust **IKWeight** (0–1) to control the intensity of the head movement

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
| Target | Transform | null | The object to look at. Stops tracking when null |
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
| IKWeight | float | 1.0 | IK weight for Humanoid Animator (0–1) |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

---

## Demo Scene

You can check the component behavior immediately using the included demo scene `Demo_LookAtSmooth` (`Assets/LogicDevSupport/Move_LookAtSmooth_001/Demo/`).

- **Left side:** Full-body rotation demo — a blue capsule smoothly tracks a yellow sphere
- **Right side:** IK head tracking demo — a character rotates only its head to follow a red sphere

> **Note:** The demo scene uses primitive shapes only. A demo video using Starter Assets is available on the Asset Store page (not included in this package).

---

## Upgrade Path

An upgraded version (`Move_LookAtSmooth_002`) is in development, with planned features including UpAxis support, easing curves, and multi-target management. Stay tuned!

---

If you find this asset useful, please consider leaving a review — it really helps!
