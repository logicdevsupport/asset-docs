# Smart Perception Lite - Vision and Awareness for Enemies

A "notice" component for enemies and NPCs (free version). Add `PerceptionSensor` to an enemy and `PerceptionTarget` to the player: the enemy sees (field of view, distance, walls, several points of the body) and raises an awareness meter (Unaware -> Suspicious -> Alert). It tells your AI what it noticed through properties and events. Moving the enemy (chase, patrol, attack) is not part of this package: your AI reads the sensor and decides what to do.

---

## Features

- **Vision with partial cover** - Each target has several check points (feet, chest, head by default). A player behind a low wall is still partly visible; the meter rises with how much of the body is seen
- **Central and peripheral vision, distance** - Inside the central angle the meter rises at full speed, toward the edge of the view and toward the view distance it rises more slowly
- **Awareness meter with three states** - Unaware -> Suspicious -> Alert, with a hold time, a decay and a margin so the state does not flicker at a threshold
- **Events** - C# events and UnityEvents for state changes and for a target spotted / lost
- **Scene-view gizmos** - The view (field of view, central vision, distance), a line to each check point (green = visible, red = blocked) and the state with the meter
- **Groups of enemies** - The bodies of other enemies do not block the view by default, so a group of guards does not hide the player from itself

---

## Requirements

| Item | Details |
|------|---------|
| Unity Version | Unity 2022.3 LTS / Unity 6.3 LTS |
| Render Pipeline | Built-in Render Pipeline, URP and HDRP (tested: Unity 2022.3 URP / Built-in / HDRP, Unity 6.3 URP / Built-in / HDRP). The components do not use any render-pipeline-specific API. The materials of the included demo scene are switched to the standard shader of the render pipeline in use when the package is imported. In HDRP the demo camera adds a fixed exposure and a simple sky while the demo plays |
| Input System | The components do not depend on any input system. Only the keys of the included demo use the New Input System (Input System package) |
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
else if (sensor.State == PerceptionAwareness.Suspicious)
{
    // stop and look toward sensor.FocusTarget
}

