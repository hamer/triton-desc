= Triton server architecture

== Terminology

Mission — Indepent and sufficient set of data that contains configuraion, logs,
plans and Plugin settings for set of Vehicles that collected data on
Mission-site.

Plugin — A part of Triton that solves a (single or multiple) task in User
Interface (GUI). It is usually represented as separate panel with controls
or a Map-overlay. Plugins works purely Frontend.

Agent — Counterpart of Plugins that runs on Backend and implements certain API
for Plugins.


== Common ideas

=== IMC

Every incoming or loaded IMC message is stored in a table `imc_raw` table with
proper EntityID, VehicleID, Remote timestamp and local timestamps.

Additionally, every incoming message is passed to an opetional handler that MAY
create records in other tables. This is needed to make an access to specific
data e.g. GPS Fix, SonarData, etc. The logic of handlers should be stateless
and the only side-effects allowed are writes in database.  Every record in
custom tables which refer certain IMC message SHOULD refer it in `imc_raw`
table.

Every type of IMC message should be able to represent as an Object in local
language (usually provided along with libimc).


=== DBA (Database abstraction layer)

==== Raw data queries

DBA allows queries against collected IMC messages within specified:
* the vehicle(s)
* entity(s)
* time frame

In this case `imc_raw` table is used and all messages are being parsed
on-the-fly.

It is recommended to avoid such queries as some messages require
heavy port-processing which can slow-down the TimeLine functionality.
*NB! Premature optimization warning*


==== Specific data queries

Some kind of data might be indexed by the database. For example, geospatial
data (GPS-fixes) can be indexed with R-Tree index to speed-up a lookop.

Other data might be stored for simpler/faster access to the data. For example
the vehicle parameters changing history can be useful for analysis. Some
post-processed data micht improve performance.

Anyway, every concrete Plugin-Agent or System-Agent will use only subset of IMC
message types and it makes sense to make a specific interface for every type.
It will help to avoid implementing similar logic on every agent.


==== General state

Information, such that Vehicle discovery, its description, its status might be
out of scope of IMC bus. But this information needs to be kept persistent.

For this purpose for every kind of such data a proper interface is required.

The list of interface needs to be for:
* Vehicles (discovered, dummy, formations, etc)
* Missions (those who are tracked by Triton)
* Agents (provides API for Plugins)
* ...


=== Plugin-Agent communication

All practical logic in Triton is implemented by Plugins and Agents.  Triton
core provides a general interface to Agents, such that:

* database abstraction
* network communication
* ...

An API for specific role (e.g PowerSwitch or SideScan) is provided by Agents.
Agents are instantinated according to configuration. One instance per Vehicle.

User can select which Agents he needs. Alternativaly Manufacturer can specify
which Agents are necessary for every supported Vehicle type. Every Agent has
its name (e.g. PowerSwitch) and unique ID made by DBA. The ID is needed for
Plugins to communicate the Agent.

Plugin manager on Frontend queries available agents after connection, and
receives any changes on available agents over time asynchronously.

Request:
```
{
    id: 32678,      // unique ID per Frontend and generated by Fronted
    cmd: "query-agents"
    args: []
}
```
Reply:
```
{
    id: 32768,      // should match id from Request
    rs: [
        {
            name: "vehicle-discovery",
            vehicle: 1,     // 1 — non-existent Vehiche
            id: 2
        },
        {
            name: "power-switch",
            vehicle: 7,
            id: 128
        }
    ]
}
```

Every plugin can issue commands for corresponding agent using its ID or
subscibe on messages of certain type. Subscription works through Plugin
manager.  This is necessary to multiplex subscriptions in cases when mupliple
Plugins are subscribed to the same message from the same Agent.

TODO: Where and when do we do *EntityID* filtering? Can the EntityID be found
offline?

Request:
```
{
    id: 32769,
    cmd: "subscribe",
    args: [
        128,        // Agent ID
        13,         // Subscripion ID (unique ID per Frontend and generated by Fronted)
        "voltage"   // Message-type
    ]
}
```
Reply:
```
{
    id: 32769,
    rs: null        // Result is an error-code (Error string or null on success)
}
```

And when corresponing event happens, Plugin manager will receive:
Notification:
```
{
    sid: 13,        // Subscription id
    data: {         // Data provided by an Agent (can be arbitary Object)
        channel: 0,
        value: 28.6
    }
}
```

In case of client disconnect (on losing WebSocket connection), all
subscriptions on a Backend are canceled. Frontend has to request subscribtions
after reconnection again.

Commands to Agents are being sent with message:
Request:
```
{
    id: 32770,
    aid: 128,               // Agent ID
    cmd: "channel-set",     // Turn the channel ON/OFF
    args: [
        0,                  // Index of channel on PowerSwitch
        0                   // New State of Channel
    ]
}
```
Some commands doesn't require Response.


=== Plugin instantination process

Right after instantination the Plugin has almost no information. It is possible
that Plugin knows the Vehicle it works with.

Plugin starts in _STARING_ state and display some-kind of progress-animation in
the beginning.

With all existing information the Plugin asks Plugin Manager the list of
available Agents. If all necessary Agents for Plugin purposes are available,
Plugin executes necessary Commands and establishes Subscriptions. After that,
Plugin turns into _ONLINE_ state and display its UI.

If necessary Agents are unavailable (e.g. incompatible Plugin or Vehicle
selected), it shows error indicating which exactly Agent is necessary to work.
Plugin turns into _ERROR_ state.

Agents-List update might affect plugins to change state from ERROR to STARTING.
