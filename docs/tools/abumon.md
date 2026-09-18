# AbU Monitor

The AbU Monitor (`abumon`) is the tool for deploying and monitoring AbU nodes on single devices.
It is packaged as a Docker Compose service, from the
[abumon-goabu-agent](https://github.com/Autonomous-Systems-Laboratory-UNIUD/abumon-goabu-agent)
repository, and configured from the `abu-config` folder — which also holds a set of examples.

One container is one AbU node. Put it on the companion computer next to the autopilot, point it at
the autopilot's MAVLink endpoint, give it a rules file, and it joins the fleet on its own — there is
no central component to register with.

## Running

From the directory containing `docker-compose.yaml`:

```bash
docker compose up
```

The compose file mounts your local `abu-config/` into the container and selects the config file
through the `ABUCONFIG_PATH` environment variable:

```yaml
services:
  abumon:
    image: lucagemolotto/abumon-agent
    hostname: sub1
    network_mode: host
    volumes:
      - ./abu-config:/home/aislab/agent/abumon-goabu-agent/abu-config
      - ./zenoh_bridge.json5.template:/home/aislab/agent/abumon-goabu-agent/zenoh_bridge.json5.template
    environment:
      - ABUCONFIG_PATH=/home/aislab/agent/abumon-goabu-agent/abu-config/config.yaml
```

`network_mode: host` matters: ROS 2 discovery and the MAVLink UDP endpoint both need the host's
network. Set `hostname` to the node's identifier so logs are readable.

`ABUCONFIG_PATH` has no default — the agent exits immediately if it is unset.

## Node configuration

The AbU node is configured from a single YAML file, conventionally `abu-config/config.yaml`.

| Field | Type | Example value | Description |
| --- | --- | --- | --- |
| `id` | string | `"sub1"` | Unique identifier for the node. Used as the AbU local namespace and as the goROSetta node name |
| `type` | string | `"sub"` | The vehicle resource to instantiate. One of `basic`, `copter`, `argoCopter`, `sub`. (`plane` and `rover` are recognised but return *not yet supported*) |
| `tick` | duration | `50ms` | Delay between two `Exec` steps of the agent |
| `rules` | string (path) | `"/…/abu-config/rules.yaml"` | Path to the YAML file containing the node's rules, **as seen inside the container** |
| `memory` | object | — | The node's initial memory, organised by data type |
| `memory.Text` | object | — | Text/string variables. Each key is a variable name, each value a string |
| `memory.Bool` | object | — | Boolean variables, `true` or `false` |
| `memory.Integer` | object | — | Integer variables |
| `memory.Float` | object | — | Floating-point variables |
| `autopilot` | object | — | How to reach the autopilot. Only used for vehicle types |
| `autopilot.addr` | string | `"0.0.0.0"` | Address the MAVLink UDP server binds to |
| `autopilot.port` | integer | `14551` | MAVLink port |
| `autopilot.id` | integer | `1` | MAVLink system ID of the vehicle, 1–255 |

!!! warning "`tick` must carry a unit"
    `Tick` is a Go `time.Duration`, so it has to be written with a unit — `50ms`, `1s`. A bare
    integer is **not** accepted: `tick: 50` fails to unmarshal with
    *cannot unmarshal !!int `50` into time.Duration*, and the agent exits reporting it.

!!! note "Copy from `config.yaml` or `config_template.yaml`"
    The older `config_example.yaml` and `config_old.yaml` still say `tick_rate` and `autopilot.ip`.
    Unknown keys are dropped **silently**, so a config following those starts with a tick of `0` — a
    busy `Exec` loop — and an empty autopilot address.

`memory.Text.id` is not optional for vehicle types: it is what the agent passes as the vehicle
identifier when constructing the goROSetta node. Set it to the same value as the top-level `id`.

Reserved [physical attributes](../rabu.md#default-physical-attributes) do not need to be declared —
the vehicle resource provides them, with defaults. Declare only your own attributes here.

### Example

A configuration for an `argoCopter` called `cop1` which runs an `Exec` every 50 ms and has a `Text`
attribute `foo` with value `"a"`:

```yaml
id: "cop1"
type: "argoCopter"
tick: 50ms
rules: "/home/aislab/agent/abumon-goabu-agent/abu-config/rules.yaml"
memory:
  Text:
    id: "cop1"
    foo: "a"

autopilot:
  addr: "0.0.0.0"
  port: 14551
  id: 1
```

## Rule configuration

The rules file named by `rules:` is a **flat YAML list** of rule objects — one list item per rule.

```yaml
- name: <string>        # required — identifier for the rule
  event: <string>       # required — attribute(s) that trigger the rule, space-separated
  condtype: <integer>   # required — 0 for a local task (`for`), 1 for a remote task (`for all`)
  condition: <string>   # required — boolean expression, e.g. "mode == \"GUIDED\""
  action: <string>      # required — assignments, e.g. "set_mode = \"GUIDED\""
```

Each entry is assembled into the rule string
`rule <name> on <event> for|for all <condition> do <action>`, so anything valid in the
[Robo-AbU syntax](../rabu.md#syntax) is valid here. Remember to escape the double quotes around
string literals.

### Example

A set of rules to start a copter and take off to 8 meters:

```yaml
- name: "InitRule"
  event: "foo"
  condtype: 0
  condition: "\"abc\" == foo"
  action: "init = true"

- name: "ModeRule"
  event: "init"
  condtype: 0
  condition: "true == init"
  action: "set_mode = \"GUIDED\""

- name: "ArmRule"
  event: "mode"
  condtype: 0
  condition: "mode == \"GUIDED\""
  action: "set_arm = true"

- name: "TakeOffRule"
  event: "arm"
  condtype: 0
  condition: "arm == true"
  action: "take_off = 8.0"
```

Nothing happens until something writes `foo`. The agent injects `foo = "abc"` itself 45 seconds
after startup, which is the built-in bootstrap; drive it yourself instead if you need a different
trigger.

The shipped `abu-config/` contains several ready-made rulesets worth reading: `rules_complete.yaml`,
`rules_geofence.yaml`, `rules_no_kalman.yaml` and `rules_sub.yaml`.

## Startup sequence

1. Read `ABUCONFIG_PATH`, load the node configuration.
2. Load and parse the rules file.
3. Create a `rosAgent` — `abumon` always uses the decentralized 2PC transaction manager.
4. Build the memory for the configured `type`, seeded from `memory:`.
5. Build the `RosExecuter` with namespace `aburos` and **lazy** evaluation.
6. For vehicle types, create the goROSetta node against `autopilot`, retrying up to 4 times with a
   short backoff. If every attempt fails, the agent exits.
7. Loop: `Exec()`, then sleep for `tick`.

## Monitoring API

The repository contains an mTLS-protected HTTP server exposing the node's state:

| Endpoint | Returns |
| --- | --- |
| `GET /api/connect` | `ConnectionInfo` — name, type, memory, update pool and rules |
| `GET /api/memory` | `NodeMemoryRes` — memory, update pool and running status |
| `GET /api/log` | The node's most recent log file |

It listens on `127.0.0.1:31337`, requires a client certificate signed by the AbU root CA
(`RequireAndVerifyClientCert`, TLS ≥ 1.2), and reads its certificates from `./certs/`.

!!! note "Currently disabled"
    The call that starts this server is commented out in the agent's `main.go`, so a stock
    `abumon` container does not listen on 31337. Re-enable `server.NewAbumonServer` and provide the
    certificates to use it.

## Sequencer

If you want to run nodes against the centralized sequencer (`seqAgent`) instead of decentralized
2PC, a ROS 2 service implementing the sequencer is published as a Docker image:

```bash
docker pull lucagemolotto/abumon-sequencer
```

Nodes then request a sequence number from `/aburos/request_sequence` and publish their transactions
with that number attached. See [agent](../contributing.md#agent).
