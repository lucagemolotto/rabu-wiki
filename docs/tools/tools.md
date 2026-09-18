# Tools

Robo-AbU is a Go library: you can embed it directly, build a `RosExecuter` by hand and drive it from
your own `main`. The tools below exist so you do not have to.

| Tool | Answers | Nodes run… |
| --- | --- | --- |
| [**abumon**](abumon.md) — AbU Monitor | *How do I put an AbU node on this drone?* | One container per physical device, on the device |
| [**abusim**](abusim.md) — AbU Simulator | *How does this fleet behave before I fly it?* | One container per node, all on one machine, spawned for you |
| [**goROSetta**](gorosetta.md) | *How does any of this reach the autopilot?* | As a library under both of the above |

## Choosing between abumon and abusim

They are two deployments of the same agent, and they read the same rules.

**`abumon`** is the deployment target. One config file, one rules file, one `docker compose up` on
the companion computer of each vehicle. Configuration is static — a node is what its YAML says it is
at boot. Nodes find each other over the network by themselves; there is no central authority, which
is the point of attribute-based interaction.

**`abusim`** is the development target. A coordinator process owns the whole simulation: it spawns
and destroys agent containers on demand, proxies `Input` into them, reads their memory back out, and
serves a web UI over the lot. You can add a node, edit a ruleset and restart a run without touching
a file. Nothing is persisted between runs.

The rule syntax, the attribute model and the vehicle resources are identical, so a ruleset developed
against `abusim` drops into `abumon` unchanged. What differs is the *transport* — see below — and
the fact that `abusim` nodes talk to SITL instances instead of real autopilots.

## Transaction managers

Both tools pick an implementation of `RoboAbuAgent`, the component that carries remote (`for all`)
tasks between nodes. The choice is a deployment decision, not a language one: the same rules run
under any of them, with different ordering and failure characteristics.

| Agent | Selected by | Description |
| --- | --- | --- |
| `rosAgent` | `abumon` (fixed), `abusim` with `AGENT_TYPE=D2PC` | Decentralized 2PC over ROS 2. No central component |
| `seqAgent` | `abusim` with `AGENT_TYPE=SEQ` | Centralized sequencer. Needs the sequencer service running |
| `abCast` | `abusim` with `AGENT_TYPE=ABCAST` | ISIS atomic broadcast, for total ordering |
| `zenohAgent` | — | Zenoh-native, WIP |

See [agent](../contributing.md#agent) for the details.

## AbU-UI

[AbU-UI](https://github.com/Autonomous-Systems-Laboratory-UNIUD/AbU-UI) is the web front-end served
by the `abusim` coordinator. It is a separate repository, built with Vite, and is described as part
of the [simulator](abusim.md#ui).
