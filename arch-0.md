# Agents

Agent implements handling the specific logic regarding the task:

* it processes IMC messages in its own way
* stores additional data in database
* handles communication with clients

## Agents instantination

Every Agent is a singleton that might be instantinated if it was specified in a
Server configuration. During instantination the Agent gets an ID that is unique
per application. It can be that the same Agent will get different IDs in a
separate launches of Server.

In this model, the Agent is not bound to specific Vehicle or Plugin/Client.
Agents handles all available Vehicles as stateless as possible. In case if
multiple Vehicles sends similar data (e.g. Motor RPMs) — Agent notifies all
clients who are interested (subscribed) on this kind of events.


## Handling IMC Messages

Generic IMC handler typically parses a Message and store the Message-BLOB along
with some parsed data in a Database.

A type of IMC Message can be identified by a field in this message. The handler
may refer a Map (or Hash) where Key is the number that represents the typeof
the message and Value is a List of Agents that handles it.

In this case every Agent has a method handleIMC(IMCMessage msg) and his
constructor "registers" himself to designated IMC Message type in Generic IMC
Handler.

NB: Perhaps this logic can replace handling of specific IMC messages we
discussed earlier. Agents might create necessary tables in Database if they're
absent and write telemetry or other information as they will.


## Communicating with Clients

Similarly as with IMC Message handlers, Agent's constructor registers itself
in a list of available agents. Such that Clients can query a list of available
Agents: `{ id: 7, name: "power-switch" }` as well as add itself in a Map where
its unique ID is mapped to an Agent-instance that has a method
handleClientRequest(string name, ReqArgs arguments) and returns asynchronous
result: Uni<Resp>.

Client might issue a Request:
```
{
    id:      512, // unique per-client per-request id
    command: "upload", // took any of available commands except start/stop
    type:    "agent",
    data: {
        id:     12,             // agent ID,
        cmd:    "get-channels",
        args: {
            vehicle: 1234,      // ID (address) of the Vehicle
            entity:  4321,      // entity ID that represents Power Switch
            timeline: null      // null — actual information, otherwise it
                                // should be a UNIX time
        }
    }
}
```

In the example above the method of PowerSwitchAgent instance
handleClientRequest("get-channels", JsonNode args) needs to be called.


## Subscriptions and Notifications

Agent might push Messages to Clients. In order to do that it need to store
a list of Session's (WebSocket connections). Agent's abstract class might
implement handlers for START/STOP commands such that if "agent" endpoint
for WebSocket receives a request:
```
{
    id:      512, // unique per-client per-request id
    command: "start", // "start" subscribes in event, "stop" unsubscribes
    type:    "agent",
    data:    12 // agent ID
}
```

it adds the corresponding Session to the list of subscribers to an Agent.
Of'course the Agent needs somehow clean broken session up (e.g. check before
transmission and remove if outdated).

The method notify(JsonNode) sends the serialized data to all subscribed
recipients.


## Handling TimeLine

It is potentially possible to have multiple Client connections to a Server.
One of Clients might switch to TimeLine mode to review the state in the past.

In this case it needs to unsubscribe from all events with all Agents and
request historical data from an Agent(s).

This operation is usually stateless: a request(s) to Database and get results
back. The Agent needn't to switch in any other mode. The only difference in
querying actual and historical data is the request to Database: query the
latest data or the data that is not newer than specified timestamp.


## Detection of Vehicles

Plugins might want to know whether the Vehicle goes Online/Offline or a new
Vehicle is detected. For example for a plugin that shows and controls
Navigation Lights on ALL available Vehicles.

For this purpose an Agent "vehicle-monitor" might notify Clients about such
events.


# Client Plugins

Plugin implements specific UI/UX logic to observe and control processes such
as:
* observe Side-Scan Sonar data
* control Power channels
* observe Thrusters and Servos status as well as issue commands (homing for
  Servos)

Every (almost every) Plugin has its own widget that is located in a Layout. The
exact way of storing and configuration the Layout is out of scope of this
document.

NB: Future Plugins might implement Map Layers and TimeLine lines. That's why
"almost every" is present.

