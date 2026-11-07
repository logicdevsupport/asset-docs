# LogicDevLookAtSmooth (UE) - Smooth Look-At Component Complete Edition

A LookAt component that smoothly tracks a target actor. Comes with two variants: a full-body rotation version (`ULogicDevLookAtSmoothComponent`) and an IK head-tracking version (`ULogicDevLookAtSmoothIKComponent`). It also includes a component that automatically picks which target to look at from a list of candidates and switches between them (`ULogicDevLookAtTargetSelectorComponent`). This version (Complete Edition, the last of the series) adds smooth fading of the head look, a mode that picks the candidate most in front, field of view / distance / line of sight for the selection, a display in the editor viewport, and a debug display in the game view for development.

This is the final version of the Smooth LookAt Component series. No higher version will be released; future updates to this package will be bug fixes and compatibility updates only.

---

## Features

- **Two LookAt types included** — Switch between full-body rotation and IK head tracking
- **Automatic multi-target switching** — Picks "the nearest candidate", "the highest-priority candidate (the nearest one among equal priorities)" or **"the candidate most in front" (new)** from a candidate list and switches the look-at target automatically. Includes two settings that prevent switching too often (hold time and switch margin) and an event (Blueprint-assignable) that tells you when the target changes. Works with both the full-body and the IK head-tracking component
- **Field of view, distance and line of sight (new)** — Candidates outside a field of view, outside a distance range, or hidden behind something can be ignored. All three are off by default
- **Smooth fade of the head look (new)** — `FadeOut()` / `FadeIn()` and **WeightFadeTime** blend the IK head look out and back in over a set time instead of snapping (0 = immediately, as before)
- **Editor viewport display (new)** — The editor viewport shows the angle limits, the field of view / distance limits, every candidate (selectable or not) and the selected target
- **Game-view debug display (new, development only)** — The same information plus a status table per object, in the game view during Play. It uses DrawDebug drawing, so nothing is drawn in Shipping and Test builds
- **Predictive LookAt support** — Automatically estimates the target's velocity and aims ahead of its current position, reducing the visible tracking lag on a moving target. Off by default; when off, the component aims at the target's current location
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
| Input System | No dependency. The plugin does not require Enhanced Input or any other input plugin (the M, V and F keys of the included demo and the G key of the debug display read the key directly from the player controller) |

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

> **Note:** The published rotation already includes the angle clamps, the RotationSpeed smoothing and IKWeight. You do not need to process the value in the Animation Blueprint. While there is no Target and while `Pause()` is in effect it keeps its last value (the head holds its direction); with IKWeight at 0 and after `FadeOut()` it is no rotation (the animated pose).

> **Note:** The Animation Blueprint reads the component's value on its next animation update, so the head reacts 1 to 2 frames later than in a PoseableMesh setup.

> **Note:** The body's forward direction (the 0° reference for left/right) is derived from the positions of the left and right clavicle bones (`clavicle_l` / `clavicle_r`; `upperarm_l` / `upperarm_r` if those do not exist). On a skeleton without bones of these names, the mesh component's own facing is used as forward.

> **Note:** To read the value from C++ / Blueprint without an Animation Blueprint, use the component's `HeadLookAtDeltaRotation` (read-only) or `GetHeadLookAtDeltaRotation()`.

> **Note:** The Target field in the Details panel can only be assigned to actors already placed in the level. For Blueprint characters or actors not yet placed in the level, use SetTarget() at runtime instead (e.g. in BeginPlay).

### Fading the Head Look In and Out (new)

At the end of a conversation or the start of a cutscene, you can blend the IK head look out and back in instead of cutting it.

```cpp
ULogicDevLookAtSmoothIKComponent* IK = GetOwner()->FindComponentByClass<ULogicDevLookAtSmoothIKComponent>();
IK->FadeOut(0.5f);   // over 0.5 seconds, return the head and stay there
IK->FadeIn(0.5f);    // over 0.5 seconds, return to looking
IK->FadeOut();       // no argument: uses WeightFadeTime
```

