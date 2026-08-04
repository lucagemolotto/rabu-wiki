## AbU Monitor
The AbU Monitor `(abumon)` is the tool for deploying and monitoring AbU nodes on single devices.
It's packaged as Docker Compose container, available [here](https://) and a [GitHub repository](https://github.com/Autonomous-Systems-Laboratory-UNIUD/abumon-goabu-agent).
It can be configured from the `abu-config` folder, which also provides some examples.

To start up the tool, it is sufficient to use `docker compose up` while inside the directory containing `docker-compose.yaml`.

### Node configurations
The AbU node can be configured from `abu-config.yaml`.

| Field                            | Type            | Example Value           | Description                                                                                                           |
| -------------------------------- | --------------- | ----------------------- | --------------------------------------------------------------------------------------------------------------------- |
| id                             | string        | "sub1"                | Unique identifier for the agent or component instance.                                                                |
| type                           | string        | "sub"                 | Defines the type or role of the agent.                                                                                |
| tick_rate                      | integer       | 50                    | Execution/update frequency of the agent, expressed in milliseconds.                                           |
| rules                          | string (path) | "/path/to/rules.yaml" | Path to the YAML file containing the agent's configurable behavior rules.                                             |
| memory                         | object        | —                       | Defines the agent's internal memory variables. Memory is organized by data type.                                      |
| memory.Text                    | object        | —                       | Collection of text/string variables. Each entry name is a variable identifier, and its value is a string.             |
| memory.Text.<variable_name>    | string        | val: "example"             | A configurable text variable stored in memory.                                                                        |
| memory.Bool                    | object        | —                       | Collection of boolean variables. Each entry name is a variable identifier, and its value is either true or false. |
| memory.Bool.<variable_name>    | boolean       | val: true                  | A configurable boolean variable stored in memory.                                                                     |
| memory.Integer                 | object        | —                       | Collection of integer variables. Each entry name is a variable identifier, and its value is an integer number.        |
| memory.Integer.<variable_name> | integer       | val: 10                    | A configurable integer variable stored in memory.                                                                     |
| memory.Float                   | object        | —                       | Collection of floating-point variables. Each entry name is a variable identifier, and its value is a decimal number.  |
| memory.Float.<variable_name>   | float         | val: 3.14                  | A configurable floating-point variable stored in memory.                                                              |
| autopilot                      | object        | —                       | Configuration parameters for communication with the autopilot system.                                                 |
| autopilot.ip                   | string        | "0.0.0.0"             | IP address used for the autopilot communication interface.                                                            |
| autopilot.port                 | integer       | 14551                 | Network port used for autopilot communication.                                                                        |
| autopilot.id                   | integer       | 1                     | Identifier of the autopilot system or vehicle within the communication protocol.                                      |                              | Additional attributes for the vehicle

Here's an example for a configuration of an `argoCopter` called `cop1` which does an `Exec` every 50ms and has a `Text` attribute `foo` with value `"a"`.

```
id: "cop1"
type: "argoCopter"
tick_rate: 50
rules: "/home/aislab/agent/abumon-goabu-agent/abu-config/rules.yaml"
memory:
  Text:
    id: "cop1"
    foo: "a"

autopilot:                  
  ip: "0.0.0.0"
  port: 14551           
  id: 1
``` 
### Rule configuration
The set of rules used by the vehicle is indicated in the node configuration.
The YAML file containing the rules should have the following structure:

```
rules:
  rule:
    name: <string>             # required (identifier for the rule)
    event: <string>            # required (the event that triggers the rule, e.g. "init", "mode", "altitude", etc.)
    condtype: <integer>        # required (the type of condition, 0 for local task, 1 for remote task)
    condition: <string>        # required (the condition to evaluate when the event occurs, e.g. "mode == \"GUIDED\"", "altitude > 10", etc.)
    action: <string>           # required (the action to execute if the condition is true, e.g. "set_mode = \"GUIDED\"", "set_arm = true", etc.)
  rule:
    name: <string>
    event: <string>
    condtype: <integer>
    condition: <string>
    action: <string>
```

This is an example of a set of rules to start a copter and make it off to 8 meters
```
rules:
  rule:
        "name": "InitRule",
        "event": "foo",
        "condtype": 0,
        "condition": "\"abc\" == foo",
        "action": "init = true"
  rule:
        "name": "ModeRule",
        "event": "init",
        "condtype": 0,
        "condition": "true == init",
        "action": "set_mode = \"GUIDED\""
  rule:
        "name": "ArmRule",
        "event": "mode",
        "condtype": 0,
        "condition": "mode == \"GUIDED\"",
        "action": "set_arm = true"
  rule:
        "name": "TakeOffRule",
        "event": "arm",
        "condtype": 0,
        "condition": "arm == true",
        "action": "take_off = 8.0"
```