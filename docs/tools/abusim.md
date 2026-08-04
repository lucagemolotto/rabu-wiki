## AbU Simulator
For simulation purposes we also provide a centralized simulator, called `abusim`.

It is made up of three components:
* `abusim-agent`
* `abusim-coordinator`
* AbU-sim UI

### Agent
Agents get their configuration on startup via an `AgentConfiguration` provided via ENV.
Currently supports `Copter`, `Rover`, `Plane`, `Sub`, `Static`, `Basic`.
### Coordinator
The coordinator manages the agents, it spawns and deletes their containers. It also acts as a proxy for (Input) rules.
It can be interacted with via the [UI](abusim#ui), or via JSON/HTTP APIs.

```
APIs Definition

AddNode 
	endpoint: POST("/api/nodes")
	type: JSON { resource_type, id, rulesets {name1(str): NIL, name2: NIL, ...}, memory{(str):(int,float,bool,string)} }
RemoveNode
	endpoint: DELETE("/api/nodes/:nodename")
GetNode:
	endpoint: GET("/api/nodes/:nodename")
	type: JSON { resource_type, id, rulesets {name1(str): NIL, name2: NIL, ...}, memory{(str):(int,float,bool,string)} }
GetNodeState:
	endpoint: GET("/api/nodes/memory/:nodename")
	type: JSON { memory{(str):(int,float,bool,string)} }
(FUTURE) GetNodeLogs:
	endpoint: GET("/api/nodes/memory/:nodename")
	type: JSON { memory{(str):(int,float,bool,string)} }

AddSet
	endpoint: POST("/api/rulesets/")
	type JSON { id: str, rules: []rule}
	type rule : {name: str, event: str, cndtype: "for"/"for-all", condition: str, action: str}
RemoveSet
	endpoint: DELETE("/api/rulesets/:setname")
GetSets
	endpoint: GET("/api/rulesets/")
	type JSON []{ id: str, rules: []rule}

StartNode:
	endpoint: GET("/api/sim/start/:nodename")
StopNode:
	endpoint: GET("/api/sim/stop/:nodename")
StartSim:
	endpoint: GET("/api/sim/start/")
StopSim:
	endpoint: GET("/api/sim/stop/")
NodeInput:
	endpoint: POST("/api/sim/input/")
	type: JSON { id: str, input: [](var(str),val(int,float,bool,string)) }
```

### UI

The UI is served by the `abusim-coordinator`, it serves as both a monitor and a controller for the simulation.

It provides a way to add/remove agents/nodes, monitor their attributes and manage rules via rulesets.

Execution of nodes can also then be started/stopped.