- With a SkeletalMesh the head returns to the animated pose; with a PoseableMesh or a SceneComponent it returns to the pose it had when the component started
- Tracking keeps running during a fade-out, so `FadeIn()` blends straight into the current direction. Calling `FadeIn()` during a fade-out (or the other way round) turns back smoothly from where the head is
- **WeightFadeTime** (default 0) is also used when **IKWeight** is changed: the head moves to the new weight over that time. At 0 everything happens within one frame, as in the previous version
- `IsFadedOut()` (true from `FadeOut()` until `FadeIn()`) and `GetCurrentWeight()` (the weight in use, 0 to 1) tell you the state

Fading exists only on the IK head-tracking component. The full-body component turns the whole Actor, so it has no "animated pose" to return to. To stop it, use `Pause()` / `Resume()` (it keeps its direction while paused) or change the Target.

### Using Automatic Multi-Target Switching

1. Add the `LogicDev LookAt Target Selector` component to the Actor that has `LogicDev LookAt Smooth` or `LogicDev LookAt Smooth IK`. If both **LookAtComponent** and **LookAtIKComponent** are left empty, the components on the same Actor are used automatically (if the Actor has both, the same target is given to both)
2. Add the Actors that may be looked at to **Candidates**. If you plan to use priority-based selection, also set each candidate's **Priority** (0 to 100 in the Details panel, higher wins)
3. Choose how to pick with **SelectionMode**. `Nearest` picks the closest candidate (distance is measured from the location of the Actor that has the selector). `Priority` picks the candidate with the highest Priority value; if several candidates share that value, the closest one among them is picked. `Facing` (new) picks the candidate at the smallest angle from the character's front (the reference forward of the LookAt component, the direction its angle limits are measured from), seen from **ViewPoint**
4. If it switches too often, adjust **MinHoldTime** (how many seconds to keep a target after switching to it) and **SwitchMargin** (how much better a candidate must be before switching to it: distance in `Nearest` and `Priority`, angle in `Facing`). Setting either one to 0 turns it off
5. (new) To ignore candidates the character should not be able to see, choose the point to measure from with **ViewPoint** (another Actor; empty = this Actor) and **ViewPointSocket** (a bone, socket or SceneComponent name on that Actor, such as `head` for a character), and turn on any of **bUseViewAngle** (field of view: left/right and up/down from the reference forward), **bUseDistanceLimit** (MinDistance to MaxDistance from that point, in cm) and **bCheckLineOfSight** (a line from that point to the candidate, blocked by whatever blocks the **ObstacleChannel** channel). A candidate that fails any of them counts as absent. All are off by default, so nothing changes unless you turn them on
6. The selected target is assigned to the LookAt component's **Target** automatically (through `SetTarget()`). You can leave the LookAt component's own Target field empty (even if it is set, it is overwritten with the target chosen by the selector)

> **Note:** The Candidates list in the Details panel can only reference Actors already placed in the level. For Actors spawned at runtime, add them with `AddCandidate()`.

Candidates can also be added and removed from C++ or Blueprint (the nodes are in the **LogicDev | TargetSelector** category).

```cpp
// C++
ULogicDevLookAtTargetSelectorComponent* Selector = GetOwner()->FindComponentByClass<ULogicDevLookAtTargetSelectorComponent>();

// Add a candidate (the second argument is its priority; 0 if omitted).
// Passing an Actor that is already a candidate only updates its priority
Selector->AddCandidate(EnemyActor, 5);

// Change a candidate's priority
Selector->SetCandidatePriority(EnemyActor, 10);

// Remove a candidate
Selector->RemoveCandidate(EnemyActor);

// The currently selected target (nullptr when no candidate is available)
AActor* Current = Selector->GetCurrentTarget();

// Get notified when the target changes (previous target, new target).
// HandleTargetChanged must be a UFUNCTION: void HandleTargetChanged(AActor* PreviousTarget, AActor* NewTarget);
Selector->OnTargetChanged.AddDynamic(this, &AMyActor::HandleTargetChanged);
```

