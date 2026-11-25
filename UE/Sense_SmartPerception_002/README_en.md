# Smart Perception Pro (Unreal Engine) - Vision, Hearing and Awareness for Enemies

A "notice" component for enemies and NPCs. Add **Perception Sensor** (`ULogicDevPerceptionSensorComponent`) to an enemy and **Perception Target** (`ULogicDevPerceptionTargetComponent`) to the player: the enemy sees (field of view, distance, obstacles, several points of the body), hears noises, feels a touch, raises an awareness meter (Unaware -> Suspicious -> Alert), remembers where it last saw or heard the player, can call its allies when it finds you, and, with several players, chooses which one to follow. It tells your AI what it noticed through functions and events (Blueprint and C++). Moving the enemy (chase, patrol, attack) is not part of this package: your AI (Blueprint, Behavior Tree, C++ and so on) reads the sensor and decides what to do.

---

## Features

- **Vision with partial cover** - Each target has several check points (feet, chest, head by default). A player behind a low wall is still partly visible; the meter rises with how much of the body is seen
- **Central and peripheral vision, distance** - Inside the central angle the meter rises at full speed, toward the edge of the view and toward the view distance it rises more slowly
- **Distance curve** - Draw how fast the meter rises by distance (for example "Alert at once within 2 m"). Off by default (a straight line is used)
- **Awareness meter with three states** - Unaware -> Suspicious -> Alert, with a hold time, a decay and a margin so the state does not flicker at a threshold
- **Memory** - The last seen position and time, kept for a set time after the target is lost
- **Hearing** - One line makes a noise: `ULogicDevPerceptionNoise::EmitNoise(this, position, range, source)` (the **Emit Noise** node in Blueprint). Weaker with the distance and through obstacles. Noises alone never make an enemy Alert
- **Touch** - A target that comes very close is noticed at once, from any direction (bumping into a guard from behind)
- **Alert call** - An enemy that becomes Alert can call out: allies nearby hear where it last saw the player and can come to look. Calls alone never make the allies Alert, so they do not spread through the whole level
- **Several players (co-op)** - Each enemy chooses one target to follow among the players it is suspicious of: the nearest (default), the highest priority, or the highest meter. A player in sight comes before one only heard, a touch switches at once, and a hold time stops it from switching back and forth
- **Many enemies** - The vision tests of many sensors are spread over frames (an interval per sensor and a cap per frame); the meter still moves smoothly every frame. The bodies of other enemies do not block the view by default, so a group of guards does not hide the player from itself
- **Events** - Blueprint events (`BlueprintAssignable`) for state changes, spotted / lost / forgotten / touched, noises heard, alert calls and a change of the followed target. In C++, bind them with `AddDynamic`
- **Debug display and tuning panel (development only)** - A game-view display of the view cones, sight lines, meters, noises and calls, and a panel to change values during Play. In Shipping and Test builds they show nothing (their drawing code is compiled out)
- **Minimap sample** - The demo includes a small radar (players, enemies with their state colour and view cone, walls) built only from the public functions, as a starting point for your own minimap

---

## Requirements

