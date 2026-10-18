# LogicDevLookAtSmooth (UE) - Smooth Look-At Component Plus

A LookAt component that smoothly tracks a target actor. Comes with two variants: a full-body rotation version (`ULogicDevLookAtSmoothComponent`) and an IK head-tracking version (`ULogicDevLookAtSmoothIKComponent`). This version (Plus) adds Predictive LookAt: it automatically estimates the target's velocity and aims at its predicted future position.

---

## Features

- **Two LookAt types included** — Switch between full-body rotation and IK head tracking
- **Predictive LookAt support (new)** — Automatically estimates the target's velocity and aims ahead of its current position, reducing the visible tracking lag on a moving target. Off by default; when off, the component aims at the target's current location exactly as in the previous version (002)
- **C++ and Blueprint support** — `UFUNCTION(BlueprintCallable)` lets you call everything from Blueprint too
- **Real-time Details panel adjustment** — Tweak parameters while in Play Mode and see the result instantly
- **Supported setups (IK version)** — Works with a `PoseableMeshComponent` (rotates the named bone), a `SkeletalMeshComponent` (add one node to the Animation Blueprint — see "How to Use"), or a plain `SceneComponent` hierarchy (rotates the named child component)
- **Axis limits and angle clamping** — Set minimum and maximum angles independently for X and Y axes
- **UpAxis support (full-body version)** — Automatically tracks the "up" direction of a tilting object such as a vehicle or drone (see Parameter Reference for details)

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
3. Adjust **RotationSpeed**, **bEnableYAxis**, and angle clamp settings as needed
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

### Using Predictive LookAt (new)

1. Check **Enable Prediction** in the Details panel (under **LogicDev | LookAt | Prediction**). It is off by default; leaving it off keeps the original behavior
2. Adjust **Prediction Time** (how far ahead to lead, in seconds), **Velocity Smooth Time** (the time constant for smoothing the estimated velocity), and **Max Prediction Distance** (the cap on the lead offset, in cm) as needed. See the Parameter Reference below for details
3. The target's velocity is estimated automatically. If one of the Target Actor's components is simulating physics, its physics velocity (`GetPhysicsLinearVelocity()`) is used; otherwise, velocity is estimated from the change in the Actor's location over time. There is no setting to choose between the two — it is detected automatically
4. The internal velocity estimate resets automatically when you switch targets (whether by calling `SetTarget()` or by assigning the Target property directly), when the Target becomes null (even if the same Actor is assigned again afterwards), when you call `Resume()`, and when Enable Prediction is turned back on. The previous estimate never carries over
5. The effect of prediction is easiest to see with a fast-moving target

### Controlling via Code

```cpp
// C++
ULogicDevLookAtSmoothComponent* LookAt = GetComponentByClass<ULogicDevLookAtSmoothComponent>();
LookAt->SetTarget(NewTargetActor);
LookAt->Pause();
LookAt->Resume();

// Predictive LookAt (read-only state, for verification/debugging)
LookAt->bEnablePrediction = true;
const FVector Velocity = LookAt->EstimatedVelocity;   // cm/s
const FVector AimPoint = LookAt->PredictedPosition;   // the location actually being aimed at
```

From Blueprint, get a reference to the component and call the **Set Target** / **Pause** / **Resume** nodes the same way; **Enable Prediction**, **Estimated Velocity** and **Predicted Position** are available as Blueprint properties. The IK variant (`ULogicDevLookAtSmoothIKComponent`) shares the same API.

---

## Parameter Reference

### ULogicDevLookAtSmoothComponent (Full Body Rotation)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `Target` | `AActor*` | `nullptr` | The object to look at. Stops tracking when null |
| `RotationSpeed` | `float` | `5.0` (range 0.0–20.0) | Tracking speed (Slerp coefficient) |
| `bEnableYAxis` | `bool` | `true` | Enable/disable Y-axis (left/right) rotation |
| `bClampAngleY` | `bool` | `false` | Enable/disable Y-axis angle clamping |
| `MinAngleY` | `float` | `-90.0` (range -180–0) | Minimum Y-axis angle (degrees). Used when bClampAngleY is true |
| `MaxAngleY` | `float` | `90.0` (range 0–180) | Maximum Y-axis angle (degrees). Used when bClampAngleY is true |
| `ReferenceForward` | `FVector` | `(0,0,0)` | World-space direction used as Yaw=0 for angle clamping. Leave at (0,0,0) to auto-use the Actor's forward vector at BeginPlay. Changing this at runtime after BeginPlay requires calling SetReferenceForward() |
| `UpAxis` | `FVector` | `(0,0,1)` | "Up" direction in local space. **Hidden from the Details panel** (script/Blueprint-only, advanced use — see the Note below) |