In Blueprint, `OnTargetChanged` can be bound with the **+** button next to **On Target Changed** in the component's Details panel (Events), or with **Bind Event to On Target Changed**.

### Viewport Display, Debug Display and Keeping Values Changed During Play (new)

**Editor viewport display (while not playing):** the three components draw lines in the editor viewport: the angle limits (light blue), the field of view / distance limits (white), each candidate (green = can be selected, grey = outside the view angle / distance, red = line of sight blocked) and the selected target (yellow). Choose when with **ShowGizmos** on each component: `Always` (default), `Selected` (while the Actor is selected) or `Off`. It needs a **Realtime** viewport. It is not drawn during Play or in a packaged game.

**Game-view debug display (`LogicDev LookAt Debug View`):** place one of these actors in the level (Place Actors window, All Classes). It draws the same information with the same colors in the game view for every LookAt component and target selector in the level (also the current look direction in orange and the predicted point in magenta). The LookAt components do not depend on this actor, so removing it changes nothing.

- The top-left corner shows a table with one row per object. The columns are fixed: Object, Mode, Target, Blocked (how many), Out of range (how many), Prediction and Head fade. A cell that does not apply stays empty, so the columns never move. Rows are in the order of the Actor names
- **bShowOverlay** (everything), **bShowAngleLimits** (angle limits), **bShowCandidates** (candidate lines and view limits), **bShowPrediction** (predicted point) and **bShowText** (the table) turn each kind on or off. **ToggleKey** (default the **G key**) shows and hides everything
- With many objects, **MaxRows** (default 8) limits the rows, and the last row says "+N more". With **bOnScreenOnly** (default off), objects the camera cannot see get no row and no lines
- It uses DrawDebug drawing, so **nothing is drawn in Shipping and Test builds** (the drawing code is not compiled). If you forget to remove it, your released game still shows nothing

**Tune in the Details panel:** to change values during Play, press **F8** to eject to the editor (or run **Simulate**), select the Actor in the Outliner and change it in the Details panel.

**Keeping values changed during Play:** use the engine's **Keep Simulation Changes**. While ejected with F8 or during Simulate, right-click the Actor you changed (in the viewport or the Outliner) and choose **Keep Simulation Changes** (or press the **K key**). The Actor's values stay after Play stops. Save the level afterwards.

### Using Predictive LookAt

1. Check **Enable Prediction** in the Details panel (under **LogicDev | LookAt | Prediction**). It is off by default; leaving it off keeps the original behavior
2. Adjust **Prediction Time** (how far ahead to lead, in seconds), **Velocity Smooth Time** (the time constant for smoothing the estimated velocity), and **Max Prediction Distance** (the cap on the lead offset, in cm) as needed. See the Parameter Reference below for details
3. The target's velocity is estimated automatically. If one of the Target Actor's components is simulating physics, its physics velocity (`GetPhysicsLinearVelocity()`) is used; otherwise, velocity is estimated from the change in the Actor's location over time. There is no setting to choose between the two — it is detected automatically
4. The internal velocity estimate resets automatically when you switch targets (whether by calling `SetTarget()`, by assigning the Target property directly, or through the target selector), when the Target becomes null (even if the same Actor is assigned again afterwards), when you call `Resume()`, and when Enable Prediction is turned back on. The previous estimate never carries over
5. The effect of prediction is easiest to see with a fast-moving target

### Controlling via Code

