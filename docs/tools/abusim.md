# AbU Simulator

For simulation purposes we also provide a centralized simulator, called `abusim`. Unlike
[abumon](abumon.md), where each node is configured from a file and boots on its own, `abusim` puts a
coordinator in charge of the whole fleet: it spawns and destroys agent containers on demand, proxies
inputs into them, reads their memory back out, and serves a web UI over the lot.

It is made up of three components:

| Component | Repository | Role |
| --- | --- | --- |
| `abusim-coordinator` | [abusim-core](https://github.com/Autonomous-Systems-Laboratory-UNIUD/abusim-core) | Owns the simulation: Docker orchestration, HTTP API, and the UI |
| `abusim-agent` | [abusim-goabu-agent](https://github.com/Autonomous-Systems-Laboratory-UNIUD/abusim-goabu-agent) | One AbU node, one container, spawned by the coordinator |
| AbU-sim UI | [AbU-UI](https://github.com/Autonomous-Systems-Laboratory-UNIUD/AbU-UI) | Web front-end, served by the coordinator |

## Running

The coordinator needs the Docker socket — it spawns containers — and an external control network.

```bash
docker network create abusim-default-control
docker compose up
```

`docker-compose.yaml` brings up the coordinator on port **4000** alongside a Zenoh router on 7447
and 8000. `AGENT_TYPE` selects the [transaction manager](../contributing.md#agent) all nodes will
use:

```yaml
environment:
  - DOCKER_HOST=unix:///var/run/docker.sock
  - AGENT_TYPE=D2PC      # D2PC | SEQ | ABCAST
```

| Value | Agent |
| --- | --- |
| `D2PC` | Decentralized 2PC (`rosAgent`) — the default |
| `SEQ` | Centralized sequencer (`seqAgent`). The coordinator can start the sequencer container itself |
| `ABCAST` | ISIS atomic broadcast (`abCast`) |

On startup the coordinator creates a second network, `abusim-default`, and joins every agent it
spawns to it.

## Agent

Agents get their configuration on startup via an `AgentConfiguration` provided through the
environment, Base64-encoded. You never write one by hand — the coordinator builds it from the node
you added and passes it in.

```go
type AgentConfiguration struct {
    Name             string                    // node identifier
    MemoryController string                    // vehicle type
    Memory           map[string]map[string]any // initial memory, by type
    Rules            []string                  // flattened from the node's rulesets
    Endpoints        []string
    Tick             time.Duration             // fixed at 10ms by the coordinator
    SimAddr          string                    // SITL address
    SimPort          int                       // SITL port, allocated from 14560 upwards
    SimID            uint8                     // MAVLink system ID
    AgentType        AgentType                 // D2PC | SEQ | ABCAST
}
```

Supported memory controllers are `basic`, `copter` and `sub`. `plane` and `rover` are recognised but
return *not yet supported* — see [supported vehicles](../rabu.md#supported-vehicles).

Each node is allocated a SITL port from `14560` upwards, in the order nodes are added, and a
matching MAVLink system ID. `GET /api/sim/resetport` resets that counter — useful after tearing a
simulation down and rebuilding it.

## Coordinator

The coordinator manages the agents, spawning and deleting their containers. It also acts as a proxy
for `Input` rules and for reading node state. It can be interacted with via the [UI](#ui), or
directly over its JSON/HTTP API on port 4000.

### Nodes

| Operation | Endpoint | Body / response |
| --- | --- | --- |
| Add nodes | `POST /api/nodes` | **An array** of `NodeConfig`. Nodes are created in order, ~2 s apart |
| Remove a node | `DELETE /api/nodes/:id` | — |
| List nodes | `GET /api/nodes` | Array of `NodeConfig` |
| All node states | `GET /api/nodes/memory` | Map of id → `NodeState` |
| One node's state | `GET /api/nodes/memory/:id` | `NodeState` |

```jsonc
// NodeConfig
{
  "id":       "cop1",
  "type":     "copter",              // basic | copter | sub
  "ruleSets": ["takeoff", "patrol"], // names of rulesets already added
  "memory":   { "foo": "a", "base_alt": 0.0, "detection_picker": true },
  "running":  false
}
```

`memory` here is **flat**, not grouped by type as in `abumon`: the coordinator infers the AbU type
from the JSON type of each value — string → `Text`, bool → `Bool`, number → `Float`. The key
`identifier` is special-cased to `Integer`.

Rulesets must be added **before** the node that references them; `AddNode` fails if a named set is
unknown.

```jsonc
// NodeState
{
  "memory":  { "altitude": 5.0, "mode": "GUIDED" },
  "pool":    [ /* pending updates */ ],
  "running": true
}
```

### Rulesets

| Operation | Endpoint | Body / response |
| --- | --- | --- |
| Add a ruleset | `POST /api/rulesets` | `RuleSet` |
| Remove a ruleset | `DELETE /api/rulesets/:id` | — |
| List rulesets | `GET /api/rulesets` | Map of name → array of `RuleConfig` |

```jsonc
// RuleSet
{
  "name": "takeoff",
  "rules": [
    {
      "name":      "ArmRule",
      "event":     "mode",
      "condtype":  0,                      // 0 = local (`for`), 1 = remote (`for all`)
      "condition": "mode == \"GUIDED\"",
      "action":    "set_arm = true"
    }
  ]
}
```

Each `RuleConfig` is assembled into `rule <name> on <event> for|for all <condition> do <action>` —
the same shape as an [abumon rules file](abumon.md#rule-configuration), so rules move between the
two unchanged.

### Simulation control

| Operation | Endpoint | Body |
| --- | --- | --- |
| Start one node | `GET /api/sim/start/:id` | — |
| Stop one node | `GET /api/sim/stop/:id` | — |
| Start all nodes | `GET /api/sim/start` | — |
| Stop all nodes | `GET /api/sim/stop` | — |
| Node logs | `GET /api/sim/logs/:id` | — |
| Send an input | `POST /api/sim/input/:id` | Array of `[name, value]` pairs |
| Reset port allocation | `GET /api/sim/resetport` | — |

An input is a list of two-element arrays, which the coordinator turns into an AbU assignment string
and forwards to the node's `Input`:

```json
[["foo", "abc"], ["base_alt", 0.0], ["armed", true]]
```

Values may be strings, numbers or booleans; anything else is rejected.

!!! bug "`stop/:id` starts the node"
    `api.StopNode` calls `c.StartNodeExecution(id)` rather than `StopNodeExecution`, so stopping a
    single node currently starts it. Stopping the whole simulation with `GET /api/sim/stop` is
    unaffected.

## UI

The UI is served by the `abusim-coordinator` itself, as static files, at
**<http://localhost:4000/>**. It serves as both a monitor and a controller for the simulation.

It provides a way to add and remove nodes, monitor their attributes live, and manage rules via
rulesets. Execution of nodes can then be started and stopped, individually or all at once.

The UI is a thin client over the API above: everything it does can be scripted, which is what the
[environment harness](#environment-harness) does.

## Environment harness

`abusim-environment/` holds a small Python harness for scripting the *world* around the simulation —
the part that is not a robot. You register callbacks on a node's attribute and respond by posting
inputs back into other nodes.

```python
from environment import Environment

env = Environment()

def change_temp(action, variables):
    room_temp = env.get_variables('temp_S1')['temperature']
    if action == 'increase':
        env.post_input('temp_S1', f'temperature = {room_temp + 1}')

env.on('conv_S1', 'action', 10, change_temp)   # node, attribute, poll period
env.loop()
```

This is how you close the loop on non-physical scenarios — thermostats, doors, alarms — without
writing a vehicle resource for them.

!!! warning "The harness targets an older API"
    `environment.py` calls `GET/POST http://localhost:4000/memory/<agent>`, which the current
    coordinator does not route. Point `get_variables` at `/api/nodes/memory/<agent>` and
    `post_input` at `/api/sim/input/<agent>` (with the `[[name, value], …]` body documented above)
    before using it.
