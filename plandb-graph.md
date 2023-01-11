# PlanDB exchange between Server and Client(s)

Here only communication in irder to create, retrieve, update and delete Plan
components is described. These operations affect Graph only.

Description and specification of other aspects (such as manever types,
conditions of transition, actions of maneuvers and the plan itself) ar out
of scope of this document.

Core block of Plan transfering is

```
PlansContainer = {
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
required. Response is just a PlansContainer.


### Plan modification

To modify plan there are 3 operations possible: deletion, creation and modification.
A type PlanContainer can be used to enumerate all parts to delete as well as separate
PlanContainer can contain a set of plan components to modify:

```
PatchPlanContainer = {
    "patch": PlanContainer,       // Things to modify
    "delete": PlanContainer       // Things to delete
}
```

This container can be sent in a "patch" request. Reply should be either NULL if
request is accepted or error (which is already controlled on a lower-level).

After applying the patch successfully -- all subscribers needs to be notified by
"plan" event;

In case if new entries of PlanView, TransitionView or ManeuverView are needed --
they need to be requested from a server.


#### New entries

In case if new Plan, Transition or Maneuver is needed a special request is
issued to a server: "create"

```
CreatePlanContainer = {
    "plans": int,               // Number of empty PlanView's to create
    "transitions": int,         // Number of empty TransitionView's to create
    "maneuvers": int            // Number of empty ManeuverView's to create
}
```

A response to this request contains PlanContainer with freshly created entries
only. Other clients are NOT notified with these while processing that request.

NB! It might happen that other client will issue a "get" request and retrieve
these fresh non-assigned entries -- there is nothing bad in that thing.

NB! In case if client for some reason will fail to complete his operations --
newly created entries will be left unused forever. Some mechanism of cleanup or
reuse them might be useful.


## Notifications

When Plan is being changed -- all clients should be notified with "plan" event.
It should send PatchPlanContainer which describes plan modification.

Practically that PatchPlanContainer might be the copy of the argument for "patch"
request if all database operations were completed successfully.