```cpp
// C++
ULogicDevLookAtSmoothComponent* LookAt = GetComponentByClass<ULogicDevLookAtSmoothComponent>();
LookAt->SetTarget(NewTargetActor);
LookAt->Pause();
LookAt->Resume();
LookAt->ResetClampState();

// Predictive LookAt (read-only state, for verification/debugging)
LookAt->bEnablePrediction = true;
const FVector Velocity = LookAt->EstimatedVelocity;   // cm/s
const FVector AimPoint = LookAt->PredictedPosition;   // the location actually being aimed at
```

From Blueprint, get a reference to the component and call the **Set Target** / **Pause** / **Resume** nodes the same way; **Enable Prediction**, **Estimated Velocity** and **Predicted Position** are available as Blueprint properties. The IK variant (`ULogicDevLookAtSmoothIKComponent`) shares the same API.

> **Note:** While the target selector is ticking, a target set manually with `SetTarget()` is overwritten on its next update. To choose the target manually for a while, stop the selector's tick (**Set Component Tick Enabled** = false; set it back to true to resume).

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
| `ShowGizmos` | `ELogicDevLookAtGizmoMode` | `Always` | (new) When to draw the angle limits, the facing direction and the line to the target in the editor viewport: `Always` / `Selected` (while selected) / `Off`. Only while not playing |

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
| `IKWeight` | `float` | `1.0` (range 0–1) | IK blend weight (0 = no effect, 1 = full look-at). With **WeightFadeTime** at 0, setting it to 0 on a PoseableMesh or SceneComponent setup holds the head where it is, as before (use `FadeOut()` or set WeightFadeTime to return it) |
| `WeightFadeTime` | `float` | `0` (range 0–5) | (new) Seconds over which the head blends when `FadeIn()` / `FadeOut()` is called without a duration, and when IKWeight is changed. 0 = within one frame, as before |
| `ShowGizmos` | `ELogicDevLookAtGizmoMode` | `Always` | (new) When to draw the angle limits and the line to the target in the editor viewport: `Always` / `Selected` (while selected) / `Off`. Only while not playing |
| `HeadLookAtDeltaRotation` | `FRotator` (read-only) | – | Output used by the SkeletalMesh setup only: the world-space rotation to add to the head bone (see "Using it with a SkeletalMeshComponent" above) |

> **Note (keep the left/right clamp range at 180° or less):** Keep the width from `MinAngleY` to `MaxAngleY` (`MaxAngleY` - `MinAngleY`) at 180° or less. The rotation is interpolated along the shortest arc, so with a width over 180° a turn from one limit to the other passes outside the range (through the back). When `bClampAngleY` is on and the width is over 180°, a Warning is written once to the Output Log at runtime.

> **Note (clamp side and switching targets):** With `bClampAngleY` on, while the target is outside the angle limits the rotation stays fixed at the limit on the side where the target first left the range. When the target changes to a different one - through `SetTarget()` or by writing the Target property directly - this is released, and the limit nearest to the new target is chosen again (setting the same target again does not release it; setting it to null first and then assigning a target does, even if it is the same one).

> **Note (defaults that differ from the Unity version):** `RotationSpeed` 12.0 (5.0 in the Unity version) is an intended difference that comes from the different interpolation method. The default left/right clamp of ±80 (±60 in the Unity version) is kept as it is so that the behavior of the published versions does not change. For the same range as the Unity version, set `MinAngleY` / `MaxAngleY` to ±60.

### ULogicDevLookAtTargetSelectorComponent (Automatic Multi-Target Switching)