The primary idea is that Plugin implements a component that interacts with User
from one side and Agents on the other.


## Instantination

Unlike Agents, Plugins might have multiple instances of the same Plugin. For
example a Mission has multiple Vehicles and each of them has a Side-Scan Sonar.
It might be useful to observe the data from both Vehicles simultanioussly and
thus locate two instances of "Side-Scan Sonar View" in one screen.

This model adds a limitation to implementation Angular components: it might be
difficult to use a Dependency-Injection for components with such life-cycle.


## Communication with Server

Plugins communicate with Agents indirectly. A Service PluginMultiplexer is used
as a wrapper for that. The reason is the fact that the Client might have
mupltiple Plugins that are subscribed to the same Agent.

Communication between Server and Client is done over WebSocket. Agents have no
idea which Plugins are present on every Client and from their perspective the
Client itself subscribes on events and executes commands.

For this pupose a PluginMultiplexer is needed. It tracks subscriptions from all
Plugins and when detects attempt to subscribe to an Agent which is already done
by other plugin — it handles that internally.


## List of available Agents

Available Agents list is constant duringa whole session with a Server. Thus the
PluginMultiplexer requests Agents list right after establishing the WebSocket
connection and replies on getAgents() methow with a list:
```
[
    { id: 1, name: "power-switch" },
    { id: 2, name: "vehicle-monitor" },
    ...
]
```
or in form of object:
```
{
    "power-switch": 1,
    "vehicle-monitor": 2,
    ...
}
```

NB: This request might take a while, so the result needs to be handled
asynchronously. (Observer?)

In case if Plugin needs an Agent that is absent in List — it shows an error. An
Error resets in case when a WebSocket connection gets either lost or
established. (For the Tracer bullet this logic is unnecessary).


## Plugin/Agent communication

PluginMultiplexer implements following methods:
```
execute(agentID: number, command: string, args: any) -> Promise<any>;

subscribe  (agentID: number, handler: (name: string, data: any) -> void) -> Promise<any>;
unsubscribe(agentID: number, handler: (name: string, data: any) -> void) -> Promise<any>;
```

NB: It is likely rxjs lets to pass handlers for subscribe in more beautiful
way. For example PluginMultiplexer implements a Producer and Plugins get their
Consumers.


## Sample Plugin logic

The logic is being descrideb in a context of Power-Switch plugin. The purpose
of this Plugin is to:

* let User select one of all available Power Switches (Vehicle might have one,
  multiple or no switches)
* show information about all power channels that are controlled by selected
  Power Switch (e.g. Navigation Light, Thrusters, Payload CPU):
  - the name of channel
  - its state (ON/OFF/ERROR)
  - control to issue the change of state (turn OFF or ON)


### Power-Switch selection

When User wants to select a switch a method getSwitches() executes command
"get-switches" for "power-switch" Agent. The response will look like:

```
[
    {
        vehicle: 32768, // address of the Vehicle
        entity:  2048,  // Entity-ID of the Power-Switch
        name:    "Main Power-Switch"
    },
    {
        vehicle: 32768,
        entity:  2049,
        name:    "Payload Power-Switch"
    },
    {
        vehicle: 16384,
        entity:  256,
        name:    "Power-Switch"
    }
]
```

NB: It is assumed, that Vehicle names can be resolved by the Plugin itself with
help of "vehicle-monitor" Agent.

As soon as Power-Switch is selected, Plugin queries channels: executes
command "get-channels" for "power-switch" with specific Vehicle and Entity IDs
(see Communicating with Clients chapter above for example). A response looks
like:
```
[
    {
        entity:  4096,  // Entity-ID of specific channel
        state:   "off",
        name:    "Navigation Lights"
    },
    {
        entity:  4097,
        state:   "on",
        name:    "Left Thruster"
    },
    {
        entity:  4098,
        state:   "unknown",
        name:    "Right Thruster"
    }
]
```

It might be that query returns unknown state for some channels. In this case
all controls for this channel have to be blocked until a notification about
the state of this channel is received.

In order to get notifications, Plugin needs to subscribe to "power-switch"
Agent events and handle "state" Event for entities (4096, 4097 and 4098).
