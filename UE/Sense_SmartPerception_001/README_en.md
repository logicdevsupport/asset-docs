# Smart Perception Lite (Unreal Engine) - Vision and Awareness for Enemies

A "notice" component for enemies and NPCs (free version). Add **Perception Sensor** (`ULogicDevPerceptionSensorComponent`) to an enemy and **Perception Target** (`ULogicDevPerceptionTargetComponent`) to the player: the enemy sees (field of view, distance, obstacles, several points of the body) and raises an awareness meter (Unaware -> Suspicious -> Alert). It tells your AI what it noticed through functions and events (Blueprint and C++). Moving the enemy (chase, patrol, attack) is not part of this package: your AI (Blueprint, Behavior Tree, C++ and so on) reads the sensor and decides what to do.

---

## Features

- **Vision with partial cover** - Each target has several check points (feet, chest, head by default). A player behind a low wall is still partly visible; the meter rises with how much of the body is seen
- **Central and peripheral vision, distance** - Inside the central angle the meter rises at full speed, toward the edge of the view and toward the view distance it rises more slowly
- **Awareness meter with three states** - Unaware -> Suspicious -> Alert, with a hold time, a decay and a margin so the state does not flicker at a threshold
- **Events** - Blueprint events (`BlueprintAssignable`) for state changes and spotted / lost. In C++, bind them with `AddDynamic`
- **Editor viewport display** - The view (field of view, central vision, distance) drawn as lines in the editor viewport
- **Groups of enemies** - The bodies of other enemies do not block the view by default, so a group of guards does not hide the player from itself

---

## Requirements

| Item | Details |
|------|---------|
| Unreal Engine | UE 5.3 / 5.5 / 5.7 / 5.8 |
| Language | C++ (plugin `LogicDevSmartPerception`, one Runtime module, Win64) |
| Blueprint | Settings are `BlueprintReadWrite`, functions `BlueprintCallable` / `BlueprintPure`, events `BlueprintAssignable`. It can be used from Blueprint only |
| Input | No dependency. No input plugin such as Enhanced Input is needed (the bundled demo's keys read the key state from the player controller) |
| Collision | Obstacles are found with line traces against components that block **ObstacleChannel** (Visibility by default). The player needs no collision (the points can be made from **Height**) |
| Units | All distances are Unreal units (cm). Angles are degrees, times are seconds |

---

## How to Use

### Setup (add two components)

1. Add **Perception Sensor** to the enemy Actor (Add Component, search for "Perception"; category LogicDevSupport)
2. Place the sensor **at the eyes** and turn its forward (+X) to where the enemy looks. **The sensor component itself is the eyes**: the view is measured from its location and its forward (+X). To put it on a character's head, choose the head bone or socket as the sensor's Parent Socket
3. Add **Perception Target** to the player Actor. When **CheckPoints** is empty, three points (feet, chest, head) are made from the Actor's colliding components (the capsule of a Character), or from **Height** if there are none
4. Press Play. The enemy's state follows what it sees. Read **Get State** or use the events

Everything works with the default values. Obstacles are components that block **ObstacleChannel** (Visibility by default). The sensor's own Actor and the target's Actor (each with the Actors attached to it) and overlap-only components never block. The bodies of other sensors (Actors with a Perception Sensor, and Actors attached to them) do not block unless **bOtherSensorsBlock** is on.

In the Details panel the settings are under **Smart Perception**: Vision, Awareness and Gizmos. The Details panel shows the names with spaces between the words and without the leading `b` (for example `bOtherSensorsBlock` -> **Other Sensors Block**).

### Reading the Sensor from Your AI

In Blueprint, call the nodes **Get State**, **Get Awareness**, **Get Focus Target**, **Can See Focus** and so on (category LogicDevSupport | SmartPerception) on the sensor component. For the events, use **+** under Events in the Details panel of the component (**On Awareness Changed**, **On Suspicious**, **On Alert**, **On Calm**, **On Target Spotted**, **On Target Lost**), or **Bind Event to ...**.

In C++, add `"LogicDevSmartPerception"` to the dependencies in your module's `Build.cs` first.

```cpp
#include "LogicDevPerceptionSensorComponent.h"
#include "LogicDevPerceptionTargetComponent.h"

ULogicDevPerceptionSensorComponent* Sensor = GetOwner()->FindComponentByClass<ULogicDevPerceptionSensorComponent>();

if (Sensor->GetState() == ELogicDevPerceptionAwareness::Alert && Sensor->CanSeeFocus())
{
    // chase Sensor->GetFocusTarget()->GetOwner()
}
else if (Sensor->GetState() == ELogicDevPerceptionAwareness::Suspicious)
{
    // stop and turn toward Sensor->GetFocusTarget()
}

// Events. The handlers must be UFUNCTIONs:
// void HandleAwarenessChanged(ELogicDevPerceptionAwareness Previous, ELogicDevPerceptionAwareness Current);
// void HandleSpotted(ULogicDevPerceptionTargetComponent* Target);
Sensor->OnAwarenessChanged.AddDynamic(this, &AMyAIController::HandleAwarenessChanged);
Sensor->OnTargetSpotted.AddDynamic(this, &AMyAIController::HandleSpotted);
```

**Get State** and **Get Awareness** are the highest meter over all targets. **Get Focus Target** is the target with that meter.

**Online games**: a sensor runs only on the machine it is on and does no replication. In an online game, run the sensors on one machine (usually the server) and send the results (the state and so on) to the other players with your game code.

### Editor Viewport Display (while not playing)

A sensor draws its view (field of view, central vision, distance) as lines in the editor viewport. **ShowGizmos** sets when: `Selected` (default; while the Actor is selected), `Always`, `Off`. It is drawn while the viewport's **Realtime** is on. It is not drawn during Play or in a packaged game (during Play, read the state and show it your own way; in the bundled demo the guards' colour shows their state).