Details panel category **LogicDev | TargetSelector**.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `LookAtComponent` | `ULogicDevLookAtSmoothComponent*` | `nullptr` | The full-body component that receives the selected target. If both this and LookAtIKComponent are empty, the components on the same Actor are found automatically at BeginPlay |
| `LookAtIKComponent` | `ULogicDevLookAtSmoothIKComponent*` | `nullptr` | The IK head-tracking component that receives the selected target (same rule as above). If both are set, the same target is given to both |
| `Candidates` | `TArray<FLogicDevLookAtCandidate>` | empty | The candidate list. Each entry has a **Target** (`AActor*`) and a **Priority** (`int32`, 0 to 100 in the Details panel). Empty, destroyed and hidden (**Actor Hidden In Game**) candidates are never selected |
| `SelectionMode` | `ELogicDevLookAtSelectionMode` | `Nearest` | `Nearest`: the closest candidate. `Priority`: the candidate with the highest Priority (the closer one if the values are equal). `Facing` (new): the candidate at the smallest angle from the reference forward, seen from ViewPoint. Priority values are used only in Priority mode |
| `MinHoldTime` | `float` | `0.5` (range 0–5) | After switching targets, the component does not switch again for this many seconds, even if a better candidate appears (0 = off). When the current target becomes unavailable, and right after SelectionMode is changed, it picks again without waiting for this time |
| `SwitchMargin` | `float` | `0.1` (range 0–0.9) | How much better a candidate must be before switching between candidates of the same Priority (0 = off). At 0.1, it switches only when the new candidate is more than 10% closer than the current target (Facing mode compares angles: if the current target is at 20°, it switches only to a candidate below 18°). In Priority mode, a candidate with a higher Priority is switched to without this margin (MinHoldTime still applies) |
| `ViewPoint` | `AActor*` | `nullptr` | (new) The Actor the view limits and the Facing angle are measured from. Empty = this Actor |
| `ViewPointSocket` | `FName` | `None` | (new) A bone, socket or SceneComponent name on ViewPoint (or this Actor if empty) to measure from instead of the Actor's location. For a character, the head (`head`, for example). None or not found = the Actor's location |
| `bUseViewAngle` | `bool` | `false` | (new) Ignore candidates outside the field of view around the reference forward (the driven LookAt component's reference forward, the direction its angle limits are measured from; this Actor's forward if it drives none) |
| `ViewAngleY` | `float` | `60` (range 0–180) | (new) Field of view to the left and to the right, in degrees (half width) |
| `ViewAngleX` | `float` | `45` (range 0–90) | (new) Field of view above and below, in degrees (half height) |
| `bUseDistanceLimit` | `bool` | `false` | (new) Ignore candidates closer than MinDistance or farther than MaxDistance from the point measured from |
| `MinDistance` | `float` | `0` | (new) cm |
| `MaxDistance` | `float` | `2000` | (new) cm |
| `bCheckLineOfSight` | `bool` | `false` | (new) Ignore candidates hidden behind something: one line is traced from the point measured from to the candidate Actor's location. This Actor, the driven LookAt Actors, ViewPoint and the candidate itself (and Actors attached to them) never block. Overlap-only collision (triggers) does not block |
| `ObstacleChannel` | `ECollisionChannel` | `Visibility` | (new) Trace channel of the line: whatever blocks this channel blocks the line of sight |
| `ShowGizmos` | `ELogicDevLookAtGizmoMode` | `Always` | (new) When to draw the view limits, the candidates and the selected target in the editor viewport: `Always` / `Selected` (while selected) / `Off`. Only while not playing |
| `OnTargetChanged` | delegate `(AActor* PreviousTarget, AActor* NewTarget)` | – | Raised whenever the selected target changes. Either argument can be null; the previous target is null if it has been destroyed. `BlueprintAssignable` |

**Functions** (`BlueprintCallable`; `GetCurrentTarget` is `BlueprintPure`)