| Item | Details |
|------|---------|
| Unreal Engine | UE 5.3 / 5.5 / 5.7 / 5.8 |
| Language | C++ (plugin `LogicDevSmartPerception`, one Runtime module, Win64) |
| Blueprint | Settings are `BlueprintReadWrite`, functions `BlueprintCallable` / `BlueprintPure`, events `BlueprintAssignable`. It can be used from Blueprint only |
| Input | No dependency. No input plugin such as Enhanced Input is needed (the bundled demo's keys, the debug display's G and the tuning panel's T read the key state from the player controller) |
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

In the Details panel the settings are under **Smart Perception**: Vision, Awareness, Memory, Hearing, Alert Call, Touch, Focus, Performance and Gizmos. The Details panel shows the names with spaces between the words and without the leading `b` (for example `bAlertCall` -> **Alert Call**).

### Reading the Sensor from Your AI

In Blueprint, call the nodes **Get State**, **Get Awareness**, **Get Focus Target**, **Can See Focus**, **Try Get Last Known Position** and so on (category LogicDevSupport | SmartPerception) on the sensor component. For the events, use **+** under Events in the Details panel of the component (**On Awareness Changed**, **On Suspicious**, **On Alert**, **On Calm**, **On Target Spotted** and so on), or **Bind Event to ...**.

In C++, add `"LogicDevSmartPerception"` to the dependencies in your module's `Build.cs` first.

```cpp
#include "LogicDevPerceptionSensorComponent.h"
#include "LogicDevPerceptionTargetComponent.h"

ULogicDevPerceptionSensorComponent* Sensor = GetOwner()->FindComponentByClass<ULogicDevPerceptionSensorComponent>();

FVector Pos;
if (Sensor->GetState() == ELogicDevPerceptionAwareness::Alert && Sensor->CanSeeFocus())
{
    // chase Sensor->GetFocusTarget()->GetOwner()
}
else if (Sensor->GetState() != ELogicDevPerceptionAwareness::Unaware && Sensor->TryGetLastKnownPosition(Pos))
{
    // go to Pos and look around (last seen, touched or heard position)
}

// Events. The handlers must be UFUNCTIONs:
// void HandleAwarenessChanged(ELogicDevPerceptionAwareness Previous, ELogicDevPerceptionAwareness Current);
// void HandleSpotted(ULogicDevPerceptionTargetComponent* Target);
Sensor->OnAwarenessChanged.AddDynamic(this, &AMyAIController::HandleAwarenessChanged);
Sensor->OnTargetSpotted.AddDynamic(this, &AMyAIController::HandleSpotted);
```

**Get State** and **Get Awareness** are the highest meter over all targets. **Get Focus Target** is the one target the sensor follows ("Several Players"); the memory, the alert call and the debug display use this target.

### Noises (one line)

```cpp
#include "LogicDevPerceptionNoise.h"

// Footsteps, a gunshot, a door: pass the source (Perception Target) and the guards raise their meter of that target
ULogicDevPerceptionNoise::EmitNoise(this, GetActorLocation(), 800.0f, MyTarget);

// A thrown stone (no source): the guards only report it (On Noise Heard, Get Last Noise)
ULogicDevPerceptionNoise::EmitNoise(this, Stone->GetActorLocation(), 1000.0f, nullptr);
```

In Blueprint it is the **Emit Noise** node (it returns how many sensors heard it). Every active sensor within `range` (cm) x its **HearingSensitivity** hears it. The strength is 1 at the noise and 0 at that distance (linear in between), multiplied by **WallDamping** for each obstacle (a component that blocks ObstacleChannel) between the noise and the sensor. When heard, **NoiseAwareness** x the strength is added to the meter (strength 1 adds NoiseAwareness itself). Noises alone never raise the meter above **NoiseMaxAwareness** (0.7 by default: Suspicious, but never Alert). The guard remembers where it heard the noise (**Try Get Last Known Position**). A noise weaker than 0.01 is not heard.

You may call **Emit Noise** from inside a noise event (**On Noise Heard**), for example to pass a noise on.

### Touch

A target closer than **TouchRadius** (100 cm by default) to the enemy's body (the line from the enemy Actor's location up to the sensor), measured to any of its check points, raises the meter to at least **TouchAwareness** (1 = Alert at once), from any direction and even when it is not seen. TouchRadius 0 turns it off. If the Actor's location is the middle of the body (the centre of a capsule, for example), the "body" runs from there up to the sensor.

### Distance Curve

By default the meter rises fastest right in front of the sensor and at the **FarFactor** speed at **ViewDistance** (linear in between). Turn on **bUseDistanceCurve** to draw the factor yourself in **DistanceCurve** instead (FarFactor is then greyed out and not used):

- Horizontal: distance / ViewDistance (0 = at the sensor, 1 = at ViewDistance)
- Vertical: the speed factor (1 = normal, 0 = the meter does not rise, above 1 = faster; below 0 counts as 0)
- The default curve is the same straight line as FarFactor 0.4 ((0, 1) and (1, 0.4)), so turning it on changes nothing until you edit the curve

**Example: "Alert at once within 2 m"** (ViewDistance 1500, RiseRate 0.7): keys (0, 30), (0.13, 30), (0.14, 1), (1, 0.4). Right-click the keys and set their interpolation to **Linear** to join them with straight lines. Within 195 cm (0.13 x 1500) the meter rises 30 times faster (21 per second): Alert in about 3 frames at 60 fps. Farther away nothing changes.

### Alert Call