sensor.AwarenessChanged += (previous, next) => Debug.Log(previous + " -> " + next);
sensor.TargetSpotted += target => Debug.Log("Spotted " + target.name);
```

The same events can be wired up in the Inspector (**OnAwarenessChanged**, **OnSuspicious**, **OnAlert**, **OnCalm**, **OnTargetSpotted**, **OnTargetLost**).

`State` and `Awareness` are the highest meter among all targets. `FocusTarget` is the target with that meter.

### How the Meter Rises

- A point is visible when it is within **ViewDistance**, within **ViewAngle** left / right and **VerticalViewAngle** up / down (measured from the Eyes' forward), and no collider of **ObstacleLayers** is in between
- The target is seen when at least **RequiredVisiblePoints** of its points are visible
- While seen, the meter rises by **RiseRate** x exposure x the target's **Visibility** per second. The exposure is the average over the points of (angle factor x distance factor): the angle factor is 1 inside **CentralAngle** and **PeripheralFactor** at ViewAngle, the distance factor is 1 at the Eyes and **FarFactor** at ViewDistance (straight lines in between); a hidden point counts as 0
- When the target is out of sight, the meter holds for **DecayDelay** seconds, then falls by **DecayRate** per second
- The state goes up exactly at **SuspiciousThreshold** / **AlertThreshold**, and goes down only when the meter is **CalmMargin** below the threshold (always Unaware at 0)

### Scene-view Gizmos

In the Scene view, the sensor draws its view (field of view, central vision, distance) and, in Play Mode, a line to each check point (green = visible, red = blocked, up to the wall) and its state with the meter. **ShowGizmos** chooses when: `Selected` (default), `Always` or `Off`.

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
| RequiredVisiblePoints | 1 | How many check points must be visible to count the target as seen |
| ObstacleLayers | all layers | Layers that block the view (the sensor's and the target's own colliders and triggers never block) |
| OtherSensorsBlock | Off | Whether the bodies of other sensors (an object with a PerceptionSensor and its children) block the view |

### PerceptionSensor - Awareness

| Parameter | Default | Description |
|-----------|---------|-------------|
| RiseRate | 0.7 | Meter rise per second while a target is fully seen in central vision, right in front (1 = Alert in one second) |
| DecayRate | 0.2 | Meter fall per second once the target has been out of sight for DecayDelay seconds |
| DecayDelay | 2 | Seconds the meter holds after the target was last seen |
| SuspiciousThreshold | 0.3 | Meter value at which the state becomes Suspicious |
| AlertThreshold | 1 | Meter value at which the state becomes Alert |
| CalmMargin | 0.1 | The state goes back down only when the meter falls this far below the threshold (or reaches 0) |
| ShowGizmos | Selected | When the Scene-view gizmos are drawn (Off / Selected / Always) |

### PerceptionSensor - Events

| Event (Inspector / C#) | When |
|-------|------|
| OnAwarenessChanged / AwarenessChanged | The state changes (previous, new) |
| OnSuspicious, OnAlert, OnCalm | The state becomes Suspicious / Alert / Unaware |
| OnTargetSpotted / TargetSpotted | A target comes into sight |
| OnTargetLost / TargetLost | A target goes out of sight |

### PerceptionSensor - Properties and Methods

| Member | Description |
|--------|-------------|
| State, Awareness | Current state and meter (0..1): the highest meter among all targets |
| FocusTarget, Focus, CanSeeFocus | The target with the highest meter, what the sensor knows about it (PerceptionRecord), whether it is in sight |
| Records, GetRecord(target), CanSee(target) | Everything the sensor knows, one PerceptionRecord per target |
| ResetAwareness() | Set every meter to 0 (the state becomes Unaware) |
| DistanceFactorAt(d), AngleFactorAt(yaw), TestPoint(...) | The speed factors and the vision test of one point |
| GetViewFrame(...), NextState(...) | The view origin and directions; the state a meter value leads to |
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

### Other Types

| Type | Description |
|------|-------------|
| PerceptionRecord | Per target: Awareness, IsSeen, VisibleFraction, Exposure, LastSeenTime, TimeSinceSeen, Points |
| PerceptionAwareness | Unaware, Suspicious, Alert |

---

## Known Limitations

- 3D physics only. 2D games (Collider2D, Physics2D) are not supported
- Each check point is one ray from the Eyes. A point just behind the edge of a wall counts as hidden even if a little of the body around it would be visible
- The field of view is two angles (left / right and up / down separately), not a round cone
- Every sensor runs its vision test every frame (the cost grows with the number of sensors x targets x check points)

---

## Demo Scene

Open `Demo_SmartPerception` (`Assets/LogicDevSupport/SmartPerception/Demo/`) and press Play. Three guards patrol a walled area; you are the blue capsule at the bottom, in a small hiding place.

- **Controls**: **WASD / arrow keys** move, hold **Shift** to sneak (slower, harder to see), **R** restarts (every guard calms down and the player goes back to the start)
- **Guards**: a guard patrols its route (a thin line on the ground goes to the waypoint it walks to). Its colour shows its state (grey = Unaware, yellow = Suspicious, red = Alert). When Suspicious or Alert it stops and turns toward you while it sees you; when it calms down it walks on
- **Hide behind cover**: the low walls hide only your legs (the guards still see part of you, more slowly); the tall walls hide you completely
- **View**: the guards' views and sight lines are drawn in the Scene view (their ShowGizmos is set to Always)

> Note: patrolling, stopping and turning are done by demo-only scripts (`PerceptionDemoPatrol` and the others in `Demo/Scripts/`), as a sample of what your own AI could do with the sensor. They are not part of the product: delete them and the sensors still work. **They move in a straight line and do not find paths.** In your game, move the enemies with a navigation system such as Unity's NavMesh.

> Note: in HDRP, the demo camera (`PerceptionDemoExposure`) adds a fixed exposure and a simple gradient sky while the demo plays, so that the demo looks as in the other render pipelines. It does nothing in the Built-in Render Pipeline and URP, and does not change your own scenes.

> Note: The demo scene uses primitive shapes only. A demo using Starter Assets is available on the Asset Store page (not included in this package).

---

## Upgrade to Smart Perception Pro

Smart Perception Pro has everything in Lite (the same classes, settings and events, with the same names) and adds:

- **Memory** - The last seen position and time, kept for a set time after the target is lost
- **Hearing** - One line makes a noise (`PerceptionNoise.Emit`): weaker with the distance and through walls
- **Touch** - A target that bumps into the enemy is noticed at once, from any direction
- **Alert call** - An Alert enemy calls its allies, who come to look where it last saw the player
- **Distance curve** - Draw how fast the meter rises by distance (for example "Alert at once within 2 m")
- **Several players** - Each enemy chooses which player to follow (nearest, priority or meter), without switching back and forth
- **Many enemies** - The vision tests are spread over frames
- **Debug display and tuning panel** - The view, meters, noises and calls in the Game view, and a panel to tune the sensors while playing (development only)
- **A larger demo** - Chase and search, footsteps and claps, alert calls, two players and a minimap sample

https://assetstore.unity.com/packages/slug/426874

**Moving from Lite to Pro**: delete the Lite folder (`Assets/LogicDevSupport/SmartPerception/`) before importing Pro. Both versions use the same folder, the same class names and the same file identities, so importing Pro on top of Lite can leave files of the wrong version in the folder. If you have edited any files inside that folder (such as the demo scene), back them up first. Your scenes keep working after the change.

---

If you find this asset useful, please consider leaving a review - it really helps!

Leave a review here (via the Reviews tab):

https://assetstore.unity.com/packages/slug/426870
