# Smart Perception Pro - Vision, Hearing and Awareness for Enemies

A "notice" component for enemies and NPCs. Add `PerceptionSensor` to an enemy and `PerceptionTarget` to the player: the enemy sees (field of view, distance, walls, several points of the body), hears noises, feels a touch, raises an awareness meter (Unaware -> Suspicious -> Alert), remembers where it last saw or heard the player, can call its allies when it finds you, and, with several players, chooses which one to follow. It tells your AI what it noticed through properties and events. Moving the enemy (chase, patrol, attack) is not part of this package: your AI reads the sensor and decides what to do.

---

## Features

- **Vision with partial cover** - Each target has several check points (feet, chest, head by default). A player behind a low wall is still partly visible; the meter rises with how much of the body is seen
- **Central and peripheral vision, distance** - Inside the central angle the meter rises at full speed, toward the edge of the view and toward the view distance it rises more slowly
- **Distance curve** - Draw how fast the meter rises by distance (for example "Alert at once within 2 m"). Off by default (a straight line is used)
- **Awareness meter with three states** - Unaware -> Suspicious -> Alert, with a hold time, a decay and a margin so the state does not flicker at a threshold
- **Memory** - The last seen position and time, kept for a set time after the target is lost
- **Hearing** - One line makes a noise: `PerceptionNoise.Emit(position, range, source)`. Weaker with the distance and through walls. Noises alone never make an enemy Alert
- **Touch** - A target that comes very close is noticed at once, from any direction (bumping into a guard from behind)
- **Alert call** - An enemy that becomes Alert can call out: allies nearby hear where it last saw the player and come to look. Calls alone never make the allies Alert, so they do not spread through the whole level
- **Several players (co-op)** - Each enemy chooses one target to follow among the players it is suspicious of: the nearest (default), the highest priority, or the highest meter. A player in sight comes before one only heard, a touch switches at once, and a hold time stops it from switching back and forth
- **Many enemies** - The vision tests of many sensors are spread over frames (an interval per sensor and a cap per frame); the meter still moves smoothly every frame. The bodies of other enemies do not block the view by default, so a group of guards does not hide the player from itself
- **Events** - C# events and UnityEvents for state changes, spotted / lost / forgotten / touched, noises heard, alert calls and a change of the followed target
- **Debug display and tuning panel (development only)** - A Game-view display of the view cones, sight lines, meters, noises and calls, a runtime tuning panel, and a way to keep values changed in Play Mode. They exist only in the Editor and in Development Builds
- **Minimap sample** - The demo includes a small radar (players, enemies with their state colour and view cone, walls) built only from the public API, as a starting point for your own minimap

---

## Requirements

| Item | Details |
|------|---------|
| Unity Version | Unity 2022.3 LTS / Unity 6.3 LTS |
| Render Pipeline | Built-in Render Pipeline, URP and HDRP (tested: Unity 2022.3 URP / Built-in / HDRP, Unity 6.3 URP / Built-in / HDRP). The components do not use any render-pipeline-specific API. The materials of the included demo scene are switched to the standard shader of the render pipeline in use when the package is imported. In HDRP the demo camera adds a fixed exposure and a simple sky while the demo plays |
| Input System | The components do not depend on any input system. Only the keys of the included demo and of the debug tools use the New Input System (Input System package) |
| Physics | 3D physics only: walls are found with raycasts against 3D colliders. 2D physics (Collider2D) is not supported. The player needs no collider (check points can come from Height) |

---

## How to Use

### Setup (add two components)

