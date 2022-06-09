# Parties Announce and Discovery

Every Party (a node in network, an instance DUNE, Neptus or Triton) finds other
Parties by sending a specific Multicast group. The communication for this
purpose is done over UDP-socket that is bound to port in range 30100..30104.
This means that every message ist sent to every port in specifief range (e.g. 5
times) on IP-addres 224.0.75.69. This address is in range of multicast
addresses which means, that it will be delivered to every host in network, that
has a subscription to a Multicast-group with this address.

NB! An IP-address fir Multicast group and range of ports specified above are
typical values that are being used by DUNE/Neptus. It needs to be somewhere in
configuration file to let Manufacturer modify these values for customization
purposes.


## Port binding

The Server needs to try to bind an UDP-socket to one of ports in specified
range. The socket needs to have an option SO\_REUSEADDR option to let multiple
instances of DUNE, Neptus ot Triton to bind the same port on the same Host.

In case when no ports from the range are available to bind (in POSIX there are
reasons: EADDRINUSE if port is bound to other process without SO\_REUSEADDR; or
EACCESS if port is priviledged, so only administrator can use it) — the Server
can't find any other Parties and should fall in Error state and make periodic
attempts to bind a port again User should know about this problem in UI and
logs.


## Announcement process

Every Party in network sends its Announcement messages periodically. The format
of announcement is an IMC-Message of the type Announcement. This message has:
* name of a system (sys\_name), e.g. "Sonobot-35";
* an IMC-address of the node (src);
* set of services as ';'-serarated set of URLs
* geodetic position if available (lat, lon and height)

Typical parameters from DUNE Announcements are:
```
    m_announce_loc.setSource(getSystemId());
    m_announce_loc.setDestination(0);
    m_announce_loc.owner = 0xFFFF;
    m_announce_loc.sys_name = getSystemName();
    m_announce_loc.sys_type = translateSystem(m_args.sys_type);
```

An example list of services for DUNE is:
```
dune://0.0.0.0/version/$DUNE_VERSION   <- version of DUNE
dune://0.0.0.0/uid/$UID                <- unique ID of the instance
imc+info://0.0.0.0/version/$IMC_VER    <- version of IMC-definitions used
imc+udp://192.168.0.50:6002            <- all known IP addresses, that are
imc+udp://10.0.0.50:6002                  announced to be available by UDP
imc+tcp://192.1868.0.50:6001
imc+tcp://10.0.0.50:6001
```

Neptus uses similar Announcement:
```
neptus://0.0.0.0/version/$NPTS_VERSION <- version of DUNE
neptus://0.0.0.0/uid/$UID              <- unique ID of the instance
imc+info://0.0.0.0/version/$IMC_VER    <- version of IMC-definitions used
imc+udp://192.168.0.51:6002            <- all known IP addresses, that are
imc+udp://10.0.0.51:6002                  announced to be available by UDP
```

Unique identifier is usually a timestamp of server launch and is used to find
whether the Party is restarted since its last discovery. Some actions for
Agents in Triton might be necessary in this case. For example Entity IDs might
be changed (by changing the config or race condition in dynalic ID allocation).

Parts with imc+udp:// or imc+tcp:// scheme define addresses to communicate with
nodes. The Server needs to enumerate all network interfaces on its host and add
corresponding service to the list.

In pracice UDP-only connection is used between the Vehicles and Control station.
Lines with imc+tcp:// scheme are added as example only. However on first
approach Triton will use a hardcoded TCP address to communicate with a Vehicle.

Serialized IMC message with with Announce needs to be sent to all ports in
announcement range (30100..30105) on Multicast address (224.0.75.69).


## Communication over UDP with Vehicles

After reception and parsing an Announcement Triton needs to send Heartbeats
(an IMC message) to all "active" contacts periodically (10 seconds) to keep
the contact on the remote side "active".

It is also expected that remote side will send Heartbeats periodically. At
least it is necessary to receive anything from remote Party to refresh the
activity status of the Vehicle. Otherwise, when no data from the Vehicle
within long period (15..30 seconds) received, the connection should be
considered as lost.

NB! It might happen that DUNE on a Vehicle is restarted fast and therefore
Triton receives an Announcement with different UID (unique identifier) while
the UDP-Transport contact is still remain "active". In this case Triton needs
to execute Initialization procedure with the Vehicle anyway.

Every Agent should know when a Vehicle is discovered and/or becomes active or
inactive. For example, PowerSwitch Agent queries Power-Switch channel list
after connection to a Vehicle. It makes no sense to query this list every time
the Vehicle becomes reachable (or "active") unless the DUNE is restarted (and
thus its UID is changed which mean that configuration and/or entities ID might
be also changed).


## Communication layer Architecture

Discovery mechanism might be useful by UI Plugins in order to notify them
about change the status of the Vehicle or let them query more specific
information about Vehiche that is generic enough to keep it in VehicleMonitor
Agent. For example every Vehicle has its own Entities List where every Entity
has an ID, a Name and Status (BOOT, ONLINE, ERROR and maybe others).

Entities are abstract things, but represents source and destination address
of an IMC message within one Node (similarly like a Port-number defines the
process who will receive a TCP or UDP packed received on network interface
on host with specific IP address).

Core functionality needs to be represented as a Singleton instance, which role
is:
* implement Discovery mechanism (handle Announce'ments on separate socket)
* open connections to discovered vehicles (a single UDP socket might be
  sufficient for this purpose)
* watch for vehicles to be active (track Heartbeats)
* keep connection with vehicles active (send Heartbeats back)
* implement a low-level interface to send IMC messages to active vehicles
* properly handle incoming messages from vehicles

Other part needs to be a VehicleMonitor Agent. Its role:
* store discovery and vehicle status (active/inactive) events in database
* query EntityList if needed after vehicle discovery
* implement API for WebSocket to query collected data and/or subscribe to
  status changes notifications

NB: Neptus has more complex devinitions of Vehicles: it describes icon used for
the Vehicle, a set of supported Maneuvers, parameters for remote control (an
axes for control similarly to axes on Joystick) as well as hints how to connect
to the vehicle. In Triton this is needed as well, but it is planned to add an
IMC messages (request and reply) that lets get this data directly from the
Vehicle. A library of such definitions is needed in Triton anyway, but for the
first approach let's assume that all Vehicles are the same, supports ALL
maneuvers and etc. However a placeholder for Vehicle metadata would be nice to
have.


### Database scheme for VehicleMonitor Agent

Every time a new Vehicle instance is discovered, it is at least for EntityList
queried. A dictionary of Entities needs to be stored in a database, such that
every unique Vehicle has a history of this dictionary.

If the Vehicle is rebooted a new version of Entities list is stored.
Per-Vehicle vesion of the Dictionary is stored with a timestamp. With that it
is possible to look-up and find correspondence between an ID and Name of an
Entity for any moment in time: either actual (latest version) or in the past.

Vehicle status changes needs to be stored in a database as well.