| Function | Description |
|---|---|
| `void AddCandidate(AActor* NewCandidate, int32 Priority = 0)` | Adds a candidate. If the Actor is already a candidate, only its priority is updated. Passing null logs a warning and is ignored |
| `bool RemoveCandidate(AActor* Candidate)` | Removes a candidate. Returns false if it was not in the list |
| `bool SetCandidatePriority(AActor* Candidate, int32 Priority)` | Changes a candidate's priority. If it is not in the list, logs a warning and returns false |
| `AActor* GetCurrentTarget() const` | The currently selected target. null when no candidate is available |
| `bool IsSelectable(AActor* Candidate) const` | (new) True if the Actor could be selected right now: it exists, is not hidden and passes the view limits that are on. It does not have to be in the candidate list. Selection uses exactly this test |
| `ELogicDevLookAtCandidateStatus GetCandidateStatus(AActor* Candidate, FVector& OutBlockPoint) const` | (new) The same test broken down: `Selectable`, `Unavailable` (null / destroyed / hidden), `OutOfDistance`, `OutOfViewAngle` or `Blocked` (OutBlockPoint = where the line hits the obstacle). Read-only: it never changes the selection |
| `void GetViewFrame(FVector& OutOrigin, FVector& OutForward, FVector& OutUp) const` | (new) The point and the directions the view limits and the Facing angle are measured from, right now |

`Candidates`, `SelectionMode`, `MinHoldTime`, `SwitchMargin` and the view-limit settings are `BlueprintReadWrite`, so you can also change them directly at runtime.

Both LookAt components also have `ResetClampState()` (`BlueprintCallable`; see "Angle clamp side" under "Target Selector Behavior" below) and the read-only `GetCurrentReferenceForward()` / `GetCurrentReferenceUp()` (the direction their angle limits are measured from). The IK head-tracking component also has `FadeIn()` / `FadeOut()`, `IsFadedOut()`, `GetCurrentWeight()` and the read-only `GetCurrentHeadPosition()` / `GetCurrentAimDirection()` (new).

### Predictive LookAt Parameters (shared by both components)

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

## Target Selector Behavior

| Situation | What happens |
|---|---|
| The current target is destroyed, removed from the candidates, or hidden (Actor Hidden In Game) | A new target is picked from the remaining candidates on the next update, without waiting for MinHoldTime |
| (new) The current target leaves the field of view or the distance range, or becomes hidden (view limits on) | It counts as absent, and a new target is picked without waiting for MinHoldTime. When it can be seen again it is a normal candidate (MinHoldTime and SwitchMargin apply) |
| No candidate is available | The LookAt component's Target is set to null. The LookAt component stops rotating and keeps its last direction. `OnTargetChanged` is raised with (previous target, null). If a candidate is added or shown again later, it is selected right away |
| A better candidate appears or is added while MinHoldTime is running | The switch happens after the hold time ends |
| A candidate's Priority is changed | Takes effect from the next update. If a hold is running, the switch happens after the hold time ends |
| SelectionMode is changed | For the first update after the change only, a new target is picked without waiting for MinHoldTime (SwitchMargin still applies). Setting the same value again is not a change |
| `SetTarget()` is called manually on the LookAt component | It is overwritten with the selected target at the next update. `OnTargetChanged` is not raised |
| The LookAt component is paused with `Pause()` | Selection continues, and Target updates and `OnTargetChanged` happen as usual. Rotation stays stopped; after `Resume()` the component turns toward whichever target is selected at that time |
| The selector's tick is stopped (Set Component Tick Enabled = false) | Selection stops. The LookAt component's Target stays at the last selected target. Selection starts again from where it left off when the tick is enabled again |
| No LookAt component is found or assigned | Only target selection and `OnTargetChanged` take place (a warning is written to the Output Log at BeginPlay) |
| Angle clamp side (when bClampAngleY is on) | While the target is outside the angle limits, the LookAt component stays fixed at the limit on the side where the target first left the range. When the target changes to a different one - switched by the selector, through `SetTarget()`, or by assigning the Target property directly - this is released and the limit nearest to the new target is chosen again. Setting the same target again does not release it (setting it to null first and then assigning a target does, even if it is the same one). To release it without changing the target, call `ResetClampState()` |
| Predictive LookAt is enabled | Velocity estimation resets on every switch and starts again from the new target's current location |

