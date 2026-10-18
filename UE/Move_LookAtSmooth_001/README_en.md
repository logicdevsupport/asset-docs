# LogicDevLookAtSmooth (UE) - Smooth Look-At Component

A LookAt component that smoothly tracks a target actor. Comes with two variants: a full-body rotation version (`ULogicDevLookAtSmoothComponent`) and an IK head-tracking version (`ULogicDevLookAtSmoothIKComponent`).

---

## Features

- **Two LookAt types included** — Switch between full-body rotation and IK head tracking
- **C++ and Blueprint support** — `UFUNCTION(BlueprintCallable)` lets you call everything from Blueprint too
- **Real-time Details panel adjustment** — Tweak parameters while in Play Mode and see the result instantly
- **Supported setups (IK version)** — Works with a `PoseableMeshComponent` (rotates the named bone), a `SkeletalMeshComponent` (add one node to the Animation Blueprint — see "How to Use"), or a plain `SceneComponent` hierarchy (rotates the named child component)
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
3. Set **HeadBoneName** (default `"head"`) to the bone name to rotate (for a PoseableMesh or SkeletalMesh setup) or the name of a child SceneComponent
4. Adjust **IKWeight** (0–1) to control the intensity of the head movement

> At BeginPlay the component automatically detects the owner's setup: if a `UPoseableMeshComponent` is present it rotates the bone named `HeadBoneName` directly. If there is no `UPoseableMeshComponent` but a `USkeletalMeshComponent`, it computes the rotation to add to the head bone and publishes it; the character's Animation Blueprint applies it to the bone (see "Using it with a SkeletalMeshComponent" below). If the owner has neither, it rotates the `USceneComponent` whose name matches `HeadBoneName` (e.g. a cube-hierarchy rig).

### Using it with a SkeletalMeshComponent (characters driven by an Animation Blueprint)

In UE the Animation Blueprint decides the pose of every bone each frame, so a component cannot write the head bone directly. For a SkeletalMesh setup the IK component therefore computes "the rotation to add to the head bone" (`HeadLookAtDeltaRotation`), and the character's Animation Blueprint adds it to the head bone. What you add to the Animation Blueprint is one variable, three Event Graph nodes and one AnimGraph node.

1. Add the `LogicDev LookAt Smooth IK` component to the character Actor (the Actor that owns the SkeletalMeshComponent, e.g. a Character Blueprint), and set **Target** and **HeadBoneName** (the head bone; `head` on the Mannequin)
2. Open the Animation Blueprint that character uses and add one variable of type Rotator (e.g. `HeadLookAtDelta`)
3. In the Event Graph, connect the execution pin of **Event Blueprint Update Animation** to **Set HeadLookAtDelta**. For the value, connect **Get Owning Actor** → **Get Head Look At Delta Rotation From Actor** (category LogicDev | LookAtIK) and plug its return value into the Set node
4. In the AnimGraph, insert a **Transform (Modify) Bone** node right before Output Pose (when you connect it to a local-space pose, the Local To Component / Component To Local conversions are inserted automatically). Select the node and set it up in the Details panel as in the table below
5. Compile and save the Animation Blueprint

| Transform (Modify) Bone setting | Value |
|---|---|
| Bone to Modify | The head bone (same as HeadBoneName; `head` on the Mannequin) |
| Rotation: Rotation Mode | **Add to Existing** |
| Rotation: Rotation Space | **World Space** |
| Rotation pin | Connect a Get of the variable from step 2 (`HeadLookAtDelta`) |
| Translation Mode / Scale Mode | Ignore (default) |
| Alpha | 1.0 (default) |

> **Note:** The published rotation already includes the angle clamps, the RotationSpeed smoothing and IKWeight. You do not need to process the value in the Animation Blueprint. While there is no Target and while `Pause()` is in effect it keeps its last value (the head holds its direction); with IKWeight at 0 it is no rotation (the animated pose).

> **Note:** The Animation Blueprint reads the component's value on its next animation update, so the head reacts 1 to 2 frames later than in a PoseableMesh setup.

> **Note:** The body's forward direction (the 0° reference for left/right) is derived from the positions of the left and right clavicle bones (`clavicle_l` / `clavicle_r`; `upperarm_l` / `upperarm_r` if those do not exist). On a skeleton without bones of these names, the mesh component's own facing is used as forward.

> **Note:** To read the value from C++ / Blueprint without an Animation Blueprint, use the component's `HeadLookAtDeltaRotation` (read-only) or `GetHeadLookAtDeltaRotation()`.

> **Note:** The Target field in the Details panel can only be assigned to actors already placed in the level. For Blueprint characters or actors not yet placed in the level, use SetTarget() at runtime instead (e.g. in BeginPlay).

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
| `ReferenceForward` | `FVector` | `(0,0,0)` | World-space direction used as Yaw=0 for angle clamping. Leave at (0,0,0) to auto-use the Actor's forward vector at BeginPlay. Changing this at runtime after BeginPlay requires calling SetReferenceForward() |

> **Note (keep the left/right clamp range at 180° or less):** Keep the width from `MinAngleY` to `MaxAngleY` (`MaxAngleY` - `MinAngleY`) at 180° or less. The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). When `bClampAngleY` is on and the width is over 180°, a Warning is written once to the Output Log at runtime.

