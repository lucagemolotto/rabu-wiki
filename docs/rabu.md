# Robo-AbU: A DSL for Event-Driven Robotic Systems

Robo-AbU is a variant of `goAbU`, the original Golang implementation of the [AbU calculus]().
It provides a way to program robots via Event-Condition-Actions (ECA) rules, building event-driven and reactive robotic systems.
AbU also abstracts away inter-agent communication by using *remote tasks*, which are actions acting on *remote* nodes, even though the rules is triggered locally.

## Syntax
AbU rules take the following structure:
```
rule <RuleName>
on <event>
for / forall <boolean condition>
do <assignments>
```
meaning that rule `<RuleName>` is triggered by the change of value of attribute `<event>`.
If we used the keyword `for` and the boolean condition holds locally, it will results in a series of assignments, to local attribute.
If, instead, we used keyword `for all` the assignments will be done on all remote nodes that satisfy the boolean condition.
We call task the couple made up of the condition and the assignments (actions).

The method evaluation of the tasks depends on whether we are using *eager* or *lazy* evaluation, more details [here](contributing.md#lazyevaluation).
## Getting started
To start writing Robo-AbU code you need 5 things:
* A `RoboAbuAgent`, the Transaction Manager, see [here](contributing.md#agent) for which are available.
* A `RosResource`, the I/O Manager, which might also need a related `Vehicle`, see [RosResources](contributing.md#rosresources)
* A slice of AbU rules, one rule per string
* A `RosExecuter`, which takes as arguments the agent, the resource and the slice of rules. Additionally it takes the global namespace for communication (the default is `/aburos`) and the type of evaluation.

Then all we need is calling `Exec` and/or `Input` on the `RosExecuter`.

## Example
In this example we will spawn two Copters, `cop1` and `cop2` which will be controlled by AbU rules and coordinate with each other.
We start by making the two
```
	cop1, err := vehicles.NewCopterGoROSetta("copter1", "", nil, nil)
	if err != nil {
		//return fmt.Errorf("failed to create executer: %v", err)
		fmt.Println("failed to create vehicle: %v", err)
	}
	mem := ros_resources.NewCopterResource(cop1)
	mem.Text["foo"] = "aaa"
	mem.Bool["detection_alarm"] = false
	mem.Float["base_alt"] = 0.0

	cop2, err := vehicles.NewCopterGoROSetta("copter2", "", nil, nil)
	if err != nil {
		//return fmt.Errorf("failed to create executer: %v", err)
		fmt.Println("failed to create vehicle: %v", err)
	}
	mem_cop := ros_resources.NewCopterResource(cop2)
	mem_cop.Text["foo"] = "aaa"
	mem_cop.Bool["detection_picker"] = true
	mem_cop.Float["detectionfound_lat"] = 0.0
	mem_cop.Float["detectionfound_lon"] = 0.0
	mem_cop.Bool["detectionfound"] = false
	mem_cop.Float["base_alt"] = 0.0
```
We declared the vehicles, then the resource and add some attributes which will come in handy for the logic of our system.

Now we declare the rules. InitRule is used as the starting trigger of our vehicles, putting them in `GUIDED` mode.
Then we arm them via `armRule` and take off via `takeoffRule`.
```
	initRule := `rule InitRule on foo for "abc" == foo do set_mode = "GUIDED"`
	armRule := `rule ModeRule on start for mode == "GUIDED" do set_arm = true`
	takeoffRule := `rule TakeOff on arm for arm do take_off = 5.0`
```
Now comes the import part. `moveRule` waits for the vehicle to have taken off at a good enough altitude and moves by in a Front-Right-Down (FRD) frame by vector (-5m,0m,2m).
If at any time, `cop1`'s built-in object detection sensor is triggered, it sends a message to all copter capable of picking the object (in our case, `cop2`). At this point, `cop2` will take off and reach the coordinates sent by `cop1`.
```
	moveRule := `rule MoveRule on altitude foo for "GUIDED" == mode && altitude > 3 && foo == "abb" do move_x = -5.0, move_y = 0.0, move_z = 2.0, foo = "qwe"`
	alarmRule := `rule AlarmRule on detection_sensor for true do detection_alarm = detection_sensor`
	globalRule := `rule detectionRule on detection_alarm for all true == ext.detection_picker do ext.detectionfound = true, ext.detectionfound_lat = position_lat, ext.detectionfound_lon = position_lon`
	copterRule1 := `rule detectionTakeOffRule on detectionfound for detectionfound do foo = "bca" `
	copterRule2 := `rule GoTodetectionRule on detectionfound altitude for base_alt > 0 && altitude > 3 && detectionfound do setposition_lat = detectionfound_lat, setposition_lon = detectionfound_lon, setposition_alt = position_alt`
```
Finally, we make some rules for landing and putting the vehicles back in their default mode.
```
	landRule := `rule LandRule on foo for "cba" == foo do set_mode = "LAND"`
	landRule2 := `rule LandRule2 on altitude for "cba" == foo && altitude == 0 do mode = "STABILIZE"`
```
Now we make the transaction managers and the executers.
```
	cop1_agent, _ := NewRosAgent()
	cop2_agent, _ := NewRosAgent()

	executer, err := NewRosExecuter(mem, []string{initRule, armRule, takeoffRule, alarmRule, globalRule, landRule, landRule2, takeoffRule}, cop1_agent, "copter1", "/aburos", "lazy")
	if err != nil {
		//return fmt.Errorf("failed to create executer: %v", err)
		fmt.Printf("failed to create executer: %v\n", err)
	}
	fmt.Println("Created first executer")
	executer2, err := NewRosExecuter(mem_cop, []string{initRule, localRule2, localRule3, takeoffRule, copterRule1, copterRule2, landRule, landRule2}, cop2_agent, "copter2", "/aburos", "lazy")
	if err != nil {
		//return fmt.Errorf("failed to create executer: %v", err)
		fmt.Printf("failed to create executer: %v\n", err)
	}
	fmt.Println("Created second executer")
```
Lastly, we loop the executor and use inputs for interacting with the system with timers.
```
	time.Sleep(1 * time.Second)
	fmt.Println("Input start")
	time.Sleep(2 * time.Second)
	executer.Input(`foo = "abc"`)
	executer.Input(`foo = "bca"`)
	executer2.Input(`foo = "abc"`)
	fmt.Println("Input done")

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
	time.Sleep(6 * time.Second)
	executer.Input(`foo = "abb"`)
	state4, _ := executer2.TakeState()
	newalt := state4.Float["altitude"]
	fmt.Println("newalt", newalt)
	time.Sleep(30 * time.Second)
	executer.Input(`foo = "cba"`)
	executer2.Input(`foo = "cba"`)
	time.Sleep(60 * time.Second)
```
## Supported vehicles
* ArduCopter
* ArduPlane (WIP)
* ArduRover
* ArduSub

## Physical Attributes
| Attribute                            | Components     | Type |        | Example Value           | Description                                                                                                           |
| -------------------------------- | --------------- | ----------------------- | ---- | --- |
| move | move_x, move_y, move_z | Output - Float | (3.0, -1.1, 2.4) | Forward-Right-Down (vehicle-centric) movement of the vehicle in meters |
| mode | - | Input - Text | "GUIDED" | Current Autopilot mode of the vehicle | 
| set_mode | - | Output - Text | "GUIDED" | Sends to the autopilot the desidered mode |
| arm | - | Input - Bool | false | Current arming state of the vehicle's motors |
| set_arm | - | Output - Bool | true | Sets the arming state of the vehicle's motors |
| altitude | - | Input - Float | 5.0 | Altitude of the vehicle relative to the ground (Copter/Plane only) |
| take_off | - | Output - Float | 5.0 | Altitude of the vehicle relative to the ground (Copter only) |
| depth | - | Input - Float | 3.2 | Depth of the vehicle relative to the water surface (Sub only) |
| position | position_lat, position_lon, position_alt | | Current GNSS coordinates of the vehicle. Latitude and longitude are specificied in WGS84 and multiplied by 10e7. Altitude is meters above average sea level (ASL). |
| set_position | setposition_lat, setposition_lon, setposition_alt | | Sends to the autopilot the desidered GNSS coordinates of the vehicle. Latitude and longitude are specificied in WGS84 and multiplied by 10e7. Altitude is meters above average sea level (ASL). |