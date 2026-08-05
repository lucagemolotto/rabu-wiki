# Contributing
The repository for Robo-Abu can be found [here]().
It is divided different logical modules

## Setup
Robo-AbU depends on TII-UAE's [rclgo](https://github.com/tiiuae/rclgo), as such it currently support only ROS2 Humble.

Once ROS2 Humble is setup, install the custom ROS2 messages for Robo-AbU and goROSetta.
```
# build the messages
cd /PATH_TO_ROBOABU/aburos_msgs/ && source /opt/ros/humble/setup.bash && colcon build
# put the source command in the shell startup, bash is given as an example, use .zshrc if you use zsh
echo 'source /PATH_TO_ROBOABU/aburos_msgs/install/setup.bash' >> ~/.bashrc 

# do the same for gorosetta
cd /PATH_TO_GOROSETTA/goROSetta_msgs/ && source /opt/ros/humble/setup.bash && colcon build
echo 'source /PATH_TO_GOROSETTA/goROSetta_msgs/install/setup.bash' >> ~/.bashrc
``` 

And generate the CGO bindings for the messages
```
source /opt/ros/humble/setup.bash && source /home/aislab/aburos/aburos_msgs/install/setup.bash && source /home/aislab/goROSetta/goROSetta_msgs/install/setup.bash && go run github.com/tiiuae/rclgo/cmd/rclgo-gen generate -d /PATH_TO_ROBOABU/msgs --message-module-prefix "github.com/Autonomous-Systems-Laboratory-UNIUD/aburos/msgs"

## only needed if developing for goROSetta too
source /opt/ros/humble/setup.bash && source /home/aislab/goROSetta/goROSetta_msgs/install/setup.bash && go run github.com/tiiuae/rclgo/cmd/rclgo-gen generate -d /PATH_TO_GOROSETTA/ROSetta/msgs --message-module-prefix "github.com/Autonomous-Systems-Laboratory-UNIUD/goROSetta/ROSetta/msgs"
```

Alternatively, one can use the provided devcontainer available [here](https://github.com/Autonomous-Systems-Laboratory-UNIUD/AbU-ROS-devcontainer) and attach to it via Visual Studio Code.

## Modules
### agent
Implementations of different agents for the transaction mananager. They should all be implementations of the `RoboAbuAgent` interface.
Currently, we provide the following agents:

* `rosAgent` - ROS2-based agent, using the topic `/aburos/aburos/global` for communication. Implements the Decentralized 2PC Protocol from [Comini et al.](https://link.springer.com/chapter/10.1007/978-3-031-95497-9_6)
* `zenohAgent` - Zenoh-native agent, currently WIP.
* `abCast` - Implementation of the ISIS-ABCast protocol for atomic broadcast/total ordering.
* `seqAgent` - Simple centralized sequencer, nodes are clients using the service `/aburos/request_sequence` to request a sequence number and publish transaction with the number attached. Needs a ROS2 server implementing the sequencer, a docker image is available with `docker pull lucagemolotto/abumon-sequencer`.
### eagerevaluation
Implementation of the classical eager evaluation of updates seen in the original AbU papers.
### lazyevaluation
Implementation of the new *lazy* evaluation featured in Robo-AbU.
In this, updates take the form of the whole task (condition + actions) and are evaluated only when taken from the pool.
### lockCoordinator
Lock strategy of Robo-AbU, mostly taken from the original [goAbU](https://github.com/abu-lang/goabu).
### logger
The shared logger used by the different modules in Robo-AbU, writes to the directory define in `logger.config`.
### parser
Rule parser based on GRule engine, mostly taken from [goAbU](https://github.com/abu-lang/goabu).
### rosresources
This module contains the implementations of the I/O Managers of all the vehicles supported by Robo-AbU.
The I/O Manager is the gateway to the vehicle, it handles inputs from its sensors, via the `gatherInputs` function and the ouput to its actuators, using `SendCommands`.

All implementations **must** ad-here to the `RosResources` interface.
See [adding new vehicles](contributing.md#adding-new-vehicles) for more information.

### update
Interface for Robo-AbU updates, implementations should be their own module, as done by `eagerevaluation` and `lazyevaluation`.
### utilities
Generic utilities for other modules.
### vehicles
This module contains the Golang APIs for vehicles that are interface via messages, for example ROS2 or MAVLink.
Currently, we provide APIs for ArduPilot-based vehicles via goROSetta.
Vehicles for existing `RosResources` should ad-here to the interfaces in `rosVehicle.go`.
### wireTask
Utilities for sending and receiving `RemoteTasks` via transactions.
### Executer
`abuRosExecuter.go` and `abuRosGoRoutine.go` provide core of the Robo-AbU implemenations. Together, they make up the `Executer` module and implement the `Exec`, `Exec-F`, `Disc` and `Input` rules.

## Adding new vehicles

Support of a type of vehicle comes in the form of an implementation of the `RosResources` interface.
Optionally, specific vehicle Golang APIs can be implemented as part the `vehicles` module, this is the expected practice when the vehicle only has message-based APIs (for example via MAVLink or ROS2).
For example, support for VTOLs should first come in the form of a new `RosResources` implementation, while support for PX4 quadcopters should be a `vehicles.Copter` used by the already existing `CopterResource`. 

The attributes related to commands, sensors and actuators must be provided in the form a string slice and assigned to the field `rosKeywords`.
When initializing the vehicle resources, a default value for all these attributes should be set, see `CopterRosResource.go` for an example.
Additionally, vehicles can be easily extended by simply wrapping other implementations, see for example `argoCopterResource.go` which extends `CopterResource`

The `Modified` functions is used to tell the I/O manager which attributes are being modified by an `Exec`, the `SendCommands` function is then called at the end of the `Exec` to send all the gathered outputs. This is particularly useful for commands that are made up by multiple attributes, for example `move` which has the components `x`, `y` and `z`, as we can wait to see which of them are being used and then send out a single command with all three.
`ResetCommands` is used after the `SendCommands` and simply resets the modification flags done by `Modified`.

Implementations of new vehicles of existing `RosResources` must ad-here to the interfaces found in `vehicles/rosVehicle.go`.