1. Add **PerceptionSensor** to the enemy (Add Component > LogicDevSupport > SmartPerception > Perception Sensor)
2. (Recommended) Create an empty child at head height that faces forward, and assign it to **Eyes**. The view is measured from it (empty = the enemy's own Transform)
3. Add **PerceptionTarget** to the player (Add Component > LogicDevSupport > SmartPerception > Perception Target). With an empty **CheckPoints** list, three points (feet, chest, head) are taken from the player's colliders, or from **Height** when it has none
4. Press Play. The enemy's state follows what it sees: read `State`, or use the events

Every setting works with its default value. Walls are the colliders of **ObstacleLayers** (all layers by default). The bodies of other sensors (an object with a PerceptionSensor and its children) do not count as walls unless **OtherSensorsBlock** is on.

### Reading the Sensor from Your AI

```csharp
using LogicDev.Perception;

PerceptionSensor sensor = GetComponent<PerceptionSensor>();

if (sensor.State == PerceptionAwareness.Alert && sensor.CanSeeFocus)
{
    // chase sensor.FocusTarget
}
else if (sensor.State != PerceptionAwareness.Unaware && sensor.TryGetLastKnownPosition(out Vector3 pos))
{
    // go to pos and look around (where it was last seen, touched or heard)
}

sensor.AwarenessChanged += (previous, next) => Debug.Log(previous + " -> " + next);
sensor.TargetSpotted += target => Debug.Log("Spotted " + target.name);
```

The same events can be wired up in the Inspector (**OnAwarenessChanged**, **OnSuspicious**, **OnAlert**, **OnCalm**, **OnTargetSpotted**, ...).

`State` and `Awareness` are the highest meter among all targets. `FocusTarget` is the one target the sensor follows (see "Several Players"); memory, alert calls and the debug display use it.

### Noises (one line)

```csharp
// Footsteps, a gunshot, a door: who made it (a PerceptionTarget) raises the guards' meter of that target.
PerceptionNoise.Emit(transform.position, 8f, myTarget);

// A thrown stone (no source): the guards only report it (OnNoiseHeard, LastNoise).
PerceptionNoise.Emit(stone.position, 10f);
```

Every enabled sensor within `range` x its **HearingSensitivity** hears it. The strength is 1 at the noise and 0 at that distance (straight line), and is multiplied by **WallDamping** for each wall between the noise and the Eyes. A noise heard at full strength adds **NoiseAwareness** to the meter, a weaker one proportionally less, and noises alone never raise the meter above **NoiseMaxAwareness** (0.7 by default: Suspicious, but not Alert). The guard remembers where it heard the noise (`TryGetLastKnownPosition`).

`PerceptionNoise.Emit` can also be called from inside a noise event (`OnNoiseHeard` / `NoiseHeard`), for example to relay a noise.

### Touch

A target closer than **TouchRadius** (1 m by default) to the enemy's body (the line from its pivot up to the Eyes) sets the meter to at least **TouchAwareness** (1 = Alert at once), from any direction, even when it cannot be seen. Set TouchRadius to 0 to turn it off.

### Distance Curve

By default the meter rises at full speed right in front of the Eyes and at **FarFactor** speed at **ViewDistance** (a straight line in between). Turn on **UseDistanceCurve** to draw the factor yourself in **DistanceCurve** instead (FarFactor is then greyed out and not used):

- Horizontal axis: distance / ViewDistance (0 = at the Eyes, 1 = at ViewDistance)
- Vertical axis: the speed factor (1 = normal, 0 = the meter does not rise, above 1 = faster)
- The default curve is the same straight line as FarFactor 0.4, so turning it on changes nothing until you edit it

**Example: "Alert at once within 2 m"** (ViewDistance 15, RiseRate 0.7): points (0, 30), (0.13, 30), (0.14, 1), (1, 0.4). Right-click the points and choose **Both Tangents > Linear** so the curve is made of straight lines. Within 2 m (0.13 x 15) the meter rises 30 times faster (21 per second): Alert in about 3 frames at 60 fps. Farther away it works as before (in our test, 1.75 s to Alert at 6 m).

### Alert Call

Turn on **AlertCall** to let an enemy call its allies when it becomes Alert. The call is an ordinary noise made at its Eyes (with `PerceptionNoise.Emit`), whose source is the target it found:

- The allies within **AlertCallRange** (x their HearingSensitivity, through walls with WallDamping) raise their meter of that target, never above their NoiseMaxAwareness
- **The call tells where the caller last saw (or touched) the target**, not where the caller stands (when it has no memory of the target, its own Eyes). The allies remember that position: `TryGetLastKnownPosition` gives it, so your AI can send them there. How far the call is heard and the walls are still measured from the caller's Eyes
- A sensor never hears its own call
- **AlertCallInterval**: seconds between two calls while the enemy stays Alert (0 = only once each time it becomes Alert)
- **AlertCallOnlyWhileSeen** (on by default): the calls are repeated only while the target is in sight, so an enemy that has lost the target stops calling. The first call when it becomes Alert is made either way
- **Calls do not spread**: because a call raises the meter only up to NoiseMaxAwareness, the allies become Suspicious, not Alert, so they never call in turn. They become Alert only when they see or touch the player themselves (if you set NoiseMaxAwareness to AlertThreshold or more, calls do make them Alert and they call in turn)
- Event: **OnAlertCall** / `AlertCalled` (the target found). `LastAlertCallTime` and `LastAlertCallHeard` (how many sensors heard the last call)

To send your own call with a position to report, use `PerceptionNoise.Emit(position, range, source, reportedPosition)`: the listeners remember `reportedPosition` (`PerceptionHeardNoise.ReportedPosition`), while the range and the walls are measured from `position`.

### Several Players (choosing the target to follow)

With several players (co-op), each sensor follows one of them: `FocusTarget`. Only the targets it is suspicious of (meter at **SuspiciousThreshold** or more) are chosen; when there is none, the target with the highest meter is used (also kept for **FocusHoldTime**, so two low meters overtaking each other do not switch it back and forth). **FocusMode** chooses the order:

- **Nearest** (default): the target nearest to the Eyes
- **Priority**: the highest **Priority** (a setting of PerceptionTarget), then the nearest. For example, give the player carrying the objective a higher priority
- **Awareness**: the highest meter, then the nearest (when two targets are both at 100%, the nearer one)

Rules common to the three modes:

1. **A target in sight comes first**: a target the sensor sees (or touches) comes before one it has only heard or remembers. While the current target is in sight, the sensor switches only to another target in sight, so footsteps behind it do not make it turn away. The meter of the target making the noise still rises and its position is remembered: once the current target is lost, the sensor turns to it
2. **A touch switches at once**: a target that touches the sensor becomes the focus immediately (before the hold time and the priority), and stays so while it touches
3. **No switching back and forth**: after a switch, the focus is kept for **FocusHoldTime** seconds (2 by default), and then changes only to a clearly better target: **FocusSwitchMargin** (0.1) = 10% nearer, or a meter 0.1 higher in the Awareness mode. A higher priority, a target in sight after the focus was lost, and a touch win without the margin

Event: **OnFocusChanged** / `FocusChanged` (previous target, new target; either may be null). `LastFocusChangeTime`.

**Networking**: the sensors run on the machine they are in and know nothing about networks. In an online game, run the sensors on one machine (usually the server or the host) and send the results (state, focus target) to the other players from your game code. Synchronising them is up to your game.

### Many Enemies (spreading the work)

The vision test of a sensor (its raycasts) runs every **UpdateInterval** seconds (0.1 by default; 0 = every frame). The sensors take turns, and `PerceptionScheduler.MaxSensorsPerFrame` (32 by default, 0 = no limit) caps how many test in one frame. Between two tests the meter, touch and noises still update every frame with the result of the last test.

When guards gather, their own bodies would otherwise hide the player from the guards behind them. With **OtherSensorsBlock** off (the default), the colliders of other sensors (an object with a PerceptionSensor and its children) neither block the view nor muffle noises. Turn it on if your enemies should hide each other.

### Debug Display, Tuning Panel and Keeping Play Mode Values

The following tools are in the `Debug` folder. **They exist only in the Editor and in Development Builds: in a release (non-development) build their code is compiled out** (the two components stay as empty placeholders so that scenes containing them load without a missing-script warning). The sensor does not depend on them, so you can also delete the whole `Debug` folder.

**Adding and removing them**: **Tools > LogicDevSupport > SmartPerception > Add Debug Tools to Scene** creates one GameObject ("Perception Debug Tools") with both components; **Remove Debug Tools from Scene** deletes it (both can be undone with Ctrl+Z).

- **Perception Debug View**: for every sensor, the view (field of view, central vision, distance), a line to each check point (green = visible, red = blocked, up to the wall), the meter above the sensor (ticks = the thresholds), the last seen position, the last heard position, a dashed line to the last known position while Suspicious / Alert and not seeing the target (where your AI would go to look), the touch radius, each noise as a ring with a line to every sensor that heard it, each alert call as a red ring of its full range, and a status table in the top-left corner (one row per sensor). **G** shows / hides it
- **Perception Tuning Panel**: a small panel in the bottom-right corner to change the main settings of one sensor at a time while playing (`<` `>` choose the sensor), and to switch the parts of the debug view. **T** shows / hides it. Drag the title bar to move it
- **UI mode**: while the panel is on screen, the mouse cursor is shown and unlocked so you can click it. Your own camera script can stop mouse look during UI mode with one line: `if (PerceptionTuningPanel.IsUIMode) return;` (namespace `LogicDev.Perception`). In a release build `IsUIMode` is always false
- **Keeping Play Mode values (Editor only)**: press **Record values** (on the panel, or at the bottom of the sensor's Inspector during Play). When Play Mode ends, a window lists the settings that differ from the scene; tick the ones to keep and press **Apply** (Ctrl+Z reverts it)

**Scene-view gizmos (Editor)**: the view, the touch radius, the sight lines and the last seen position. **ShowGizmos** chooses when: `Selected` (default), `Always` or `Off`.

---

## Parameter Reference

### PerceptionSensor - Vision

| Parameter | Default | Description |
|-----------|---------|-------------|
| Eyes | (none) | Where the sensor looks from and its forward (empty = this object) |
| ViewDistance | 15 | How far the sensor can see (world units) |
| ViewAngle | 60 | Half of the horizontal field of view in degrees (60 = 120° wide, 180 = all around) |
| VerticalViewAngle | 45 | Half of the vertical field of view in degrees (measured from the Eyes' forward) |
| CentralAngle | 25 | Central vision in degrees left / right: the meter rises at full speed here |
| PeripheralFactor | 0.3 | Speed of the meter at the edge of the view, relative to central vision |
| FarFactor | 0.4 | Speed of the meter at ViewDistance, relative to right in front (straight line in between) |
| UseDistanceCurve | Off | Use DistanceCurve instead of FarFactor |
| DistanceCurve | line 1 -> 0.4 | Speed factor by distance / ViewDistance (see "Distance Curve") |
| RequiredVisiblePoints | 1 | How many check points must be visible to count the target as seen |
| ObstacleLayers | all layers | Layers that block the view and muffle noises (the sensor's and the target's own colliders and triggers never block) |
| OtherSensorsBlock | Off | Whether the bodies of other sensors (an object with a PerceptionSensor and its children) block the view and muffle noises |

### PerceptionSensor - Awareness and Memory

| Parameter | Default | Description |
|-----------|---------|-------------|
| RiseRate | 0.7 | Meter rise per second while a target is fully seen in central vision, right in front (1 = Alert in one second) |
| DecayRate | 0.2 | Meter fall per second once nothing has been noticed for DecayDelay seconds |
| DecayDelay | 2 | Seconds the meter holds after the target was last noticed (seen, heard or touched) |
| SuspiciousThreshold | 0.3 | Meter value at which the state becomes Suspicious |
| AlertThreshold | 1 | Meter value at which the state becomes Alert |
| CalmMargin | 0.1 | The state goes back down only when the meter falls this far below the threshold (or reaches 0) |
| ForgetTime | 10 | Seconds the sensor remembers the last seen / heard position after losing the target |

### PerceptionSensor - Focus (several players)

| Parameter | Default | Description |
|-----------|---------|-------------|
| FocusMode | Nearest | How FocusTarget is chosen among the targets the sensor is suspicious of: Nearest, Priority or Awareness |
| FocusHoldTime | 2 | Seconds the focus is kept after a switch (a touch and a lost focus target do not wait) |
| FocusSwitchMargin | 0.1 | How much better another target must be to take the focus: 0.1 = 10% nearer, or a meter 0.1 higher (Awareness) |

### PerceptionSensor - Hearing, Alert Call, Touch and Performance

| Parameter | Default | Description |
|-----------|---------|-------------|
| HearingSensitivity | 1 | A noise is heard up to its range x this value (0 = deaf) |
| WallDamping | 0.5 | Strength left after each wall between the noise and the Eyes |
| NoiseAwareness | 0.5 | Meter added by a noise heard at full strength |
| NoiseMaxAwareness | 0.7 | Noises (and alert calls) alone never raise the meter above this value |
| AlertCall | Off | Call out to the other sensors when this sensor becomes Alert |
| AlertCallRange | 12 | How far the call can be heard (world units) |
| AlertCallInterval | 0 | Seconds between two calls while Alert (0 = once each time it becomes Alert) |
| AlertCallOnlyWhileSeen | On | Repeat the call only while the target is in sight |
| TouchRadius | 1 | A target closer than this to the body is noticed at once (0 = off) |
| TouchAwareness | 1 | Meter value a touch sets at least |
| UpdateInterval | 0.1 | Seconds between two vision tests (0 = every frame) |
| ShowGizmos | Selected | When the Scene-view gizmos are drawn (Off / Selected / Always) |

### PerceptionSensor - Events

| Event (Inspector / C#) | When |
|-------|------|
| OnAwarenessChanged / AwarenessChanged | The state changes (previous, new) |
| OnSuspicious, OnAlert, OnCalm | The state becomes Suspicious / Alert / Unaware |
| OnTargetSpotted / TargetSpotted | A target comes into sight |
| OnTargetLost / TargetLost | A target goes out of sight |
| OnTargetForgotten / TargetForgotten | The sensor forgets a target (ForgetTime after it was last seen or touched) |
| OnTargetTouched / TargetTouched | A target comes within TouchRadius |
| OnNoiseHeard / NoiseHeard | The sensor hears a noise (position, reported position, range, source, strength, walls, time) |
| OnAlertCall / AlertCalled | The sensor has just called out (the target it found) |
| OnFocusChanged / FocusChanged | FocusTarget changes (previous, new; either may be null) |

### PerceptionSensor - Properties and Methods

| Member | Description |
|--------|-------------|
| State, Awareness | Current state and meter (0..1): the highest meter among all targets |
| FocusTarget, Focus, CanSeeFocus | The target the sensor follows (see "Several Players"), what the sensor knows about it (PerceptionRecord), whether it is in sight |
| LastFocusChangeTime | When FocusTarget last changed |
| Records, GetRecord(target), CanSee(target) | Everything the sensor knows, one PerceptionRecord per target |
| HasMemory, LastSeenPosition, LastSeenTime | Memory of the focus target |
| TryGetLastKnownPosition(out pos) | The newest position of the focus target: last seen, touched or heard |
| ForgetTarget(target), ResetAwareness() | Forget one target / set every meter to 0 and forget everything |
| HasHeardNoise, LastNoise, GetNoiseStrength(...), HearNoise(...) | The last noise heard; how strongly a noise would be heard; make this sensor hear a noise |
| LastAlertCallTime, LastAlertCallHeard | When the last alert call was made and how many sensors heard it |
| GetTouchDistance(target) | Distance from the body to the nearest check point |
| DistanceFactorAt(d), AngleFactorAt(yaw), TestPoint(...) | The speed factors and the vision test of one point |
| Sense(deltaTime) | One step by hand (disable the component and call it with your own time step) |
| static All | Every enabled sensor |
| static IsSensorBody(transform) | Whether the object or one of its parents has a PerceptionSensor |

### PerceptionTarget

| Parameter | Default | Description |
|-----------|---------|-------------|
| CheckPoints | (empty) | Points the sensors test. Empty = three points (feet, chest, head) from the colliders, or from Height |
| Height | 1.8 | Body height for the automatic points when there is no collider (the pivot is the feet) |
| Visibility | 1 | How easy the target is to notice (0.5 = the meter rises half as fast, 0 = never noticed) |
| Detectable | On | Off = the sensors ignore this target (for example in a hiding place) |
| Priority | 0 | Used by sensors whose FocusMode is Priority: a higher value is followed first |

### Other Types

| Type | Description |
|------|-------------|
| PerceptionNoise.Emit(position, range, source) | Makes a noise; returns how many sensors heard it. `Emitted` is raised for every noise |
| PerceptionNoise.Emit(position, range, source, reportedPosition) | The same, telling the listeners another position to remember (see "Alert Call") |
| PerceptionHeardNoise | A noise heard: Position, ReportedPosition, Range, Source, Strength, Obstacles, Time |
| PerceptionScheduler.MaxSensorsPerFrame | Most vision tests in one frame (32 by default, 0 = no limit) |
| PerceptionRecord | Per target: Awareness, IsSeen, VisibleFraction, Exposure, LastSeenTime, Points, HasMemory, LastSeenPosition, HasHeard, LastHeardPosition, LastHeardTime, IsTouching |
| PerceptionAwareness | Unaware, Suspicious, Alert |
| PerceptionFocusMode | Nearest, Priority, Awareness |

---

## Performance

Measured in the Unity Editor (Unity 2022.3 and 6.3, Intel Core i7-14700F) with 400 sensors watching one target with three check points:

| Setting | Vision tests per frame | Time per frame |
|---------|------------------------|----------------|
| UpdateInterval 0 (every frame), no cap | 400 | about 2.0 ms |
| UpdateInterval 0.1, MaxSensorsPerFrame 32 (defaults) | 32 | about 0.6 ms |

With the defaults each sensor still tests about 10 times a second, and the meter moves every frame. The time grows with the number of sensors x targets x check points.

---

## Known Limitations

- 3D physics only. 2D games (Collider2D, Physics2D) are not supported
- Each check point is one ray from the Eyes. A point just behind the edge of a wall counts as hidden even if a little of the body around it would be visible
- The field of view is two angles (left / right and up / down separately), not a round cone
- Noises go through walls in a straight line (weaker per wall); they do not go around corners
- When the sensor becomes Alert outside its own update (for example by a noise or a call it hears), its own alert call is made at its next update
- Networking is not included: in an online game, run the sensors on one machine and synchronise the results in your game code (see "Several Players")

---

## Demo Scene

Open `Demo_SmartPerception` (`Assets/LogicDevSupport/SmartPerception/Demo/`) and press Play. Three guards patrol a walled area; you are the blue capsule at the bottom, in a small hiding place, with a second player (teal) next to you.

- **Controls**: **WASD / arrow keys** move (footsteps can be heard), hold **Shift** to sneak (silent, harder to see), **Space** claps (a loud noise), **Tab** switches the player you move (the other one stays where it is), **R** restarts (every guard forgets you and the players go back to the start), **M** shows / hides the minimap, **G** shows / hides the debug display, **T** shows / hides the tuning panel
- **Guards**: a guard patrols its route (a thin line on the ground goes to the waypoint it walks to). Its colour shows its state (grey = Unaware, yellow = Suspicious, red = Alert). When Suspicious or Alert it stops and turns toward where it last saw or heard you
- **Chase and search**: an Alert guard chases the player it follows; when it loses that player, it walks to where it last saw them, looks around, and goes back to its patrol
- **Noises**: walking makes footsteps; a guard that hears them behind it becomes Suspicious, walks to where it heard them, looks around, and goes back to its patrol. A small noise that does not make it Suspicious only makes it turn toward the noise
- **Alert call**: the guards have AlertCall on (range 18, every 2 s while Alert and seeing you). When one guard finds you, the guards near it hear the call (a red ring on the debug display), walk to where the caller last saw you, look around, and go back to their patrol
- **Two players**: move one player into a guard's view, then switch with Tab and bring the other one closer, or bump into the guard with it: the guard follows the nearer player in sight, and turns to a player that touches it at once
- **Minimap**: the radar in the top-right corner (M) shows the walls, the players (arrows) and the guards (a dot in their state colour with their view cone). `PerceptionDemoMinimap` uses only the public API (`PerceptionSensor.All`, `State`, `GetViewFrame`, `ViewAngle`, `ViewDistance`, `PerceptionTarget.All`) and can be used as a sample for your own minimap
- **Hide behind cover**: the low walls hide only your legs (the guards still see part of you, more slowly); the tall walls hide you completely
- **Touch**: bump into a guard from behind and it notices you at once

> Note: patrolling, chasing, walking to noises and calls, looking around and stepping aside for other guards are done by demo-only scripts (`PerceptionDemoPatrol`, `PerceptionDemoChase` and the others in `Demo/Scripts/`), as a sample of what your own AI could do with the sensor. They are not part of the product: delete them and the sensors still work. **They move in a straight line and do not find paths**: a guard that cannot get around a wall stops in front of it, looks around and goes back to its patrol; it then walks its route for a few seconds (`ReturnTime`, 6 s) before a noise can send it out again, so it does not keep standing at the wall while a noise behind it continues. In your game, move the enemies with a navigation system such as Unity's NavMesh and use the sensor's positions (`TryGetLastKnownPosition` and so on) as the destinations.

> Note: in HDRP, the demo camera (`PerceptionDemoExposure`) adds a fixed exposure and a simple gradient sky while the demo plays, so that the demo looks as in the other render pipelines. It does nothing in the Built-in Render Pipeline and URP, and does not change your own scenes.

> Note: The demo scene uses primitive shapes only. A demo using Starter Assets is available on the Asset Store page (not included in this package).

---

## Before You Import (Important)

If Smart Perception Lite (the free version) is in your project, delete its folder (`Assets/LogicDevSupport/SmartPerception/`) before importing this package. Both versions use the same folder, the same class names and the same file identities, so importing on top of Lite can leave files of the wrong version in the folder. Your scenes keep working after the change: Pro has every class, setting and event of Lite with the same names.

If you have edited any files inside that folder (such as the demo scene), please back them up before deleting it.

---

## Upgrade Path

To try the free version first, Smart Perception Lite is available here:

https://assetstore.unity.com/packages/slug/426870

---

If you find this asset useful, please consider leaving a review - it really helps!

Leave a review here (via the Reviews tab):

https://assetstore.unity.com/packages/slug/426874
