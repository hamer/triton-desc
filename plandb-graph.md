# PlanDB exchange between Server and Client(s)

Here only communication in irder to create, retrieve, update and delete Plan
components is described. These operations affect Graph only.

Description and specification of other aspects (such as manever types,
conditions of transition, actions of maneuvers and the plan itself) ar out
of scope of this document.

Core block of Plan transfering is

```
PlanContainer = {
    "plans": [ PlanView, ... ],
    "transition": [ TransitionView, ... ],
    "maneuver": [ ManeuverView, ... ]
}
```

where:

```
PlanView = {                    // Plan overview
    "id": int,                  // ID of the plan
    "name": string              // User-Friendly name
    "initialManeuver": int,     // ID of the Maneuver or 1 if there is no
    ...
}
```

```
TransitionView = {              // Single Transition overview
    "id": int,                  // ID of Transition
    "name": string              // User-Friendly name
    "planId": int,              // ID of the plan this Transition belongs to (or 1 if not assigned to plan)
    "sourceManeuver": int,      // ID of previous maneuver
    "targetManeuver": int,      // ID of next maneuver
    ...
}
```

```
ManeuverView = {                // Single Maneuver overview
    "id": int,                  // ID of Maneuver
    "name": string              // User-Friendly name
    "type": int                 // Maneuver type
    "planId": int,              // ID of the plan this Transition belongs to (or 1 if not assigned to plan)
    ...
}
```

## Requests

### Get

To retrieve all plans a "get" request is used. There are no arguments are
required. Response is just a PlanContainer.


### Plan modification

To modify plan there are 3 operations possible: deletion, creation and modification.
A type PlanContainer can be used to enumerate all parts to delete as well as separate
PlanContainer can contain a set of plan components to modify:

```
PatchPlanContainer = {
    "patch": PlanContainer,       // Things to modify
    "delete": PlanIdContainer     // Things to delete
}
```

where

```
PlanIdContainer = {
    "plans": [ int, ... ],
    "transition": [ int, ... ],
    "maneuver": [ int, ... ]
}
```

This container can be sent in a "patch" request. Reply should be a
PlanIdContainer in case of success. Otherwise it should have an error
message.

PlanIdContainer that is response to patch request should contain ID's of all
entries that should be cleaned up from local plan copy (i.e. negative ID's).

### Entries creation

Previous section describes the process to modify or delete existing entries in
PlanContainer, but does not the process of creation of Entries.

Single "patch" call should commit valid plan at once, so new entries need to be
created during that commintment. On the other hand they need to be referenced
other entries: e.g. newly created maneuver needs to be referenced by some
transition, or some plan might reference it as initial maneuver.

In order to achieve that a special values ID needs to be used. To understand
which values can be used for this purpose the propertied of IDs that can be
assigned by the database needs to be used:

Database assigns values, that are monotonically increasing and begins from 1.
The value 1 is reserved as special value.

Before uploading to backend, frontend may use negative ID's to address parts of
plans. This allows to establish connections between them and it clearly
indicates which parts are new.

The logic on backend should replace negative ID's with fresly created ones and
notify clients with real, existing ID's and reply to client that make the
request with set of used negative ID's to clean them up.

Original client, who has issued such change will receive an update from backend
and create new entries and overwrite references to new entries, and then clean
up older stuff by handling reply from backend.


#### Multiple clients editing

It might happen, that two clients, who use the same mission might edit plans
simultaniously, and thus they both periodically receives updates with update
events.




## Notifications

When Plan is being changed -- all clients should be notified with "plan" event.
It should send PatchPlanContainer which describes plan modification.