> **Note (clamp side and switching targets):** With `bClampAngleY` on, while the target is outside the angle limits the rotation stays fixed at the limit on the side where the target first left the range. When the target changes to a different one - through `SetTarget()` or by writing the Target property directly - this is released, and the limit nearest to the new target is chosen again (setting the same target again does not release it; setting it to null first and then assigning a target does, even if it is the same one).

> **Note (Actors whose rotation is driven by something else):** The full-body component sets the Actor's rotation directly every frame. An Actor that is simulating physics, or whose rotation is driven by Character Movement (Orient Rotation to Movement, Use Controller Rotation Yaw, etc.), would fight over the rotation and is out of scope. On such an Actor, turn off the other system's rotation control or use the IK head-tracking component instead.

> **Note (vs. the Unity version):** Unity uses a Y-up axis convention (the horizontal plane is XZ), so the Unity version's demo sets this value explicitly to a horizontal vector such as `(0,0,-1)`. Unreal Engine uses a Z-up convention (the horizontal plane is XY), so a Unity-style value like `(0,0,-1)` would point straight down here and would not be a valid horizontal direction -- it gets projected away as degenerate. For that reason, this UE demo instead leaves `ReferenceForward` at its default `(0,0,0)` (auto) and relies on the Actor's own spawn rotation to already face the correct horizontal direction. If you rotate your Actor to a different starting facing, set `ReferenceForward` explicitly to the world-space horizontal direction you want as Yaw=0.

### ULogicDevLookAtSmoothIKComponent (IK Head Tracking)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `Target` | `AActor*` | `nullptr` | The object to look at. Stops tracking when null |
| `HeadBoneName` | `FName` | `"head"` | Bone name to rotate (PoseableMesh, SkeletalMesh) or child SceneComponent name |
| `RotationSpeed` | `float` | `12.0` (range 0.0–20.0) | Tracking speed; higher values follow faster (PoseableMesh and SkeletalMesh setups use an exponential Slerp, the SceneComponent setup uses the `RInterpTo` coefficient). Intentionally higher than the Unity version (5.0), because the interpolation method differs; this value gives a similarly natural feel |
| `bClampAngleX` | `bool` | `true` | Enable/disable X-axis angle clamping |
| `MinAngleX` | `float` | `-30.0` (range -180–0) | Minimum X-axis angle (degrees) |
| `MaxAngleX` | `float` | `30.0` (range 0–180) | Maximum X-axis angle (degrees) |
| `bClampAngleY` | `bool` | `true` | Enable/disable Y-axis angle clamping |
| `MinAngleY` | `float` | `-80.0` (range -180–0) | Minimum Y-axis angle (degrees) |
| `MaxAngleY` | `float` | `80.0` (range 0–180) | Maximum Y-axis angle (degrees) |
| `IKWeight` | `float` | `1.0` (range 0–1) | IK blend weight (0 = no effect, 1 = full look-at) |
| `HeadLookAtDeltaRotation` | `FRotator` (read-only) | – | Output used by the SkeletalMesh setup only: the world-space rotation to add to the head bone (see "Using it with a SkeletalMeshComponent" above) |

> **Note (keep the left/right clamp range at 180° or less):** Keep the width from `MinAngleY` to `MaxAngleY` (`MaxAngleY` - `MinAngleY`) at 180° or less. The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). When `bClampAngleY` is on and the width is over 180°, a Warning is written once to the Output Log at runtime.

> **Note (clamp side and switching targets):** With `bClampAngleY` on, while the target is outside the angle limits the rotation stays fixed at the limit on the side where the target first left the range. When the target changes to a different one - through `SetTarget()` or by writing the Target property directly - this is released, and the limit nearest to the new target is chosen again (setting the same target again does not release it; setting it to null first and then assigning a target does, even if it is the same one).

> **Note (defaults that differ from the Unity version):** `RotationSpeed` 12.0 (5.0 in the Unity version) is an intended difference that comes from the different interpolation method. The default left/right clamp of ±80 (±60 in the Unity version) is kept as it is so that the behavior of the published versions does not change. For the same range as the Unity version, set `MinAngleY` / `MaxAngleY` to ±60.

---

## Demo Scene

You can check the component behavior immediately using the included demo level `L_LookAtSmooth_Test` (`Content/LogicDevLookAtSmooth/Demo/`).

- **Left side:** Full-body rotation demo — `Watcher_Body` tracks a yellow sphere
- **Right side:** IK head tracking demo — `IK_Character` rotates only its head to follow a red sphere

> **Note:** The demo scene uses primitive shapes only. A demo using Mannequin is available on the Asset Store page (not included in this package).

---

## Upgrade Path

Paid upgrades are now available on Fab:

- `Move_LookAtSmooth_002` (Standard): https://www.fab.com/listings/9f156df3-0e0a-4faf-a1c1-57476c214b86
- `Move_LookAtSmooth_003` (Plus): https://www.fab.com/listings/28758cd4-7f9b-406c-8bba-2297a53039a6

---

If you find this asset useful, please consider leaving a review — it really helps!

You can leave a review on the product page below.

https://www.fab.com/listings/b16ba60d-cb73-48ab-9e13-b23cd5e8bd80