- The selector ticks in the `TG_PrePhysics` tick group and is registered as a tick prerequisite of the LookAt components it drives, so it always decides before they move. A switch is reflected in the rotation of that same frame
- In `Nearest` and `Priority`, distance is the straight-line 3D distance measured from the location of the Actor that has the selector (`GetActorLocation()`). This is also true when it drives the IK head-tracking component: the reference point is the Actor's location (for a character, usually the capsule or root), not the head. The Facing angle and the view limits are measured from the point given by **ViewPoint** and **ViewPointSocket**
- The line of sight is one line from that point to the candidate Actor's location (its pivot). If the pivot is hidden, the candidate counts as hidden even if part of it can be seen (and the other way round). A candidate does not need collision of its own
- The field of view uses separate left/right and up/down angles (it is not a cone)
- Every evaluation looks at every candidate (frames are not skipped)

**Example of what MinHoldTime and SwitchMargin do (our measurement)**: with two candidates swaying back and forth at nearly the same distance (so that "the nearer one" changes 46 times in 10 seconds) and the full-body version selecting between them, the number of switches was 46 with both set to 0, 15 with only MinHoldTime 0.5, 25 with only SwitchMargin 0.1, and 11 with both (the default values).

---

## Known Limitations

**Automatic multi-target switching**

- When Predictive LookAt is enabled, velocity estimation starts over from zero every time the target switches. Frequent switching leaves less time with an effective lead, so use MinHoldTime and SwitchMargin to keep the switching rate down
- Selection uses the candidates' locations at the moment the selector ticks (`TG_PrePhysics`). Candidates that are moved later in the frame (for example by physics, or in a later tick group) are evaluated at their previous-frame locations (a change in which one is nearer is detected one frame late)

**Predictive LookAt**

- Predictive LookAt is a simple linear extrapolation based on the target's **current** velocity. Error increases when the target is accelerating, decelerating, or turning (its velocity direction is changing)
- It does not include ballistic calculations or an interception solution that accounts for projectile speed
- The physics-velocity check happens when a target is assigned. If the target only starts simulating physics after it has been assigned, its velocity keeps being estimated from its change in location until the target is changed (the result is still a valid velocity estimate)
- If the Target Actor has more than one component simulating physics, the first one found is used for the velocity

---

## Demo Scene

You can check the component behavior immediately using the included demo level `L_LookAtSmooth_Test` (`Content/LogicDevLookAtSmooth/Demo/`).

- **Left side:** Full-body rotation demo — `Watcher_Body`, mounted on a tilting platform, tracks either the yellow sphere `A` or the green sphere `A2`. You can also see that tracking continues while the platform tilts (vehicle/drone-style banking)
- **Right side:** IK head tracking demo — `IK_Character` rotates only its head to follow either the red sphere `B` or the purple sphere `B2`. The red sphere moves up and down as well as side to side, so you can also see the head tracking vertically
- **Automatic multi-target switching:** the demo starts in `Nearest` mode. The two spheres move back and forth sideways, and each time the nearer one changes the character switches the target it looks at
- **Switching the mode:** press the **M key** during Play to go through `Nearest`, `Priority` and `Facing`. In the demo, `A2` and `B2` have Priority 2 and `A` and `B` have Priority 1, so in `Priority` mode the characters keep looking at `A2` / `B2` even when they are farther away (except while those cannot be seen). To switch without the key, change **Mode** on the `Demo_SelectionMode` actor in the Details panel (this also works during Play)
- **Reading the labels:** labels are shown above each character. The mode label shows the current mode and the name of the target being looked at (for example `Nearest: A2` or `Priority: B2`); the target's name is shown in a color close to that sphere's color. The prediction label shows the Predictive LookAt state (`Prediction: OFF` / `Prediction: ON`)
- **Predictive LookAt:** both demos automatically switch prediction OFF / ON every 4.5 seconds (starting with OFF). While prediction is ON, a small marker shows the location actually being aimed at (`PredictedPosition`) for the target currently being looked at. To make the marker easy to see, the demo characters use `PredictionTime = 0.4` (the component default is 0.25)
- **Field of view and line of sight (new):** a wall stands on the tilting platform. The full-body side checks the line of sight, so it does not pick `A2` while `A2` is behind the wall (the label shows `(A2 hidden)`). The IK side has a field of view of ±70° left/right and ±45° up/down from the head and does not pick a sphere outside it (`(B2 out of view)`). Press the **V key** to turn these checks off / on
- **Head fade (new):** `IK_Character` fades its head look out and back in on a cycle (on for 8 seconds, fade out for 1 second, off for 2 seconds, fade in for 1 second). A label shows `Look: ON` / `Look: fading out` / `Look: OFF` / `Look: fading in`. Press the **F key** to fade out / in by hand (this stops the cycle)
- **Debug display (new):** the `Demo_Debug` actor (`LogicDev LookAt Debug View`) is showing from the start. Press the **G key** to hide / show it. See "Viewport Display, Debug Display and Keeping Values Changed During Play" above

