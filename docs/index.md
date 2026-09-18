# Robo-AbU

**Robo-AbU** is a domain-specific language and runtime for programming distributed, event-driven
robotic systems with Event-Condition-Action (ECA) rules.

It is a robotics-oriented variant of [goAbU](https://github.com/abu-lang/goabu), the reference Go
implementation of the [AbU calculus](https://doi.org/10.1016/j.tcs.2023.113841). Robo-AbU keeps the
AbU programming model — local rules plus *attribute-based* remote tasks — and binds it to real
vehicles through ROS 2 and MAVLink, so a fleet coordinates without any node addressing another node
by name.

```
rule detectionRule
on detection_alarm
for all true == ext.detection_picker
do ext.detectionfound = true,
   ext.detectionfound_lat = position_lat,
   ext.detectionfound_lon = position_lon
```

One rule. Every node in range that can pick up the object learns where it is.

## Where to go

| If you want to… | Read |
| --- | --- |
| Learn the rule syntax and write your first program | [Robo-AbU](rabu.md) |
| Look up the attributes a vehicle exposes | [Default physical attributes](rabu.md#default-physical-attributes) |
| Deploy and monitor nodes on real hardware | [AbU Monitor](tools/abumon.md) |
| Run a fleet in simulation, with a web UI | [AbU Simulator](tools/abusim.md) |
| Talk to an ArduPilot vehicle from ROS 2 or Go | [goROSetta / goMavUtil](tools/gorosetta.md) |
| Build the code, or add a new vehicle | [Contributing](contributing.md) |

## The repositories

| Repository | What it is |
| --- | --- |
| [AbU-ROS](https://github.com/Autonomous-Systems-Laboratory-UNIUD/AbU-ROS) | The Robo-AbU language, runtime and vehicle resources (Go module `aburos`) |
| [goROSetta](https://github.com/Autonomous-Systems-Laboratory-UNIUD/goROSetta) | MAVLink ⇄ ROS 2 compatibility layer, plus the `goMavUtil` MAVLink client |
| [abumon-goabu-agent](https://github.com/Autonomous-Systems-Laboratory-UNIUD/abumon-goabu-agent) | `abumon` — the deployable agent for a single physical device |
| [abusim-core](https://github.com/Autonomous-Systems-Laboratory-UNIUD/abusim-core) | `abusim` coordinator: spawns, controls and proxies simulated nodes |
| [abusim-goabu-agent](https://github.com/Autonomous-Systems-Laboratory-UNIUD/abusim-goabu-agent) | The agent container the coordinator spawns |
| [AbU-UI](https://github.com/Autonomous-Systems-Laboratory-UNIUD/AbU-UI) | Web front-end for the simulator |
| [AbU-ROS-devcontainer](https://github.com/Autonomous-Systems-Laboratory-UNIUD/AbU-ROS-devcontainer) | Ready-made ROS 2 Humble development container |

## Requirements at a glance

* Go ≥ 1.24
* ROS 2 **Humble** (fixed by [rclgo](https://github.com/tiiuae/rclgo), the Go ROS 2 client)
* An ArduPilot vehicle or SITL instance, reachable over MAVLink

See [Setup](contributing.md#setup) for the full build procedure.