> **Note (the full-body component follows left/right only):** The full-body component follows the target left/right (yaw) only. It does not rotate up/down. `bEnableXAxis` (up/down rotation) and the up/down angle clamp (`bClampAngleX`, `MinAngleX`, `MaxAngleX`) of earlier versions are no longer used and are no longer shown in the Details panel (they can still be read and written from Blueprint, but the values are ignored). The up/down rotation setting of the free version (Move_LookAtSmooth_001) is not carried over either. If you also need the up/down direction, use the IK head-tracking component (it turns only the head, both up/down and left/right).

> **Note (keep the left/right clamp range at 180° or less):** Keep the width from `MinAngleY` to `MaxAngleY` (`MaxAngleY` - `MinAngleY`) at 180° or less. The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). When `bClampAngleY` is on and the width is over 180°, a Warning is written once to the Output Log at runtime.

> **Note (clamp side and switching targets):** With `bClampAngleY` on, while the target is outside the angle limits the rotation stays fixed at the limit on the side where the target first left the range. When the target changes to a different one - through `SetTarget()` or by writing the Target property directly - this is released, and the limit nearest to the new target is chosen again (setting the same target again does not release it; setting it to null first and then assigning a target does, even if it is the same one).

> **Note (Actors whose rotation is driven by something else):** The full-body component sets the Actor's rotation directly every frame. An Actor that is simulating physics, or whose rotation is driven by Character Movement (Orient Rotation to Movement, Use Controller Rotation Yaw, etc.), would fight over the rotation and is out of scope. On such an Actor, turn off the other system's rotation control or use the IK head-tracking component instead.

> **Note (UpAxis):** `UpAxis` is re-resolved to world space every frame via `Owner->GetActorRotation()`, so it automatically follows a tilting object such as a vehicle body or a banking drone — normally you won't need to change it. It is hidden from the Details panel (`BlueprintReadWrite` only, no `EditAnywhere`). You can still set it from Blueprint/C++.

> **Note (ReferenceForward vs. the Unity version):** Unity uses a Y-up axis convention (the horizontal plane is XZ), so the Unity version's demo sets this value explicitly to a horizontal vector such as `(0,0,-1)`. Unreal Engine uses a Z-up convention (the horizontal plane is XY), so a Unity-style value like `(0,0,-1)` would point straight down here and would not be a valid horizontal direction — it gets projected away as degenerate. For that reason, this UE demo instead leaves `ReferenceForward` at its default `(0,0,0)` (auto) and relies on the Actor's own spawn rotation to already face the correct horizontal direction. If you rotate your Actor to a different starting facing, set `ReferenceForward` explicitly to the world-space horizontal direction you want as Yaw=0.

### ULogicDevLookAtSmoothIKComponent (IK Head Tracking)

> The IK head-tracking version does not support UpAxis (full-body version only).

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

### Predictive LookAt Parameters (new, shared by both components)

Both the full-body version and the IK head-tracking version have the same parameters below (Details panel category **LogicDev | LookAt | Prediction**).

