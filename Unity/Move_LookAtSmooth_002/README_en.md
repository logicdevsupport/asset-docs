# LogicDevLookAtSmooth - Smooth Look-At Component

A LookAt component that smoothly tracks a target object. Comes with two variants: a full-body rotation version (`LogicDevLookAtSmooth`) and an IK head-tracking version (`LogicDevLookAtSmoothIK`).

---

## Features

- **Two LookAt types included** — Switch between full-body Y-axis rotation and Humanoid IK head tracking
- **Automatically follows vehicle/drone tilt** — Just attach it: even when the object itself (or its parent) banks or tilts via physics or animation, it keeps tracking the target with the correct sense of "up" — no setup required
- **Real-time Inspector adjustment** — Tweak parameters while in Play Mode and see the result instantly
- **Automatic Humanoid Animator support** — Uses `OnAnimatorIK` for natural head movement on animated characters
- **Angle clamping** — Full-body version clamps the Y axis (left/right); the IK head-tracking version clamps both X and Y axes independently
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
3. Adjust **RotationSpeed**, **EnableYAxis**, and angle clamp settings as needed (tilt from a vehicle/drone is followed automatically, so no extra setup is usually needed)
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
| EnableYAxis | bool | true | Enable/disable Y-axis (left/right) rotation |
| ClampAngleY | bool | false | Enable/disable Y-axis angle clamping |
| MinAngleY | float | -90 | Minimum Y-axis angle (degrees). Used when ClampAngleY is on |
| MaxAngleY | float | 90 | Maximum Y-axis angle (degrees). Used when ClampAngleY is on |
| UpAxis | Vector3 | (0, 1, 0) | "Up" direction in local space. **Hidden from the Inspector** (script-only, advanced use — see the Note below) |
| ReferenceForward | Vector3 | (0, 0, 0) | World-space direction used as Yaw=0. When left at (0,0,0) (unset), automatically falls back to this object's own facing direction (transform.forward) at start. Set a non-zero value to use that direction instead. To change it at runtime from code, call SetReferenceForward() (directly assigning the field only takes effect before Awake or when edited via the Inspector) |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

> **Note:** UpAxis follows the object's own tilt automatically the moment you attach the component, so it normally needs no changes. It's intentionally hidden from the Inspector to prevent the visual "rolling" symptom that manual edits can cause. You can still set it from a script if needed, but manually setting anything other than the default `(0, 1, 0)` makes the object yaw around that axis, which can make it look like it's rotating/rolling.

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

> **Note:** When used on a Humanoid character, setting a very wide angle range (Min/Max close to 180°) may not be fully achievable in practice — the character's own neck/torso muscle definition limits how far it can actually turn, independent of these settings. Making a character naturally look directly behind itself using only head/torso IK (without turning the body) is a structural limitation of Unity's built-in Humanoid LookAt IK system. Normal usage ranges (front to side) work reliably.

---

## Demo Scene

You can check the component behavior immediately using the included demo scene `Demo_LookAtSmooth` (`Assets/LogicDevSupport/Move_LookAtSmooth_002/Demo/`).

- **Left side:** Full-body rotation demo — a blue capsule smoothly tracks a yellow sphere, mounted on a tilting platform to demonstrate automatic tracking through vehicle/drone-style banking
- **Right side:** IK head tracking demo — a character rotates only its head to follow a red sphere

> **Note:** The demo scene uses primitive shapes only. A demo using Starter Assets is available on the Asset Store page (not included in this package).

---

※ Before importing this package, please delete the previous version (Move_LookAtSmooth_001) from your project. Importing both versions at the same time will cause compile errors due to duplicate class definitions.

---

## Upgrade Path

An upgraded version (`Move_LookAtSmooth_003`) is in development, with planned features including predictive look-at (anticipating target movement based on velocity). Stay tuned!

---

If you find this asset useful, please consider leaving a review — it really helps!
