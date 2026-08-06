# LogicDevLookAtSmooth (UE) - Smooth Look-At Component

A LookAt component that smoothly tracks a target actor. Comes with two variants: a full-body rotation version (`ULogicDevLookAtSmoothComponent`) and an IK head-tracking version (`ULogicDevLookAtSmoothIKComponent`).

---

## Features

- **Two LookAt types included** — Switch between full-body rotation and IK head tracking
- **C++ and Blueprint support** — `UFUNCTION(BlueprintCallable)` lets you call everything from Blueprint too
- **Real-time Details panel adjustment** — Tweak parameters while in Play Mode and see the result instantly
- **Auto-detects three mesh setups (IK version)** — Works with `PoseableMeshComponent`, `SkeletalMeshComponent`, or a plain `SceneComponent` hierarchy
- **Axis limits and angle clamping** — Set minimum and maximum angles independently for X and Y axes

---

## Requirements

| Item | Details |
|------|---------|
| Unreal Engine Version | UE 5.3 / 5.5 / 5.7 / 5.8 |
| Implementation | C++ |
| Blueprint Support | Full `UFUNCTION(BlueprintCallable)` coverage — usable from Blueprint-only projects |
| Input System | No dependency |

---

## How to Use

### ULogicDevLookAtSmoothComponent (Full Body Rotation)

1. Add the `LogicDev LookAt Smooth` component to the Actor you want to rotate
2. Assign the target Actor in the **Target** field in the Details panel
3. Adjust **RotationSpeed**, **bEnableXAxis**, **bEnableYAxis**, and angle clamp settings as needed
4. Press Play — the whole Actor will smoothly rotate to face the target

### ULogicDevLookAtSmoothIKComponent (IK Head Tracking)

1. Add `LogicDev LookAt Smooth IK` to the character Actor whose head should track a target
2. Assign the target Actor in the **Target** field
3. Set **HeadBoneName** (default `"head"`) to the bone name to rotate (for PoseableMesh/SkeletalMesh setups) or the name of a child SceneComponent
4. Adjust **IKWeight** (0–1) to control the intensity of the head movement

> At BeginPlay the component automatically detects the owner's setup: if a `UPoseableMeshComponent` is present it gets full per-bone control, if only a `USkeletalMeshComponent` is present it applies a best-effort bone override, and otherwise it rotates a `USceneComponent` matching `HeadBoneName` (e.g. a cube-hierarchy character).

### Controlling via Code

```cpp
// C++
ULogicDevLookAtSmoothComponent* LookAt = GetComponentByClass<ULogicDevLookAtSmoothComponent>();
LookAt->SetTarget(NewTargetActor);
LookAt->Pause();
LookAt->Resume();
```

From Blueprint, get a reference to the component and call the **Set Target** / **Pause** / **Resume** nodes the same way. The IK variant (`ULogicDevLookAtSmoothIKComponent`) shares the same API.

---

## Parameter Reference

### ULogicDevLookAtSmoothComponent (Full Body Rotation)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `Target` | `AActor*` | `nullptr` | The object to look at. Stops tracking when null |
| `RotationSpeed` | `float` | `5.0` (range 0.0–20.0) | Tracking speed (Slerp coefficient) |
| `bEnableXAxis` | `bool` | `true` | Enable/disable X-axis (up/down) rotation |
| `bEnableYAxis` | `bool` | `true` | Enable/disable Y-axis (left/right) rotation |
| `bClampAngleX` | `bool` | `false` | Enable/disable X-axis angle clamping |
| `MinAngleX` | `float` | `-60.0` (range -180–0) | Minimum X-axis angle (degrees). Used when bClampAngleX is true |
| `MaxAngleX` | `float` | `60.0` (range 0–180) | Maximum X-axis angle (degrees). Used when bClampAngleX is true |
| `bClampAngleY` | `bool` | `false` | Enable/disable Y-axis angle clamping |
| `MinAngleY` | `float` | `-90.0` (range -180–0) | Minimum Y-axis angle (degrees). Used when bClampAngleY is true |
| `MaxAngleY` | `float` | `90.0` (range 0–180) | Maximum Y-axis angle (degrees). Used when bClampAngleY is true |

### ULogicDevLookAtSmoothIKComponent (IK Head Tracking)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `Target` | `AActor*` | `nullptr` | The object to look at. Stops tracking when null |
| `HeadBoneName` | `FName` | `"head"` | Bone name to rotate (PoseableMesh/SkeletalMesh) or child SceneComponent name |
| `RotationSpeed` | `float` | `12.0` (range 0.0–20.0) | Tracking speed (RInterpTo coefficient) |
| `bClampAngleX` | `bool` | `true` | Enable/disable X-axis angle clamping |
| `MinAngleX` | `float` | `-30.0` (range -180–0) | Minimum X-axis angle (degrees) |
| `MaxAngleX` | `float` | `30.0` (range 0–180) | Maximum X-axis angle (degrees) |
| `bClampAngleY` | `bool` | `true` | Enable/disable Y-axis angle clamping |
| `MinAngleY` | `float` | `-80.0` (range -180–0) | Minimum Y-axis angle (degrees) |
| `MaxAngleY` | `float` | `150.0` (range 0–180) | Maximum Y-axis angle (degrees) |
| `IKWeight` | `float` | `1.0` (range 0–1) | IK blend weight (0 = no effect, 1 = full look-at) |

---

## Demo Scene

You can check the component behavior immediately using the included demo level `L_LookAtSmooth_Test` (`Content/LogicDevLookAtSmooth/Demo/`).

- **Left side:** Full-body rotation demo — `Watcher_Body` tracks a yellow sphere
- **Right side:** IK head tracking demo — `IK_Character` rotates only its head to follow a red sphere

> **Note:** The demo scene uses primitive shapes only. A demo video using Mannequin is available on the Asset Store page (not included in this package).

---

## Upgrade Path

An upgraded version (`Move_LookAtSmooth_002`) is in development. Stay tuned!

---

If you find this asset useful, please consider leaving a review — it really helps!
