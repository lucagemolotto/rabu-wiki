# Contributing

The repository for Robo-AbU is
[AbU-ROS](https://github.com/Autonomous-Systems-Laboratory-UNIUD/AbU-ROS) (Go module
`github.com/Autonomous-Systems-Laboratory-UNIUD/aburos`). It is divided into different logical
modules, described [below](#modules).

## Setup

Robo-AbU depends on TII-UAE's [rclgo](https://github.com/tiiuae/rclgo), and so currently supports
only **ROS 2 Humble**. Go ≥ 1.24 is required.

Once ROS 2 Humble is set up, install the custom ROS 2 messages for Robo-AbU and goROSetta.

```bash
# build the messages
cd /PATH_TO_ROBOABU/aburos_msgs/ && source /opt/ros/humble/setup.bash && colcon build
# put the source command in the shell startup; bash is given as an example, use .zshrc if you use zsh
echo 'source /PATH_TO_ROBOABU/aburos_msgs/install/setup.bash' >> ~/.bashrc

# do the same for goROSetta
cd /PATH_TO_GOROSETTA/goROSetta_msgs/ && source /opt/ros/humble/setup.bash && colcon build
echo 'source /PATH_TO_GOROSETTA/goROSetta_msgs/install/setup.bash' >> ~/.bashrc
```

And generate the CGO bindings for the messages:

```bash
source /opt/ros/humble/setup.bash \
  && source /PATH_TO_ROBOABU/aburos_msgs/install/setup.bash \
  && source /PATH_TO_GOROSETTA/goROSetta_msgs/install/setup.bash \
  && go run github.com/tiiuae/rclgo/cmd/rclgo-gen generate -d /PATH_TO_ROBOABU/msgs \
       --message-module-prefix "github.com/Autonomous-Systems-Laboratory-UNIUD/aburos/msgs"

## only needed if developing for goROSetta too
source /opt/ros/humble/setup.bash \
  && source /PATH_TO_GOROSETTA/goROSetta_msgs/install/setup.bash \
  && go run github.com/tiiuae/rclgo/cmd/rclgo-gen generate -d /PATH_TO_GOROSETTA/ROSetta/msgs \
       --message-module-prefix "github.com/Autonomous-Systems-Laboratory-UNIUD/goROSetta/ROSetta/msgs"
```

The bindings are generated from the *installed* message packages, so both `install/setup.bash`
scripts must be sourced in the same shell that runs `rclgo-gen`, and re-run it after any `.msg` or
`.srv` change.

Alternatively, one can use the provided devcontainer available
[here](https://github.com/Autonomous-Systems-Laboratory-UNIUD/AbU-ROS-devcontainer) and attach to it
via Visual Studio Code. Architecture-specific containers also exist for
[aarch64](https://github.com/Autonomous-Systems-Laboratory-UNIUD/abu-ros-container-aarch64) and
[aarch32](https://github.com/Autonomous-Systems-Laboratory-UNIUD/abu-ros-container-aarch32) targets.

### Logging

The shared logger reads its settings from a config file whose path is **hard-coded** to
`/home/aislab/aburos/logger/logger.config`, and panics if it is missing. It has the form:

```
ABU_LOGS_DIR=/home/aislab/aburos/logger
ABU_LOG_ENV=dev
ABU_LOG_LEVEL=info
```

| Variable | Effect |
| --- | --- |
| `ABU_LOGS_DIR` | Directory the per-node log files are written to, as `app_<id>_<RFC3339>.log` |
| `ABU_LOG_ENV` | `dev` adds human-readable console output on top of the file |
| `ABU_LOGLEVEL` | zerolog level: `trace`, `debug`, `info`, `warn`, `error` |

!!! warning "Level key mismatch"
    The shipped `logger.config` sets `ABU_LOG_LEVEL`, but `logger.go` reads `ABU_LOGLEVEL`. As
    shipped the level always falls back to the default. Set both until this is reconciled.

## Modules

### agent

Implementations of different agents for the transaction manager. They should all implement the
`RoboAbuAgent` interface:

```go
type RoboAbuAgent interface {
    Init(string, string) error            // Initializes the communication service
    Start()                               // Starts the communication service routine
    Stop() error                          // Stops the communication service
    ForAll([]byte) error                  // Broadcasts a payload/task via attribute-based communication
    GetWirePool() chan wireTask.WireTasks // Returns the WirePool of the agent
}
```

An agent is the only thing that knows how a remote task physically travels. It receives serialized
[wireTasks](#wiretask), delivers them into the executer's wire pool, and guarantees whatever
ordering its protocol provides. Swapping agents changes the fleet's consistency and failure
characteristics without touching a single rule.

Currently, we provide the following agents:

| Agent | Constructor | Description |
| --- | --- | --- |
| `rosAgent` | `NewRosAgent()` | ROS 2-based agent, using the topic `/aburos/aburos/global` for communication. Implements the decentralized 2PC protocol from [Comini et al.](https://link.springer.com/chapter/10.1007/978-3-031-95497-9_6) |
| `rosAgentNgbr` | `NewRosAgentNgbr()` | Neighbourhood-scoped variant of `rosAgent` |
| `abCast` | `NewABCastAgent()` | Implementation of the ISIS-ABCast protocol for atomic broadcast / total ordering |
| `seqAgent` | `NewSeqAgent()` | Simple centralized sequencer. Nodes are clients that request a sequence number from the service `/aburos/request_sequence` and publish transactions with the number attached. Needs a ROS 2 server implementing the sequencer — `docker pull lucagemolotto/abumon-sequencer`, or `NewSequencer()` to run one in-process |
| `zenohAgent` | — | Zenoh-native agent, currently WIP (the file is an empty stub) |

### eagerevaluation

Implementation of the classical eager evaluation of updates seen in the original AbU papers. An
update carries the already-computed assignments: the condition is evaluated when the task is
*received*, and what enters the pool is a concrete set of writes.

Selected by passing `"eager"` as the `eval` argument of `NewRosExecuter`.

### lazyevaluation

Implementation of the new *lazy* evaluation featured in Robo-AbU. Here updates take the form of the
whole task (condition + actions), and are evaluated only when taken from the pool.

Because a robot's state changes continuously — altitude, position, mode — a condition that held when
a task was sent may well be false by the time the task is applied. Deferring evaluation to the
moment of application is what keeps rules meaningful on a moving vehicle.

Selected by passing `"lazy"` as the `eval` argument of `NewRosExecuter`. Both `abumon` and `abusim`
use lazy evaluation.

### lockCoordinator

Lock strategy of Robo-AbU, mostly taken from the original
[goAbU](https://github.com/abu-lang/goabu). `coordinator.go` guards the node's memory across
concurrent `Exec` and `Input`; `execCoordinator.go` sequences the phases of a single `Exec`.

### logger

The shared logger used by the different modules in Robo-AbU. Writes to the directory defined in
`logger.config` — see [Logging](#logging).

### parser

Rule parser based on the [Grule](https://github.com/hyperjumptech/grule-rule-engine) rule engine,
mostly taken from [goAbU](https://github.com/abu-lang/goabu). The ANTLR grammars
(`EcaruleParser.g4`, `EcaruleLexer.g4`, `Grulev3Parser.g4`) define the rule syntax; regenerate the
Go parser with the module's `Makefile` after editing them.

`reservedWords.go` lists the identifiers that cannot be used as attribute names.

### rosresources

This module contains the implementations of the I/O managers of all the vehicles supported by
Robo-AbU. The I/O manager is the gateway to the vehicle: it handles inputs from its sensors, via the
`gatherInputs` function, and the output to its actuators, using `SendCommands`.

All implementations **must** adhere to the `ROSresources` interface. See
[adding new vehicles](#adding-new-vehicles) for more information.

Beyond the vehicle resources, the module also provides `BaseResource` (no hardware at all),
`AlarmResource`, `HttpResource`, `TopicResource` and `ServiceResource` for integrating non-vehicle
devices and services.

### update

Interface for Robo-AbU updates. Implementations should be their own module, as done by
[eagerevaluation](#eagerevaluation) and [lazyevaluation](#lazyevaluation).

### utilities

Generic utilities for other modules.

### vehicles

This module contains the Golang APIs for vehicles that are interfaced via messages, for example
ROS 2 or MAVLink. Currently, we provide APIs for ArduPilot-based vehicles via
[goROSetta](tools/gorosetta.md). Vehicles for existing `ROSresources` should adhere to the
interfaces in `rosVehicle.go`, which also defines `BasicVehicleKeywords`, the default reserved
attribute set.

### wireTask

Utilities for sending and receiving `RemoteTasks` via transactions — the serialized form a `for all`
task takes on the wire between an agent and its peers.

### Executer

`abuRosExecuter.go` and `abuRosGoRoutine.go` provide the core of the Robo-AbU implementation.
Together, they make up the `Executer` module and implement the `Exec`, `Exec-F`, `Disc` and `Input`
rules of the calculus. `abuRosCmd.go` dispatches the resulting commands to the I/O manager, and
`builtinFunctions.go` holds the functions callable from within rules.

## Adding new vehicles

Support for a type of vehicle comes in the form of an implementation of the `ROSresources`
interface. Optionally, specific vehicle Golang APIs can be implemented as part of the `vehicles`
module; this is the expected practice when the vehicle only has message-based APIs (for example via
MAVLink or ROS 2).

For example, support for VTOLs should first come in the form of a new `ROSresources`
implementation, while support for PX4 quadcopters should be a `vehicles.Copter` used by the already
existing `CopterResource`.

The attributes related to commands, sensors and actuators must be provided in the form of a string
slice and assigned to the field `rosKeywords`. When initializing the vehicle resources, a default
value for all these attributes should be set; see `CopterRosResource.go` for an example.

!!! important "An attribute with no default is invisible"
    `Modified` returns early unless the attribute is both present in `Resources` **and** listed in
    `rosKeywords`. An attribute missing either one is silently treated as an ordinary local
    attribute, and its command is never sent. This is why `land` exists in `CopterResources` but
    does nothing: it is not in `BasicVehicleKeywords`.

Additionally, vehicles can be easily extended by simply wrapping other implementations; see for
example `argoCopterResource.go`, which extends `CopterResource` with `kalman_x` and `kalman_y`.

### The command cycle

| Function | When | Role |
| --- | --- | --- |
| `gatherInputs` | Continuously, in its own goroutine | Polls the vehicle and pushes assignment strings into the inputs channel. Typically every 100 ms |
| `Modified(r string)` | During an `Exec`, per written attribute | Tells the I/O manager which attributes are being modified, flagging the composite command they belong to |
| `SendCommands()` | At the end of an `Exec` | Sends every flagged command to the vehicle |
| `ResetCommands()` | After `SendCommands` | Clears the modification flags set by `Modified` |

Splitting `Modified` from `SendCommands` is what makes composite commands work: `move` has the
components `x`, `y` and `z`, so the manager waits to see which of them were written during the step
and then sends a single command carrying all three. Group the components of a composite command in
one `do` clause for this reason.

Implementations of new vehicles for existing `ROSresources` must adhere to the interfaces found in
`vehicles/rosVehicle.go`.

## Adding a new agent

Implement `RoboAbuAgent` in its own file under `agent/`. `Init` receives the local and remote
namespaces, `ForAll` is handed the serialized task to broadcast, and `GetWirePool` returns the
channel the executer drains. Look at `seqAgent.go` for the smallest complete example, and at
`rosAgent.go` for the decentralized one.

To make the new agent selectable from the tools, add a case to the agent construction in
[`abumon`](tools/abumon.md)'s `main.go` and to `schema.AgentType` in
[`abusim-core`](tools/abusim.md).