Turn on **bAlertCall** and an enemy calls its allies when it becomes Alert. The call is a normal noise at the sensor's location, whose source is the target it found:

- Allies within **AlertCallRange** (cm; x the listener's HearingSensitivity; obstacles weaken it with WallDamping) raise their meter of that target (up to NoiseMaxAwareness)
- **The call passes on where the caller last saw (or touched) the target** (not where the caller stands; the caller's sensor location when it has no memory of the target). The allies remember that position and **Try Get Last Known Position** returns it, so your AI can send them there. The range and the obstacles of the call are measured from the caller's sensor
- A sensor does not react to its own call
- **AlertCallInterval**: seconds between calls while the enemy stays Alert. 0 = call only once each time it becomes Alert
- **bAlertCallOnlyWhileSeen** (on by default): the call is repeated only while the target is in sight, so an enemy that lost the target stops calling. The first call when it becomes Alert is made either way
- **Calls do not chain**: a call raises the meter only up to NoiseMaxAwareness, so the allies become Suspicious but never Alert, and they do not call in turn. An ally becomes Alert only by seeing or touching the target itself (if you set NoiseMaxAwareness to AlertThreshold or more, a call can make an ally Alert, and it calls in turn)
- Event: **On Alert Call** (the target it found). **Get Last Alert Call Time** and **Get Last Alert Call Heard** (how many sensors heard the last call)

To make a call with a position you choose, use **Emit Noise Reported** (`ULogicDevPerceptionNoise::EmitNoiseReported(this, position, range, source, reportedPosition)`). The listeners remember `reportedPosition` (ReportedPosition of `FLogicDevPerceptionHeardNoise`); the range and the obstacles are measured from `position`.

### Several Players (choosing the target to follow)

With several players (co-op), each sensor follows one of them: **Get Focus Target**. It chooses only among the targets it is suspicious of (meter at **SuspiciousThreshold** or more). With none, it is the target with the highest meter (also kept for **FocusHoldTime**, so two players with low meters that overtake each other do not make it switch back and forth). **FocusMode** sets the order:

- **Nearest** (default): the nearest one from the sensor (to the target Actor's location)
- **Priority**: the highest **Priority** (a setting of the Perception Target), then the nearest one. For example, raise the priority of the player carrying the objective
- **Awareness**: the highest meter, then the nearest one (with both at 100%, the nearer one)

Rules for all three:

1. **A target in sight comes first**: a target in sight (or touching) comes before one only heard or remembered. While the current target is in sight the sensor switches only to another target in sight, so footsteps behind it do not make it turn around; the meter of the target that made them still rises and its position is remembered. When the current target is lost, it turns to the other one
2. **A touch switches at once**: a target that touches the sensor becomes the focus target at once (before the hold time and the priority), and stays it while touching
3. **No back and forth**: after a switch the target is kept for **FocusHoldTime** seconds (2 by default). After that it switches only to a clearly better target: **FocusSwitchMargin** (0.1) = at least 10% nearer, or, in Awareness, a meter at least 0.1 higher. A higher priority, a target in sight while the current one is lost, and a touch switch without a margin

Event: **On Focus Changed** (previous target, new target; either may be empty). **Get Last Focus Change Time**.

**Online games**: a sensor works only inside the machine it runs on and does not replicate. In an online game, run the sensors on one machine (usually the server) and send the results (state, focus target) to the other players with your game code. Synchronization is up to your game.

### Many Enemies (spreading the work)

The vision test of a sensor (the line traces) runs every **UpdateInterval** seconds (0.1 by default, 0 = every frame). The sensors take turns, and a cap per world (32 by default, 0 = no cap) limits the tests in one frame. Change the cap with **Get Perception Subsystem** -> **Set Max Sensors Per Frame** (C++: `ULogicDevPerceptionSubsystem::Get(this)->SetMaxSensorsPerFrame(64)`). Between two tests the meter, touch and noises still update every frame, from the last test's result.

Our measurement (UE 5.3): with 400 sensors, UpdateInterval 0.1 and a cap of 32, at most 32 vision tests ran in one frame, and each sensor ran 9 to 10 tests in 2 seconds (the 9.6 expected from the cap).

When guards gather, the front guard's body can hide the player from the guards behind it. With **bOtherSensorsBlock** off (default), the bodies of other sensors (Actors with a Perception Sensor, and Actors attached to them) neither block the view nor muffle noises. Turn it on if enemies should hide things from each other.

### Debug Display and Tuning Panel

These two are development actors. In the Place Actors window (All Classes), search for "Perception" and place one of each in the level. **In Shipping and Test builds they show nothing (their drawing code is compiled out).** The sensors do not depend on them, so you may leave them out or delete them. The bundled demo level already has both.

- **Perception Debug View (Development only)** (`ALogicDevPerceptionDebugViewActor`): for each sensor, the view (field of view, central vision (dashed), distance), a line to each check point (green = visible, red = up to the obstacle), the meter above the sensor (ticks = thresholds), the last seen position, the last heard position, a dashed line on the ground to the last known position while Suspicious / Alert (where your AI would go to look), the touch radius, noises (a growing ring and lines to the sensors that heard it; orange = heard, dashed = muffled by obstacles), alert calls (a red ring at their range), and a status table in the top-left corner (one row per sensor; columns Sensor, State, Meter, Target, Sight, Memory, Noise). **G** shows / hides it
- **Perception Tuning Panel (Development only)** (`ALogicDevPerceptionTuningPanelActor`): a small panel in the bottom-right corner to change the main settings of one sensor at a time during Play (View Distance, View Angle, Vertical Angle, Central Angle, Peripheral Factor, Far Factor, Rise Rate, Decay Rate, Decay Delay, Suspicious Threshold, Alert Threshold, Forget Time, Hearing Sensitivity, Touch Radius; `<` `>` choose the sensor). It also switches the items of the debug display. **T** shows / hides it
- **UI mode**: while the panel is on screen, the mouse cursor is shown so the panel can be clicked (input mode Game and UI: the game's keys and mouse buttons keep working). Hiding it with T gives back the game's mouse settings. With **bUIModeWhileShown** off, the panel never touches the cursor or the input mode. Skip your own camera's mouse look while `IsAnyPanelInUIMode` (static; `IsInUIMode` for one panel) is true. There are also the events `OnUIModeEntered` / `OnUIModeExited`. In Shipping and Test builds it is never in UI mode

See "Tuning While Playing" for how to use them.

**Editor viewport display (while not playing)**: a sensor draws its view (field of view, central vision, distance) and its touch radius in the editor viewport. **ShowGizmos** sets when: `Selected` (default; while the Actor is selected), `Always`, `Off`. It is drawn while the viewport's **Realtime** is on. It is not drawn during Play or in a packaged game.

---

## How It Decides

- **A point is visible** when it is within **ViewDistance** of the sensor, within **ViewAngle** left/right and **VerticalViewAngle** up/down (both measured from the sensor's forward), and no obstacle is in between
- **A target is seen** when visible points >= min(**RequiredVisiblePoints**, number of points). A target with **bDetectable** off is never seen
- **The meter rises** (per second) by RiseRate x Exposure x the target's **Visibility**. Exposure = sum of (angle factor x distance factor) over the visible points / number of points
  - Angle factor: 1 within **CentralAngle**, **PeripheralFactor** at ViewAngle (linear in between)
  - Distance factor: 1 at 0 cm, **FarFactor** at ViewDistance (linear in between), or the distance curve
- **The meter falls**: it holds for **DecayDelay** seconds after the target was last noticed (seen, heard or touched), then falls by **DecayRate** per second
- **The state** goes up exactly at a threshold, and goes down only below threshold - **CalmMargin** (it always becomes Unaware when the meter reaches 0)
- **Memory**: the position is updated while the target is seen (and while it touches). **ForgetTime** seconds after it is lost, the sensor forgets it (**On Target Forgotten**). A heard position is kept for the same time

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
| `bUseDistanceCurve` | Off | Use DistanceCurve instead of FarFactor |
| `DistanceCurve` | line 1 -> 0.4 | Factor by distance / ViewDistance ("Distance Curve") |
| `RequiredVisiblePoints` | 1 | Visible points needed to count the target as seen (1 to 8) |
| `ObstacleChannel` | Visibility | Trace channel that blocks the view and muffles noises (the sensor's and the target's own Actors and overlap-only components never block) |
| `bOtherSensorsBlock` | Off | Whether the bodies of other sensors (Actors with a Perception Sensor, and Actors attached to them) block the view and muffle noises |

### Perception Sensor - Awareness and Memory

| Parameter | Default | Description |
|-----------|---------|-------------|
| `RiseRate` | 0.7 | Meter rise per second for a fully seen target right in front, in central vision (1 = Alert in one second) |
| `DecayRate` | 0.2 | Meter fall per second once nothing has been noticed for DecayDelay seconds |
| `DecayDelay` | 2 | Seconds after the last notice (seen, heard, touched) before the meter starts to fall |
| `SuspiciousThreshold` | 0.3 | Meter value at which the state becomes Suspicious |
| `AlertThreshold` | 1 | Meter value at which the state becomes Alert (a value below SuspiciousThreshold becomes SuspiciousThreshold) |
| `CalmMargin` | 0.1 | The state goes down only this far below a threshold (or at 0) |
| `ForgetTime` | 10 | Seconds the last seen / heard position is remembered after the target is lost |

### Perception Sensor - Focus

| Parameter | Default | Description |
|-----------|---------|-------------|
| `FocusMode` | Nearest | Order in which the focus target is chosen among the targets the sensor is suspicious of: Nearest, Priority, Awareness |
| `FocusHoldTime` | 2 | Seconds the focus target is kept after a switch (not for a touch, or when the current target is lost) |
| `FocusSwitchMargin` | 0.1 | Difference needed to switch: 0.1 = at least 10% nearer, or (Awareness) a meter at least 0.1 higher |

### Perception Sensor - Hearing, Alert Call, Touch, Performance and Gizmos

| Parameter | Default | Description |
|-----------|---------|-------------|
| `HearingSensitivity` | 1 | A noise is heard up to its range x this value (0 = deaf) |
| `WallDamping` | 0.5 | Strength left after each obstacle between the noise and the sensor |
| `NoiseAwareness` | 0.5 | Meter added by a noise heard at strength 1 (proportionally less when weaker) |
| `NoiseMaxAwareness` | 0.7 | Noises (and alert calls) alone never raise the meter above this value |
| `bAlertCall` | Off | Call the other sensors when this sensor becomes Alert |
| `AlertCallRange` | 1200 (cm) | How far the call can be heard |
| `AlertCallInterval` | 0 | Seconds between calls while Alert (0 = once each time it becomes Alert) |
| `bAlertCallOnlyWhileSeen` | On | Repeat the call only while the target is in sight |
| `TouchRadius` | 100 (cm) | A target closer than this to the body is noticed at once (0 = off) |
| `TouchAwareness` | 1 | Meter value a touch sets at least |
| `UpdateInterval` | 0.1 | Seconds between vision tests (0 = every frame) |
| `ShowGizmos` | Selected | When to draw the view and the touch radius in the editor viewport (Off / Selected / Always) |

### Perception Sensor - Events

Under Events in the Details panel, or **Bind Event to ...**. In C++, `AddDynamic`.

| Event | When |
|-------|------|
| `OnAwarenessChanged` | The state changed (previous state, new state) |
| `OnSuspicious`, `OnAlert`, `OnCalm` | The state became Suspicious / Alert / Unaware |
| `OnTargetSpotted` | A target came into sight |
| `OnTargetLost` | A target went out of sight |
| `OnTargetForgotten` | A target was forgotten (ForgetTime after it was last seen or touched) |
| `OnTargetTouched` | A target came within TouchRadius |
| `OnNoiseHeard` | A noise was heard (`FLogicDevPerceptionHeardNoise`: Position, ReportedPosition, Range, Source, Strength, Obstacles, Time) |
| `OnAlertCall` | Right after the sensor called its allies (the target it found) |
| `OnFocusChanged` | The focus target changed (previous target, new target; either may be empty) |

### Perception Sensor - Functions

| Function | Description |
|--------|-------------|
| `GetState()`, `GetAwareness()` | Current state and meter (0..1): the highest meter over all targets |
| `GetFocusTarget()`, `GetFocusRecord(Out)`, `CanSeeFocus()` | The target the sensor follows ("Several Players"), what it knows about it (`FLogicDevPerceptionRecord`), whether it is in sight |
| `GetLastFocusChangeTime()` | When the focus target last changed |
| `GetRecords()`, `GetRecord(Target, Out)`, `CanSee(Target)`, `GetTimeSinceSeen(Record)` | Everything the sensor knows (one `FLogicDevPerceptionRecord` per target) |
| `HasMemory()`, `GetLastSeenPosition()`, `GetLastSeenTime()` | Memory of the focus target |
| `TryGetLastKnownPosition(Out)` | The newest position of the focus target (last seen, touched or heard) |
| `ForgetTarget(Target)`, `ResetAwareness()` | Forget one target (its meter is kept) / set every meter to 0 and forget everything |
| `HasHeardNoise()`, `GetLastNoise()`, `GetNoiseStrength(...)`, `HearNoise(...)` | The last noise heard, how strongly a noise would be heard, make this sensor hear a noise |
| `GetLastAlertCallTime()`, `GetLastAlertCallHeard()` | When the sensor last called, and how many sensors heard it |
| `GetTouchDistance(Target)` | Distance (cm) from the body to the nearest check point |
| `GetViewFrame(Origin, Forward, Up)`, `GetViewAngles(...)` | Where the sensor looks from and its directions; the left/right and up/down angles of a direction |
| `DistanceFactorAt(cm)`, `AngleFactorAt(Yaw)`, `TestPoint(...)`, `NextState(...)` | The speed factors, the vision test of one point, the next state for a meter value |
| `GetLastVisionTestTime()`, `GetVisionTestCount()` | When the last vision test ran, and how many have run |
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
| `Priority` | 0 | Used by sensors whose FocusMode is Priority. Higher = followed first |

Functions: `GetCheckPoints(Out)` (the world positions of the points now), static `GetAllTargets(WorldContext)`.

### Other Types

| Name | Description |
|------|-------------|
| `ULogicDevPerceptionNoise::EmitNoise(WorldContext, Position, Range, Source)` | Makes a noise (Blueprint: **Emit Noise**). Returns how many sensors heard it |
| `ULogicDevPerceptionNoise::EmitNoiseReported(WorldContext, Position, Range, Source, ReportedPosition)` | The same, and passes on a separate position for the listeners to remember ("Alert Call") |
| `ULogicDevPerceptionSubsystem` | The sensors and targets of one world. `GetPerceptionSubsystem`, `SetMaxSensorsPerFrame` / `GetMaxSensorsPerFrame` (32 by default, 0 = no cap), `GetSensorCount`, `GetTestsThisFrame` |
| `FLogicDevPerceptionRecord` | Per target: Target, Awareness, bIsSeen, VisibleFraction, Exposure, LastSeenTime, Points, bHasMemory, LastSeenPosition, MemoryTime, bHasHeard, LastHeardPosition, LastHeardTime, bIsTouching |
| `ELogicDevPerceptionAwareness` | Unaware, Suspicious, Alert |
| `ELogicDevPerceptionFocusMode` | Nearest, Priority, Awareness |
| `ELogicDevPerceptionPointStatus` | Visible, OutOfRange, OutOfAngle, Blocked |

### Differences from the Unity Version

The features and default values are the same as Smart Perception Pro for Unity. The differences:

| Item | Unity | Unreal Engine |
|---|---|---|
| Eyes | `Eyes` (a Transform; empty = itself) | The sensor component itself (its location and +X) |
| Distance unit | m (ViewDistance 15, AlertCallRange 12, TouchRadius 1, Height 1.8) | cm (1500, 1200, 100, 180) |
| Obstacles | `ObstacleLayers` (layers); triggers do not block | `ObstacleChannel` (trace channel); overlap-only components do not block |
| Names | `PerceptionSensor`, `State`, `AwarenessChanged` and so on | `ULogicDevPerceptionSensorComponent`, `GetState()`, `OnAwarenessChanged` and so on. Boolean settings start with `b` (`bAlertCall` and so on) |
| Events | C# events and UnityEvents | Blueprint events (dynamic multicast delegates) |
| Cap per frame | `PerceptionScheduler.MaxSensorsPerFrame` | `SetMaxSensorsPerFrame` of `ULogicDevPerceptionSubsystem` |
| Keeping values changed during Play | Record values in the panel | The engine's Keep Simulation Changes ("Tuning While Playing") |
| Adding the debug tools | The menu Add Debug Tools to Scene | Place the two actors in the level |

---

## Tuning While Playing

The sensor's settings and the debug display's settings take effect at once when you change them during Play. There are two ways.

### 1. In the Details panel (Play in Editor)

1. Press Play and leave the game running. If the game view holds the mouse, press **Shift+F1** to get the mouse back, or **F8** to give control back to the editor (Eject). You may also run the level with **Simulate**.
2. Select what to change in the **Outliner** and change the values in the **Details** panel.
   - **A guard's sensor**: select the guard (`Guard_A` and so on in the bundled demo), then select **Sensor** in the component list at the top of the Details panel. Under **Smart Perception** (open it with ▶) are Vision (view distance and angles), Awareness (how fast the meter rises and falls, the thresholds), Memory, Hearing, Touch and more. A new value is used at once, and the meter is kept.
   - **The debug display**: select **Perception Debug View** in the Outliner (outside the folders). In the Details panel, **Debug View**: show it or not, and its key (Toggle Key, G by default). **What to Draw**: the view, the lines, the meters, memory, noises, touch and the status table, each on or off. **Which Sensors**: only the sensor chosen in the tuning panel, only the sensors on screen. **Debug Look**: how long a noise stays on screen, the rows of the table, the height of the meters.
   - **The tuning panel**: select **Perception Tuning Panel** in the Outliner (outside the folders). In the Details panel, **Tuning Panel**: start hidden (Start Hidden), its key (Toggle Key, T by default), show the mouse cursor while the panel is shown (UI Mode While Shown).
3. **The values you changed during Play go back when Play ends.** To keep them, after ejecting with F8 or while in Simulate, right-click the actor you changed in the viewport or the Outliner and choose **Keep Simulation Changes** (or press **K**). Then save the level.

### 2. The tuning panel on the screen (Play in Editor and games)

- **T** shows / hides the panel in the bottom-right corner. While it is shown, the mouse cursor is shown and you can click its buttons (the game's keys keep working).
- Buttons: **`<` `>`** choose the sensor to change, **`-` `+`** one step down / up, **`o`** switch a display item on / off.
- **G** shows / hides the debug display.
- Values changed with the panel also go back when Play ends (keep them with Keep Simulation Changes above, or copy them to the Details panel).
- **In Play in Editor, the game view may not get the keys right after you press Play. Click the game view once, then press T.** On a small game view the panel shrinks to fit.

The debug display and the tuning panel are development tools. In Shipping and Test builds they show nothing (their code is compiled out).

---

## Known Limitations

- Each point is tested with one line trace from the sensor. A point right behind the edge of an obstacle counts as hidden even if the body around it is slightly visible
- The view is two angles (left/right and up/down), not a round cone
- Noises go straight through obstacles (weaker for each one) and do not travel around corners. When one wall is made of several components, each counts as one obstacle
- When a sensor becomes Alert outside its own update (from a noise or a call it heard), its call is made in its next update
- Network synchronization (replication) is not included. In an online game, run the sensors on one machine and synchronize the results with your game code ("Several Players")
- The debug display and the tuning panel are development tools and show nothing in Shipping and Test builds

---

## Demo Scene

Open the bundled demo level `L_SmartPerception_Demo` (the plugin's `Content/Demo/`; in the Content Browser it appears in the `Demo` folder of the plugin `LogicDevSmartPerception` once **Show Plugin Content** is on in the settings) and press Play. Three guards patrol an area surrounded by walls. You are the blue cylinder (`Player`) in the small hiding place at the bottom, with a second player (teal, `Player_2`) next to you.

- **Controls**: **WASD / arrow keys** move (footsteps can be heard), hold **Shift** to sneak (slower, silent, harder to notice), **Space** claps (a loud noise), **Tab** switches the player you move (the other one stands still), **R** restarts (every guard forgets you, the players go back to the start), **M** toggles the minimap (shown at first), **G** the debug display, **T** the tuning panel (hidden at first; while it is hidden, a small "T: Tuning" hint is in the bottom-right corner)
- **Guards**: they walk their patrol routes (a thin line on the ground to the waypoint they are walking to). Their colour shows the state (grey = Unaware, yellow = Suspicious, red = Alert). While Suspicious or Alert they stop and turn toward where they last saw or heard you
- **Chase and search**: an Alert guard chases the player it follows. When it loses the player, it walks to the last seen position, looks around and goes back to its patrol
- **Noises**: walking makes footsteps. A guard that hears them behind it becomes Suspicious, walks to where it heard them, looks around and goes back to its patrol. A sound too weak to make it Suspicious only makes it turn toward the sound
- **Alert call**: the guards have bAlertCall on (range 1800 cm, every 2 seconds while Alert and you are in sight). When one guard finds you, nearby guards hear the call (a red ring in the debug display), walk to where the caller last saw you, look around and go back to their patrol
- **Two players**: put one player in a guard's view, then press Tab and move the other one closer or bump it into the guard. The guard follows the nearer of the players it can see and turns at once to a player that touches it
- **Minimap**: the radar in the top-right corner (M), centred on the player you move, shows the walls, the players (arrows) and the guards (a dot in the state colour and the view cone). The sensor part is drawn only with public functions (`GetAllSensors`, `GetState`, `GetViewFrame`, `ViewAngle`, `ViewDistance`, `GetAllTargets`), so you can use it as a sample for your own minimap (`ALogicDevPerceptionDemoHUD`)
- **Hide behind cover**: a low wall hides only your feet (the guard sees part of you and notices slowly). A high wall hides all of you
- **Touch**: bump into a guard from behind and it notices you at once

> Note: patrolling, chasing, walking to a noise or a call, looking around and stepping around other guards are done by demo-only classes (`ULogicDevPerceptionDemoPatrolComponent`, `ULogicDevPerceptionDemoChaseComponent` and the others with `Demo` in their names). They are examples of what an AI can do with the sensor, not features of the product. **They only move in straight lines and do no path finding.** A guard that cannot get around a wall stops in front of it, looks around and goes back to its patrol. In a real game, move your enemies with the Navigation Mesh (AI Move To and so on) and use the sensor's positions (**Try Get Last Known Position** and so on) as destinations.

> Note: The demo scene uses primitive shapes only. Images with mannequins are shown on the Fab product page (they are not included in this package).

---

## Troubleshooting

- **If the editor crashes right after opening the bundled demo (a GPU crash, for example on RTX 40 series cards)**, first update the GPU driver. If that does not help, add `r.RDG.AsyncCompute=0` under `[SystemSettings]` in `Config/DefaultEngine.ini` (an engine or driver problem in UE's shadow and lighting features; the plugin's code is not involved).
- **If T, G or other keys do nothing right after you press Play in Play in Editor**: the bundled demo turns the keyboard focus to the game view at the start of Play by itself (the demo's player controller, `ALogicDevPerceptionDemoPlayerController`). If the keys do nothing in your own level, click the game view once and press the key again, or turn on **Level Editor > Play > Game Gets Mouse Control** in the Editor Preferences (it is off by default in UE, and right after Play is pressed with the mouse the keys stay with the editor).
- **If a sensor sees nothing**, check the sensor's location and forward (+X; select the Actor in the viewport to see its view lines), the target's **bDetectable** and **Visibility** (0 = never noticed), and whether something in between blocks **ObstacleChannel**.
- **If the demo level does not appear in the Content Browser**, turn on **Show Plugin Content** in the Content Browser settings.

---

## Videos

- Setup guide: https://youtu.be/csn4rUaDtyI
- Overview: https://youtu.be/ySbOo_wA780
- Short: https://youtu.be/6ODQaQLS4ns
- Smart Perception Lite: overview https://youtu.be/Xk8i9PZxtCE / short https://youtu.be/OdIGHwVC3cc

---

## Moving from the Free Version (Smart Perception Lite)

There is also a free version, Smart Perception Lite (vision, awareness and events only). Levels and Blueprints made with Lite keep their settings and event bindings after you change to Pro (the same plugin name, class names and setting names).

https://www.fab.com/listings/394b12a0-78f8-470e-bcc5-0218e851a571

**If Lite is already in your project, remove it before you add Pro** (close the editor; if it is in your project's `Plugins/LogicDevSmartPerception/` folder, delete that folder; if it is installed in the engine, remove it from the engine). Both versions use the same plugin name (`LogicDevSmartPerception`) and the same class names, so having both makes the plugins conflict, and copying Pro on top of Lite can leave Lite files behind. If you have edited any files inside the Lite plugin (such as the demo level), back them up first.

---

If you find this asset useful, please consider leaving a review on this product's Fab page - it really helps!

https://www.fab.com/listings/ab734007-2ae6-4937-927d-8da8e1eb9e2f