---

## How It Decides

- **A point is visible** when it is within **ViewDistance** of the sensor, within **ViewAngle** left/right and **VerticalViewAngle** up/down (both measured from the sensor's forward), and no obstacle is in between
- **A target is seen** when visible points >= min(**RequiredVisiblePoints**, number of points). A target with **bDetectable** off is never seen
- **The meter rises** (per second) by RiseRate x Exposure x the target's **Visibility**. Exposure = sum of (angle factor x distance factor) over the visible points / number of points
  - Angle factor: 1 within **CentralAngle**, **PeripheralFactor** at ViewAngle (linear in between)
  - Distance factor: 1 at 0 cm, **FarFactor** at ViewDistance (linear in between)
- **The meter falls**: it holds for **DecayDelay** seconds after the target was last seen, then falls by **DecayRate** per second
- **The state** goes up exactly at a threshold, and goes down only below threshold - **CalmMargin** (it always becomes Unaware when the meter reaches 0)

---

## Parameter Reference

### Perception Sensor - Vision

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ViewDistance` | 1500 (cm) | How far the sensor can see (50 to 20000) |
| `ViewAngle` | 60 | Half of the horizontal field of view (degrees; 60 = 120 degrees wide, 180 = all around) |
| `VerticalViewAngle` | 45 | Half of the vertical field of view (degrees, from the sensor's forward) |
| `CentralAngle` | 25 | Central vision left/right (degrees). Here the meter rises fastest (a value above ViewAngle becomes ViewAngle) |
| `PeripheralFactor` | 0.3 | Speed of the meter at the edge of the view (relative to central vision) |
| `FarFactor` | 0.4 | Speed of the meter at ViewDistance (relative to right in front; linear in between) |
| `RequiredVisiblePoints` | 1 | Visible points needed to count the target as seen (1 to 8) |
| `ObstacleChannel` | Visibility | Trace channel that blocks the view (the sensor's and the target's own Actors and overlap-only components never block) |
| `bOtherSensorsBlock` | Off | Whether the bodies of other sensors (Actors with a Perception Sensor, and Actors attached to them) block the view |

### Perception Sensor - Awareness and Gizmos

| Parameter | Default | Description |
|-----------|---------|-------------|
| `RiseRate` | 0.7 | Meter gained per second with the whole body in sight right in front, in central vision (1 = Alert in 1 second) |
| `DecayRate` | 0.2 | Meter lost per second after DecayDelay seconds without seeing the target |
| `DecayDelay` | 2 | Seconds the meter holds after the target was last seen |
| `SuspiciousThreshold` | 0.3 | Meter value at which the state becomes Suspicious |
| `AlertThreshold` | 1 | Meter value at which the state becomes Alert (a value below SuspiciousThreshold becomes SuspiciousThreshold) |
| `CalmMargin` | 0.1 | The state goes down only when the meter is this far below the threshold (or at 0) |
| `ShowGizmos` | Selected | When to draw the view in the editor viewport (Off / Selected / Always) |

### Perception Sensor - Events

Under Events in the Details panel, or **Bind Event to ...**. In C++, `AddDynamic`.

| Event | When |
|-------|------|
| `OnAwarenessChanged` | The state changed (previous state, new state) |
| `OnSuspicious`, `OnAlert`, `OnCalm` | The state became Suspicious / Alert / Unaware |
| `OnTargetSpotted` | A target came into sight |
| `OnTargetLost` | A target went out of sight |

### Perception Sensor - Functions

| Function | Description |
|--------|-------------|
| `GetState()`, `GetAwareness()` | Current state and meter (0..1): the highest meter over all targets |
| `GetFocusTarget()`, `GetFocusRecord(Out)`, `CanSeeFocus()` | The target with the highest meter, what the sensor knows about it (`FLogicDevPerceptionRecord`), whether it is in sight |
| `GetRecords()`, `GetRecord(Target, Out)`, `CanSee(Target)`, `GetTimeSinceSeen(Record)` | Everything the sensor knows (one `FLogicDevPerceptionRecord` per target) |
| `ResetAwareness()` | Set every meter to 0 (the state becomes Unaware) |
| `GetViewFrame(Origin, Forward, Up)`, `GetViewAngles(...)` | Where the sensor looks from and its directions; the left/right and up/down angles of a direction |
| `DistanceFactorAt(cm)`, `AngleFactorAt(Yaw)`, `TestPoint(...)`, `NextState(...)` | The speed factors, the vision test of one point, the next state for a meter value |
| `Sense(DeltaTime)` | Step the sensor by hand (turn off the component's tick and call it with your own time step) |
| static `GetAllSensors(WorldContext)` | Every sensor of the world |
| static `IsSensorBody(Actor)` | Whether the Actor or its attach parent has a Perception Sensor |

### Perception Target (`ULogicDevPerceptionTargetComponent`)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `CheckPoints` | (empty) | Points the sensors test: socket or bone names of the Actor's meshes, or names of its scene components. Empty = feet, chest, head from the colliding components (or from Height) |
| `Height` | 180 (cm) | Body height for the automatic points when the Actor has no colliding component (the Actor's location is the feet) |
| `Visibility` | 1 | How easy the target is to notice (0 to 2; 0.5 = the meter rises half as fast, 0 = never noticed) |
| `bDetectable` | On | Off = the sensors ignore this target (in a hiding place, for example) |

Functions: `GetCheckPoints(Out)` (the world positions of the points now), static `GetAllTargets(WorldContext)`.

### Other Types

| Name | Description |
|------|-------------|
| `ULogicDevPerceptionSubsystem` | The sensors and targets of one world (`GetPerceptionSubsystem`) |
| `FLogicDevPerceptionRecord` | Per target: Target, Awareness, bIsSeen, VisibleFraction, Exposure, LastSeenTime, Points |
| `ELogicDevPerceptionAwareness` | Unaware, Suspicious, Alert |
| `ELogicDevPerceptionPointStatus` | Visible, OutOfRange, OutOfAngle, Blocked |

### Differences from the Unity Version

The features and default values are the same as Smart Perception Lite for Unity. The differences:

| Item | Unity | Unreal Engine |
|---|---|---|
| Eyes | `Eyes` (a Transform; empty = the object itself) | The sensor component itself (its location and +X) |
| Distance unit | m (ViewDistance 15, Height 1.8) | cm (1500, 180) |
| Obstacles | `ObstacleLayers` (layers); triggers never block | `ObstacleChannel` (a trace channel); overlap-only components never block |
| Names | `PerceptionSensor`, `State`, `AwarenessChanged` and so on | `ULogicDevPerceptionSensorComponent`, `GetState()`, `OnAwarenessChanged` and so on. Boolean settings start with `b` (`bOtherSensorsBlock` and so on) |
| Events | C# events and UnityEvents | Blueprint events (dynamic multicast delegates) |
| Display | Scene view gizmos (during Play also the lines to the points and the state) | The view lines in the editor viewport (while not playing) |

---

## Known Limitations

- Each point is tested with one line trace from the sensor. A point right behind the edge of an obstacle counts as hidden even if the body around it is slightly visible
- The view is two angles (left/right and up/down), not a round cone
- Every sensor runs its vision test every frame (the cost grows with sensors x targets x points)
- Network synchronization (replication) is not included

---

## Demo Scene

Open the bundled demo level `L_SmartPerception_Demo` (the plugin's `Content/Demo/`; in the Content Browser it appears in the `Demo` folder of the plugin `LogicDevSmartPerception` once **Show Plugin Content** is on in the settings) and press Play. Three guards patrol an area surrounded by walls. You are the blue cylinder (`Player`) in the small hiding place at the bottom.

- **Controls**: **WASD / arrow keys** move, hold **Shift** to sneak (slower, harder to notice), **R** restarts (every guard calms down, the player goes back to the start)
- **Guards**: they walk their patrol routes (a thin line on the ground to the waypoint they are walking to). Their colour shows the state (grey = Unaware, yellow = Suspicious, red = Alert). While Suspicious or Alert they stop, and turn toward you while they can see you. When they calm down they go back to their patrol
- **Hide behind cover**: a low wall hides only your feet (the guard sees part of you and notices slowly). A high wall hides all of you
- **View**: the guards' views are drawn in the editor viewport while not playing (their ShowGizmos is set to Always)

> Note: patrolling, stopping and turning are done by demo-only classes (`ULogicDevPerceptionDemoPatrolComponent` and the others with `Demo` in their names). They are examples of what an AI can do with the sensor, not features of the product. **They only move in straight lines and do no path finding.** In a real game, move your enemies with the Navigation Mesh (AI Move To and so on).

> Note: The demo scene uses primitive shapes only. Images with mannequins are shown on the Fab product page (they are not included in this package).

---

## Troubleshooting

- **If the editor crashes right after opening the bundled demo (a GPU crash, for example on RTX 40 series cards)**, first update the GPU driver. If that does not help, add `r.RDG.AsyncCompute=0` under `[SystemSettings]` in `Config/DefaultEngine.ini` (an engine or driver problem in UE's shadow and lighting features; the plugin's code is not involved).
- **If the keys do nothing right after you press Play in Play in Editor**: the bundled demo turns the keyboard focus to the game view at the start of Play by itself (the demo's player controller, `ALogicDevPerceptionDemoPlayerController`). If the keys do nothing in your own level, click the game view once and press the key again, or turn on **Level Editor > Play > Game Gets Mouse Control** in the Editor Preferences (it is off by default in UE, and right after Play is pressed with the mouse the keys stay with the editor).
- **If a sensor sees nothing**, check the sensor's location and forward (+X; select the Actor in the viewport to see its view lines), the target's **bDetectable** and **Visibility** (0 = never noticed), and whether something in between blocks **ObstacleChannel**.
- **If the demo level does not appear in the Content Browser**, turn on **Show Plugin Content** in the Content Browser settings.

---

## Upgrade to Smart Perception Pro

Smart Perception Pro has everything in Lite (the same plugin name, classes, settings and events, with the same names) and adds:

- **Memory** - The last seen position and time, kept for a set time after the target is lost
- **Hearing** - One line makes a noise (**Emit Noise**): weaker with the distance and through obstacles
- **Touch** - A target that bumps into the enemy is noticed at once, from any direction
- **Alert call** - An Alert enemy calls its allies, who come to look where it last saw the player
- **Distance curve** - Draw how fast the meter rises by distance (for example "Alert at once within 2 m")
- **Several players** - Each enemy chooses which player to follow (nearest, priority or meter), without switching back and forth
- **Many enemies** - The vision tests are spread over frames
- **Debug display and tuning panel** - The view, meters, noises and calls in the game view, and a panel to tune the sensors while playing (development only)
- **A larger demo** - Chase and search, footsteps and claps, alert calls, two players and a minimap sample

https://www.fab.com/listings/ab734007-2ae6-4937-927d-8da8e1eb9e2f

**Moving from Lite to Pro**: close the editor and remove Lite before you add Pro (if it is in your project's `Plugins/LogicDevSmartPerception/` folder, delete that folder; if it is installed in the engine, remove it from the engine). Both versions use the same plugin name (`LogicDevSmartPerception`) and the same class names, so having both makes the plugins conflict, and copying Pro on top of Lite can leave Lite files behind. If you have edited any files inside the plugin (such as the demo level), back them up first. Levels and Blueprints made with Lite keep their settings and event bindings after the change.

---

If you find this asset useful, please consider leaving a review on this product's Fab page - it really helps!

https://www.fab.com/listings/394b12a0-78f8-470e-bcc5-0218e851a571