> **Note:** Switching every selector's mode at once, updating the labels, toggling prediction OFF / ON and the fade cycle are all done by demo-only classes (`LogicDevDemoSelectionModeSwitcherActor`, `LogicDevDemoSelectionLabelComponent`, `LogicDevDemoPredictionToggleComponent` and `LogicDevDemoFadeToggleComponent`). You do not need them in your own levels — set **SelectionMode** on the target selector and **Enable Prediction** on the LookAt component directly.

> **Note:** The demo scene uses primitive shapes only. A demo using Mannequin is available on the Fab product page (not included in this package).

---

## Before You Import (Important)

This package uses the same plugin name (`LogicDevLookAtSmooth`), class names and namespace as the previous versions (`Move_LookAtSmooth_001`, `Move_LookAtSmooth_002`, `Move_LookAtSmooth_003` and `Move_LookAtSmooth_004`). **If you already have any of them in your project, please delete it before adding this package.** Having more than one version installed at the same time causes plugin conflicts and compile errors due to duplicate class definitions.

If you have edited any files inside a previous version's plugin folder (such as the demo level), please back them up before deleting it.

---

## Upgrading from 004

Notes for users of `Move_LookAtSmooth_004`:

- As noted in "Before You Import" above, delete 004 from your project before adding this package
- The component class names and the names of all existing properties are unchanged from 004, so the settings of the LookAt components and target selectors in your levels and Blueprints carry over. A saved SelectionMode (`Nearest`, `Priority`) keeps its meaning. New settings start at their defaults (view limits off, WeightFadeTime 0)
- With the new settings left at their defaults, the behavior is the same as 004 (004 and this version were run through the same 4,222 test conditions on UE 5.3, and the recorded rotations and selected targets matched bit for bit in every frame).
- The one addition you may notice is `ShowGizmos` (default `Always`). To stop the lines in the editor viewport, set it to `Selected` or `Off`

**If you are upgrading from 003 or earlier**, also note: the full-body component follows left/right only (see the Note under its Parameter Reference), the IK component supports a SkeletalMeshComponent through the Animation Blueprint (see "Using it with a SkeletalMeshComponent"), and changing to a different target releases the angle clamp side (see "Angle clamp side" under "Target Selector Behavior").

---

## Upgrade Path

This package (`Move_LookAtSmooth_005`) is the final version of the series (Complete Edition, with every feature). It adds the head fade, the "most in front" mode, field of view / distance / line of sight, the viewport display and the development debug display to the features of `Move_LookAtSmooth_004` (Advanced).

No higher version will be released; future updates to this package will be bug fixes and compatibility updates only.

Want to try the free version first? https://www.fab.com/listings/b16ba60d-cb73-48ab-9e13-b23cd5e8bd80

---

If you find this asset useful, please consider leaving a review on this product's Fab page — it really helps!
Leave a review here: https://www.fab.com/listings/33d152bb-bbbb-43bd-898a-99a6c092d4fd
