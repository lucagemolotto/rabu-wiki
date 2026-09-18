# Robo-AbU: A DSL for Event-Driven Robotic Systems

Robo-AbU is a variant of `goAbU`, the original Golang implementation of the
[AbU calculus](https://doi.org/10.1016/j.tcs.2023.113841).
It provides a way to program robots via Event-Condition-Action (ECA) rules, building event-driven and reactive robotic systems.
AbU also abstracts away inter-agent communication by using *remote tasks*, which are actions acting on *remote* nodes, even though the rule is triggered locally.

The reference implementation lives in
[AbU-ROS](https://github.com/Autonomous-Systems-Laboratory-UNIUD/AbU-ROS) (Go module `aburos`).

## Syntax

AbU rules take the following structure:

```
rule <RuleName>
on <event>
for / for all <boolean condition>
do <assignments>
```

meaning that rule `<RuleName>` is triggered by the change of value of attribute `<event>`.
If we used the keyword `for` and the boolean condition holds locally, it will result in a series of assignments, to local attributes.
If, instead, we used keyword `for all` the assignments will be done on all remote nodes that satisfy the boolean condition.
We call *task* the couple made up of the condition and the assignments (actions).

| Clause | Meaning |
| --- | --- |
| `on a b c` | The rule fires when **any** of the listed attributes changes value. Separate attributes with spaces. |
| `for <cond>` | **Local task.** `<cond>` is evaluated on this node; the actions assign this node's attributes. |
| `for all <cond>` | **Remote task.** `<cond>` is evaluated on *every* node; the actions run on each node where it holds. |
| `do <assignments>` | Comma-separated `attribute = expression` assignments. |

In a `for all` rule, the prefix `ext.` selects the **remote** node's attribute; an unprefixed name
is read from the node that fired the rule. So in

```
rule detectionRule on detection_alarm for all true == ext.detection_picker
do ext.detectionfound = true, ext.detectionfound_lat = position_lat
```

`ext.detection_picker` and `ext.detectionfound_lat` belong to the receiver, while `position_lat` is
the sender's own position. Nodes are never named — this is what makes the interaction
*attribute-based*.

The evaluation of tasks depends on whether we are using *eager* or *lazy* evaluation; see
[lazyevaluation](contributing.md#lazyevaluation) for details.

### Expressions

Conditions and right-hand sides are Grule expressions: the usual arithmetic
(`+ - * /`), comparison (`== != < <= > >=`) and boolean (`&& || !`) operators, over the four
attribute types (`Text`, `Bool`, `Integer`, `Float`). String literals use double quotes, which must
be escaped when a rule is written inside a YAML file.

Robo-AbU adds two built-in functions on top of Grule's own
([`builtinFunctions.go`](https://github.com/Autonomous-Systems-Laboratory-UNIUD/AbU-ROS/blob/main/builtinFunctions.go)):

| Function | Signature | Description |
| --- | --- | --- |
| `AbsInt` | `AbsInt(int64) int64` | Absolute value of an integer |
| `Sqrt` | `Sqrt(float64) float64` | Square root |

!!! warning "Integer and float attributes do not mix"
    `Integer` and `Float` are distinct types. `position_lat` is an `Integer` (WGS84 × 10⁷) while
    `altitude` is a `Float`; comparing or assigning across the two is a type error at rule-parse
    time, so write `altitude > 3.0`, not `altitude > 3`.

## Getting started

To start writing Robo-AbU code you need four things:

1. A **`RoboAbuAgent`** — the transaction manager, which carries remote tasks between nodes. See
   [agent](contributing.md#agent) for the available implementations.
2. A **`ROSresources`** — the I/O manager, the gateway to the vehicle's sensors and actuators. Most
   implementations wrap a **`Vehicle`**; see [rosresources](contributing.md#rosresources).
3. A **slice of AbU rules**, one rule per string.
4. A **`RosExecuter`**, built from the three above.

```go
func NewRosExecuter(
    mem        ros_resources.ROSresources, // the I/O manager
    rules      []string,                   // one rule per string
    agt        agent.RoboAbuAgent,         // the transaction manager
    localname  string,                     // this node's identifier
    remotename string,                     // shared namespace for remote communication
    eval       string,                     // "lazy" or "eager"
    invariants ...string,                  // optional global invariants
) (*RosExecuter, error)
```

`localname` identifies the node; `remotename` is the namespace all cooperating nodes share
(`"aburos"` by convention). `eval` selects the update-evaluation strategy — see
[eagerevaluation](contributing.md#eagerevaluation) and [lazyevaluation](contributing.md#lazyevaluation).
Any trailing `invariants` are boolean expressions re-checked after every update; an update that
would break one is discarded.

Then all you need is calling `Exec` and/or `Input` on the `RosExecuter`.

| Method | Description |
| --- | --- |
| `Exec()` | Runs one execution step: picks an update from the pool, applies it, and propagates the tasks it triggers. Call it in a loop. |
| `Input(actions string) error` | Injects an external assignment (e.g. `` `foo = "abc"` ``), as if the environment had changed it. This is how a system is bootstrapped or driven from outside. |
| `TakeState() (memory.Resources, []update.Update)` | Returns a snapshot of the node's memory and its pending update pool. |
| `AddRules(rules ...string) error` | Adds rules to a running executer. |
| `StartAgent() error` | Starts the underlying transaction manager (done for you by `NewRosExecuter`). |

## Example

In this example we will spawn two Copters, `cop1` and `cop2`, which will be controlled by AbU rules
and coordinate with each other. `cop1` patrols; when its object-detection sensor fires, `cop2` —
which advertises itself as a picker — takes off and flies to the reported coordinates.

We start by making the two vehicles and their resources:

```go
cop1, err := vehicles.NewCopterGoROSetta("copter1", "", nil, nil)
if err != nil {
    fmt.Printf("failed to create vehicle: %v\n", err)
    return
}
mem := ros_resources.NewCopterResource(cop1)
mem.Text["foo"] = "aaa"
mem.Bool["detection_alarm"] = false
mem.Float["base_alt"] = 0.0

cop2, err := vehicles.NewCopterGoROSetta("copter2", "", nil, nil)
if err != nil {
    fmt.Printf("failed to create vehicle: %v\n", err)
    return
}
mem_cop := ros_resources.NewCopterResource(cop2)
mem_cop.Text["foo"] = "aaa"
mem_cop.Bool["detection_picker"] = true
mem_cop.Integer["detectionfound_lat"] = 0
mem_cop.Integer["detectionfound_lon"] = 0
mem_cop.Bool["detectionfound"] = false
mem_cop.Float["base_alt"] = 0.0
```

We declared the vehicles, then the resources, and added some attributes which will come in handy for
the logic of our system. `foo` is an ordinary `Text` attribute with no meaning to the vehicle — we
use it purely as a trigger we can drive from `Input`. `detectionfound_lat` and `detectionfound_lon`
are declared as `Integer` because they will be assigned from `position_lat` / `position_lon`, which
are WGS84 degrees × 10⁷.

Now we declare the rules shared by both copters. `initRule` is the starting trigger, putting a
vehicle in `GUIDED` mode. Then we arm it via `armRule` and take off via `takeoffRule`, recording the
ground altitude in `base_alt` on the way.

```go
initRule    := `rule InitRule on foo for "abc" == foo do set_mode = "GUIDED"`
armRule     := `rule ArmRule on mode for mode == "GUIDED" do set_arm = true`
takeoffRule := `rule TakeOffRule on arm for arm do base_alt = altitude, take_off = 5.0`
```

Now comes the important part. `moveRule` waits for the vehicle to have taken off to a good enough
altitude and moves it in a Forward-Right-Down (FRD) frame by the vector (-5 m, 0 m, 2 m).
`alarmRule` lifts the raw sensor reading into an attribute, and `globalRule` — the only `for all`
rule here — broadcasts the find to every node that can pick the object up.

```go
moveRule   := `rule MoveRule on altitude foo for "GUIDED" == mode && altitude > 3.0 && foo == "abb" do move_x = -5.0, move_y = 0.0, move_z = 2.0, foo = "qwe"`
alarmRule  := `rule AlarmRule on detection_sensor for true do detection_alarm = detection_sensor`
globalRule := `rule DetectionRule on detection_alarm for all true == ext.detection_picker do ext.detectionfound = true, ext.detectionfound_lat = position_lat, ext.detectionfound_lon = position_lon`
```

On the receiving side, `copterRule1` starts `cop2`'s launch sequence by writing the trigger value
into `foo`, and `copterRule2` flies it to the reported coordinates once it is safely airborne.

```go
copterRule1 := `rule DetectionTakeOffRule on detectionfound for detectionfound do foo = "abc"`
copterRule2 := `rule GoToDetectionRule on detectionfound altitude for base_alt > 0.0 && altitude > 3.0 && detectionfound do setposition_lat = detectionfound_lat, setposition_lon = detectionfound_lon, setposition_alt = position_alt`
```

Finally, we make some rules for landing and putting the vehicles back in their default mode.

```go
landRule  := `rule LandRule on foo for "cba" == foo do set_mode = "LAND"`
landRule2 := `rule LandRule2 on altitude for "cba" == foo && altitude == 0.0 do set_mode = "STABILIZE"`
```

Now we make the transaction managers and the executers. Note that `cop2` does not get `globalRule`:
it is a receiver of that remote task, not a sender, and `for all` conditions are evaluated on every
node regardless of whether it owns the rule.

```go
cop1_agent, _ := agent.NewRosAgent()
cop2_agent, _ := agent.NewRosAgent()

executer, err := aburos.NewRosExecuter(
    mem,
    []string{initRule, armRule, takeoffRule, moveRule, alarmRule, globalRule, landRule, landRule2},
    cop1_agent, "copter1", "aburos", "lazy")
if err != nil {
    fmt.Printf("failed to create executer: %v\n", err)
    return
}
fmt.Println("Created first executer")

executer2, err := aburos.NewRosExecuter(
    mem_cop,
    []string{initRule, armRule, takeoffRule, copterRule1, copterRule2, landRule, landRule2},
    cop2_agent, "copter2", "aburos", "lazy")
if err != nil {
    fmt.Printf("failed to create executer: %v\n", err)
    return
}
fmt.Println("Created second executer")
```

Lastly, we loop the executers and use inputs to interact with the system.

```go
go func() {
    for {
        executer.Exec()
    }
}()
go func() {
    for {
        executer2.Exec()
    }
}()

time.Sleep(2 * time.Second)
executer.Input(`foo = "abc"`)          // cop1: GUIDED -> arm -> take off

time.Sleep(6 * time.Second)
executer.Input(`foo = "abb"`)          // cop1: start the patrol leg

state, _ := executer2.TakeState()
fmt.Println("cop2 altitude:", state.Float["altitude"])

time.Sleep(30 * time.Second)
executer.Input(`foo = "cba"`)          // both: land
executer2.Input(`foo = "cba"`)
time.Sleep(60 * time.Second)
```

`cop2` is never told to take off by us: `globalRule` reaches it because `detection_picker` is true,
which sets `detectionfound` and starts its own local chain.

!!! tip "Try it without hardware"
    The same program can be run against a fleet of ArduPilot SITL instances, or declaratively
    through the [AbU Simulator](tools/abusim.md), which spawns one container per node and gives you
    a web UI over the whole thing.

## Supported vehicles

| Vehicle | `ROSresources` | `Vehicle` | Notes |
| --- | --- | --- | --- |
| ArduCopter | `CopterResources` | `CopterGoROSetta` | Full support |
| ArduCopter (ARGOS) | `ArgoCopterResources` | `ArgoCopter` | Copter plus `kalman_x` / `kalman_y` |
| ArduRover | `RoverResources` | `ArduRover` | Library only — the `abumon` and `abusim` agents reject `type: "rover"` with *not yet supported* |
| ArduSub | `SubResources` | `SubVehicle` | Uses `depth` instead of `altitude` |
| ArduPlane | — | — | **WIP.** `type: "plane"` is recognised but returns *not yet supported*; there is no `PlaneResources` yet |

So the node types you can actually deploy today are `basic`, `copter`, `sub`, and — with `abumon`
only — `argoCopter`.

Non-vehicle resources also exist for testing and integration: `AlarmResource`, `BaseResource`,
`HttpResource`, `TopicResource` and `ServiceResource`.

## Default physical attributes

These attributes are reserved: they are declared by the vehicle (`GetReservedKeywords()`) and wired
to the autopilot by the I/O manager. Any other attribute you put in memory is an ordinary AbU
attribute, local to the node and free for you to use.

*Input* attributes are refreshed from the vehicle each cycle by `gatherInputs`; writing to them has
no effect on the autopilot. *Output* attributes are commands — assigning one flags it as modified,
and `SendCommands` dispatches it at the end of the `Exec`.

### Common to all vehicles

| Attribute | Components | Type | Example value | Description |
| --- | --- | --- | --- | --- |
| `mode` | — | Input — Text | `"GUIDED"` | Current autopilot mode of the vehicle. Defaults to `"UNKNOWN"` until the first telemetry arrives |
| `set_mode` | — | Output — Text | `"GUIDED"` | Sends the desired mode to the autopilot |
| `arm` | — | Input — Bool | `false` | Current arming state of the vehicle's motors |
| `set_arm` | — | Output — Bool | `true` | Sets the arming state of the vehicle's motors |
| `position` | `position_lat`, `position_lon`, `position_alt` | Input — Integer, Integer, Float | `(460817880, 132156950, 120.0)` | Current GNSS coordinates of the vehicle. Latitude and longitude are specified in WGS84 and multiplied by 10⁷. Altitude is meters above mean sea level (MSL) |
| `setposition` | `setposition_lat`, `setposition_lon`, `setposition_alt` | Output — Integer, Integer, Float | `(460817880, 132156950, 120.0)` | Sends the desired GNSS coordinates to the autopilot. Same units as `position` |
| `move` | `move_x`, `move_y`, `move_z` | Output — Float | `(3.0, -1.1, 2.4)` | Forward-Right-Down (vehicle-centric) movement offset, in meters |
| `velocity` | `vx`, `vy`, `vz` | Input — Float | `(1.2, 0.0, -0.3)` | Current velocity vector of the vehicle in the FRD frame, in m/s. Defaults to `-1` before the first reading |
| `setvelocity` | `setvel_vx`, `setvel_vy`, `setvel_vz` | Output — Float | `(1.0, 0.0, 0.0)` | Sends the desired velocity vector to the autopilot, in m/s |
| `heading` | — | Input — Float | `90.0` | Current heading of the vehicle, in degrees. Defaults to `-1` before the first reading |

Because `move`, `setposition` and `setvelocity` are single autopilot commands split across several
attributes, assign all their components in the **same** `do` clause — the I/O manager collects the
modified components and sends one command at the end of the `Exec`.

### Vehicle-specific

| Attribute | Type | Example value | Vehicles | Description |
| --- | --- | --- | --- | --- |
| `altitude` | Input — Float | `5.0` | Copter | Altitude of the vehicle relative to the ground (AGL). Reserved on Rover too, but the Rover I/O manager does not currently publish it |
| `take_off` | Output — Float | `5.0` | Copter | Commands a take-off to the given altitude above ground |
| `land` | Output — Bool | `true` | Copter | Commands a landing. Implemented in `CopterResources`, but **not** in the default reserved-keyword list — pass `"land"` in `extraKeywords` when constructing the vehicle to make it reachable |
| `depth` | Input — Float | `3.2` | Sub | Depth of the vehicle relative to the water surface |
| `init` | Output — Bool | `true` | Rover, Sub | Setting it to `true` (re)initializes the vehicle connection and restarts input gathering. Reserved on Copter, but unused: the copter starts gathering at construction |
| `fake_sensor` | Input — Bool | `false` | Copter, Sub | Stub sensor, published from the vehicle's `FakeSensor` attribute. Useful for exercising rule chains without real hardware |
| `kalman_x`, `kalman_y` | Input — Float | `0.4` | ARGOS Copter | Kalman-filtered position correction, on top of the copter attributes |

Every vehicle constructor takes an `extraKeywords []string` argument; names passed there become
reserved attributes for that node, so custom sensors and actuators plug into the same
`Modified` / `SendCommands` machinery. See
[adding new vehicles](contributing.md#adding-new-vehicles).
