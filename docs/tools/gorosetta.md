# goROSetta / goMavUtil

[goROSetta](https://github.com/Autonomous-Systems-Laboratory-UNIUD/goROSetta) is the layer that sits
between Robo-AbU and the vehicle. It is a Go port of the original
[ROSetta](https://github.com/Autonomous-Systems-Laboratory-UNIUD/ROSetta/), a compatibility layer
between MAVLink and ROS 2, and it supports all ArduPilot vehicles.

The repository holds two independent Go modules:

| Module | Purpose |
| --- | --- |
| `goMavUtil` | A MAVLink client. Connects to an autopilot, tracks its state, and exposes commands as Go methods. No ROS 2 involved. |
| `ROSetta` | A ROS 2 node wrapping a `goMavUtil` connection, republishing vehicle state on ROS 2 topics and exposing commands as ROS 2 topics and services. |

Robo-AbU's `vehicles` package talks to the `ROSetta` node, which talks to `goMavUtil`, which talks
MAVLink to the autopilot:

```
AbU rules → ROSresources → vehicles.CopterGoROSetta → ROS 2 → ROSettaNode → goMavUtil → MAVLink → ArduPilot
```

## goMavUtil

A `MavConn` is one MAVLink connection to one vehicle. It sends commands synchronously, waiting for
the autopilot's `COMMAND_ACK` (5 s timeout), and runs a background handler that keeps telemetry
fresh.

```go
conn, err := goMavUtil.NewMavConn(
    "0.0.0.0:14551",   // host:port — opened as a MAVLink UDP *server*
    1,                 // target system ID (the vehicle)
    1,                 // component ID
    gomavlib.V2,       // MAVLink version
    nil, nil,          // optional inbound/outbound signing keys
    time.Second,       // heartbeat period
)
```

The connection is a UDP server, so the autopilot (or SITL, or `mavlink-router`) dials *in* to this
address. The vehicle's `MAV_TYPE` is unknown until the first `HEARTBEAT` arrives, which is what
selects the mode table — give it a moment before issuing mode commands.

### Commands

| Method | Description |
| --- | --- |
| `Arm()` / `Disarm()` / `ToggleArm(bool)` | Arm or disarm the motors |
| `SetModeFromString(mode string)` | Set the flight mode by ArduPilot name, e.g. `"GUIDED"` |
| `SetModeFromInt(mode uint64)` | Set the flight mode by numeric ID |
| `TakeOff(alt float32)` | Take off to `alt` meters above ground |
| `SetVelocity(vx, vy, vz, yaw float32)` | Command a velocity vector, in m/s |
| `MoveByFRDOffset(x, y, z, vx, vy, vz, yaw float32)` | Move by an offset in the vehicle-centric Forward-Right-Down frame |
| `MoveToNEDPoint(x, y, z, vx, vy, vz, yaw float32)` | Move to a point in the North-East-Down frame, with the vehicle as origin |
| `MoveToGlobalPointMSL(lat, lon int32, alt float32, …)` | Fly to WGS84 coordinates, altitude above mean sea level |
| `MoveToGlobalPointRelativeAlt(lat, lon int32, alt float32, …)` | Fly to WGS84 coordinates, altitude relative to the home position |
| `MoveToGlobalPointAGL(lat, lon int32, alt float32, …)` | Fly to WGS84 coordinates, altitude above ground level (terrain-following) |
| `Close(node *gomavlib.Node)` | Tear the connection down |

Latitude and longitude are WGS84 degrees multiplied by 10⁷, matching MAVLink's own encoding and the
`position_lat` / `position_lon` attributes in Robo-AbU.

### State

| Method | Description |
| --- | --- |
| `Mode() (string, error)` | Current flight mode, decoded to its ArduPilot name |
| `IsArmed() bool` | Whether the motors are armed |
| `LoadPositionalData(key string) (int32, error)` | Read one cached positional field |

### Mode mappings

`mappings.go` carries the full numeric ⇄ name mode tables for **ArduCopter**, **ArduPlane**,
**ArduRover**, **ArduTracker**, **ArduSub** and **ArduBlimp**, in both directions. The right table is
picked from the vehicle's `MAV_TYPE`, so `SetModeFromString("GUIDED")` resolves differently on a
copter and on a rover without the caller caring.

`GetMovementTypeMask` does the same for position-target bitmasks, which differ per frame type.

## ROSetta

A `ROSettaNode` opens a MAVLink connection and mirrors it into ROS 2.

```go
node, err := gorosetta.NewROSettaNode(
    "copter1",   // node/vehicle id, used as the topic namespace
    "0.0.0.0",   // autopilot address
    "14551",     // autopilot port
    1,           // MAVLink system ID
    100,         // publish rate, in milliseconds
    nil,         // optional *zerolog.Logger
)
defer node.Close()
```

All interfaces are namespaced under the vehicle id, written `%vehicle_id` below.

### Publishers

| Topic | Type | Description |
| --- | --- | --- |
| `%vehicle_id/movement/position` | `goROSetta/Position` | Current GNSS coordinates of the vehicle |
| `%vehicle_id/movement/velocity` | `geometry_msgs/Twist` | Velocity vector of the vehicle, in m/s |
| `%vehicle_id/movement/telemetry` | `goROSetta/Telem` | Telemetry data — currently mode and armed/disarmed |

### Subscribers

| Topic | Type | Description |
| --- | --- | --- |
| `%vehicle_id/movement/move_frd` | `goROSetta/Movement` | FRD (Forward-Right-Down) movement vector, in meters |
| `%vehicle_id/movement/move_ned` | `goROSetta/Movement` | NED (North-East-Down) movement vector, in meters. The vehicle is the origin |
| `%vehicle_id/movement/move_glb` | `goROSetta/MovementGlobal` | WGS84 coordinates plus altitude relative to MSL |
| `%vehicle_id/movement/set_velocity` | `goROSetta/Movement` | Desired velocity vector, in m/s. Also supports a yaw angle |

### Services

| Service | Type | Description |
| --- | --- | --- |
| `%vehicle_id/movement/arm` | `std_srvs/SetBool` | Sets the arming state of the motors |
| `%vehicle_id/movement/set_mode` | `goROSetta/SetString` | Sets the ArduPilot mode |
| `%vehicle_id/movement/take_off` | `goROSetta/SetFloat` | Takes off to the given altitude, where the vehicle supports it |

### Message definitions

From `goROSetta_msgs`:

```
# Position.msg — a position in a 3-axis space, plus rotation
int32 x
int32 y
int32 z
int32 z_alt   # optional second Z, e.g. MSL in z and AGL in z_alt
int32 roll
int32 pitch
int32 yaw
```

```
# Movement.msg — a point to move to, and the velocity to use
geometry_msgs/Vector3 position
geometry_msgs/Vector3 velocity
float32 yaw
```

```
# MovementGlobal.msg — a global waypoint and velocity
int32 x       # latitude, WGS84 × 10^7
int32 y       # longitude, WGS84 × 10^7
float32 z     # altitude
float32 vx
float32 vy
float32 vz
```

```
# Telem.msg
string mode
bool arm_status
```

```
# SetFloat.srv / SetString.srv
float32 data     # (or: string data)
---
bool success     # whether the triggered service ran successfully
string message   # informational, e.g. an error message
```

## Installation

Go ≥ 1.24.4 and ROS 2 Humble are required.

```bash
cd goROSetta_msgs
source /opt/ros/humble/setup.bash && colcon build
echo 'source YOUR_PATH_TO/goROSetta_msgs/install/setup.bash' >> ~/.bashrc
```

Verify with `ros2 interface list` and look for `goROSetta/Movement`.

The Go bindings for the messages are generated by `rclgo-gen`; see
[Setup](../contributing.md#setup) for the exact command.

## Running

```bash
go run ./ROSetta/test/main.go -id copter1 -host 0.0.0.0 -port 14551 -sysid 1
```

| Flag | Default | Description |
| --- | --- | --- |
| `-id` | `test` | Node ID, used as the ROS 2 namespace |
| `-host` | `0.0.0.0` | Autopilot address |
| `-port` | `14551` | Autopilot port |
| `-sysid` | `1` | MAVLink system ID, 1–255 |
| `-test` | `false` | Run a smoke test against the node instead of blocking |

Or build it with `go build ./ROSetta/test/main.go` inside a ROS 2 environment. See
[`ROSetta/test/main.go`](https://github.com/Autonomous-Systems-Laboratory-UNIUD/goROSetta/blob/main/ROSetta/test/main.go)
for example usage.