| Parameter | Type | Default | Description |
|---|---|---|---|
| `bEnablePrediction` (Enable Prediction) | `bool` | `false` | Enables/disables predictive lead. When false, the component aims at `Target->GetActorLocation()` exactly as in the previous version |
| `PredictionTime` | `float` | `0.25` (min 0) | How far ahead to lead, in seconds. The actual aim point is "current target location + estimated velocity × PredictionTime". PredictionTime exists to cancel out the tracking lag caused by `RotationSpeed`, so **if you raise RotationSpeed (faster tracking), lower PredictionTime; if you lower RotationSpeed (slower tracking), raise PredictionTime**. Tune it while watching `PredictedPosition` (below). **The IK head-tracking component's default `RotationSpeed` is 12, which is fast, so use a small PredictionTime there (0.08 to 0.1 is a good starting point)**. Left at the default 0.25, the head looks ahead of the target (measured with a target swinging left and right: within about 1° of the target at 0.08 to 0.1, 5° to 10° ahead at 0.25) |
| `VelocitySmoothTime` | `float` | `0.1` (min 0) | Time constant, in seconds, for smoothing the estimated velocity. Larger values are more resistant to noise but react more slowly to real speed changes. 0 disables smoothing |
| `MaxPredictionDistance` | `float` | `1000.0` (cm) | Caps the lead offset (estimated velocity × PredictionTime), in cm. **This value has a second role**: if the target's location jumps further than this distance in a single frame, it is treated as a teleport, and velocity estimation resets once (re-converging from zero). **Setting this value too small can cause a target that is genuinely moving fast to be misidentified as having teleported.** 0 or less disables both the offset cap and teleport detection |
| `EstimatedVelocity` | `FVector` (read-only) | – | The currently estimated velocity of the target (cm/s). Exposed to Blueprint/C++ for verification/debugging |
| `PredictedPosition` | `FVector` (read-only) | – | The location actually being aimed at (the predicted location when prediction is on, otherwise the target's current location; `(0,0,0)` while Target is null). Exposed to Blueprint/C++ for verification/debugging |

---

## Known Limitations (Predictive LookAt)

- Predictive LookAt is a simple linear extrapolation based on the target's **current** velocity. Error increases when the target is accelerating, decelerating, or turning (its velocity direction is changing)
- It does not include ballistic calculations or an interception solution that accounts for projectile speed
- The physics-velocity check happens when a target is assigned. If the target only starts simulating physics after it has been assigned, its velocity keeps being estimated from its change in location until the target is changed (the result is still a valid velocity estimate)
- If the Target Actor has more than one component simulating physics, the first one found is used for the velocity

---

## Demo Scene

You can check the component behavior immediately using the included demo level `L_LookAtSmooth_Test` (`Content/LogicDevLookAtSmooth/Demo/`).

- **Left side:** Full-body rotation demo — `Watcher_Body` tracks a yellow sphere, mounted on a tilting platform to demonstrate automatic tracking through vehicle/drone-style banking
- **Right side:** IK head tracking demo — `IK_Character` rotates only its head to follow a red sphere. The red sphere moves up and down as well as side to side, so you can also see the head tracking vertically
- **Predictive LookAt (new):** both demos automatically switch prediction OFF / ON every 4.5 seconds (starting with OFF). A label above each character shows the current state (`Prediction: OFF` / `Prediction: ON`), and while prediction is ON, a small marker shows the location actually being aimed at (`PredictedPosition`). Comparing OFF and ON on the same character and the same target makes the difference in tracking lag easy to see. To make the marker easy to see, the demo characters use `PredictionTime = 0.8` (the component default is 0.25)

> **Note:** The automatic OFF / ON switching in the demo is done by a demo-only component, `LogicDevDemoPredictionToggleComponent`. You do not need it to use Predictive LookAt in your own levels — just set the component's **Enable Prediction** directly.

> **Note:** The demo scene uses primitive shapes only. A demo using Mannequin is available on the Fab product page (not included in this package).

---

## Before You Import (Important)

This package uses the same plugin name (`LogicDevLookAtSmooth`), class names and namespace as the previous versions (`Move_LookAtSmooth_001` and `Move_LookAtSmooth_002`). **If you already have `Move_LookAtSmooth_001` or `Move_LookAtSmooth_002` in your project, please delete it before adding this package.** Having more than one version installed at the same time causes plugin conflicts and compile errors due to duplicate class definitions.

If you have edited any files inside a previous version's plugin folder (such as the demo level), please back them up before deleting it.

---

## Upgrading from 002

Notes for users of `Move_LookAtSmooth_002`:

- Predictive LookAt is off by default, so unless you explicitly enable it, the component aims at the target's current location exactly as in 002
  (We recorded all 269 frames of the included demo at a fixed 60 fps and compared them with 002: the IK head-tracking version's head rotation matched bit-for-bit in every frame. The full-body version also matched bit-for-bit on UE 5.5/5.7; on 5.3/5.8 the only differences were floating-point rounding of at most about 3×10⁻⁷°)
- The component class names and the names of all existing properties (RotationSpeed, bClampAngleY, etc.) are unchanged from 002. The new Predictive LookAt properties start at the defaults listed above
- As noted above, please delete 002 itself from your project before adding 003

---

## Upgrade Path

This package (`Move_LookAtSmooth_003`) is the Plus upgrade of `Move_LookAtSmooth_001` (the free version) and `Move_LookAtSmooth_002` (Standard, UpAxis support), adding Predictive LookAt.

Want to try the free version first? https://www.fab.com/listings/b16ba60d-cb73-48ab-9e13-b23cd5e8bd80

A further upgrade (`Move_LookAtSmooth_004`) is in development. Stay tuned!

---

If you find this asset useful, please consider leaving a review on this product's Fab page — it really helps!
Leave a review here: https://www.fab.com/listings/28758cd4-7f9b-406c-8bba-2297a53039a6
