# Entity State API

## Data Types

Entities are transfered in terms of Entity object described below.

```
Entity = {
    id:       number;
    vehicle:  number;
    entityId: number;
    name:     string;
    desc:     string;
    state:    "boot" | "normal" | "fault" | "error" | "failure" | "unknow";
    timeStamp: number;
}
```

At a first step, backend will have a list of Entity names and IDs. Later it
could get entity state information.


## Logic on a backend side

On EntityList reception -- VehicleState collection of entities has to be
processed:

* new entities should be added to it
* entities that are already present there should be updated (name might be
  changed)
* rest of entities that are not present in EntityList IMC message should be
  considered as removed and should be excluded from entities collection in
  VehicleState

The Agent should react on EntityList message and use VehicleState -> entities
information and compare it to local copy of Entity States (e.g. taken from
Database) to find which entities were removed.

When Logic finds that certain entity was removed -- it adds its ID to a
separate array that will be sent to a Frontend side as a "entities-remove"
notification.


## Quirks

Since there is not database support availalbe yet we'll support only a single
Vehicle and use entityID as id in Entity object.

Implementation of entries removal in Observable Collection is a separate task
and might be omited on a first approach.
