## Map Layers

Map Layers helps to plan a Mission as well as observe its execution. There are 3
kinds of Map Layers grouped by preoperties:

* Network maps (GoogleMaps, BingMaps, OpenStreetMaps)
* Custom raster maps (user-added)
* Vector drawings
* Graticule
* Telemetry
* Markers Layer (detected objects, POIs, etc.)


### Network maps

In cases when Internet-connection is available for Triton, it can load map data
from publicly available map providers which usually helps planning the Mission.

During the execution of the Mission, on the other hand, Internet access might
be slow or it can be no Internet at all.

Therefore the idea of network maps is an ability to cache loaded mapping data
for later offline reuse. Currently in Neptus user needs to center map on area
of interest and initiate caching process. In that case all tiles for visible
area (plus couple of zoom levels below and above) will be cached. Maybe we can
design an UI for similar logic. On other software (Maperitive) it is
implemented with one extra step: user have to select desired rectangular area
and then initiate data download.

It is unclear wherer multiple network map layers can be visible at the same
time.  In pactice there are some mostly tranparent maps exist (e.g.
OpenSeaMaps), but its use is very unlikely.

In case if that will not overcomplicate UI, it makes sense add multiple Network
Maps layers: each for provider. Otherwise a single layer with selectable
provider is enough.


### Custom maps

Custom maps are typically some raster data that is somehow bounded to geodetic
location. For example it can be a scans of sea maps or contain mission-specific
telemetry (e.g. bathymetry).

One of custom maps format is *GeoTIFF*. Along with raster data it has special
metadata that describes its georeferencing (coordinates) as well as its
projection (a way, how 3-dimentionaldata are projectedona plane).

Just to demonstrate this, below is the ouput of GDAL utility:

```
    $ gdalinfo bathy_geotiff_AUV_CCZ.tif
    Driver: GTiff/GeoTIFF
    Files: bathy_geotiff_AUV_CCZ.tif
    Size is 6681, 3678
    Coordinate System is:
    GEOGCRS["WGS 84",
        DATUM["World Geodetic System 1984",
            ELLIPSOID["WGS 84",6378137,298.257223563,
                LENGTHUNIT["metre",1]]],
        PRIMEM["Greenwich",0,
            ANGLEUNIT["degree",0.0174532925199433]],
        CS[ellipsoidal,2],
            AXIS["geodetic latitude (Lat)",north,
                ORDER[1],
                ANGLEUNIT["degree",0.0174532925199433]],
            AXIS["geodetic longitude (Lon)",east,
                ORDER[2],
                ANGLEUNIT["degree",0.0174532925199433]],
        ID["EPSG",4326]]
    Data axis to CRS axis mapping: 2,1
    Origin = (-117.069703385523880,11.870067558131607)
    Pixel Size = (0.000009727805736,-0.000009583883614)
    Metadata:
      AREA_OR_POINT=Area
    Image Structure Metadata:
      INTERLEAVE=PIXEL
    Corner Coordinates:
    Upper Left  (-117.0697034,  11.8700676) (117d 4'10.93"W, 11d52'12.24"N)
    Lower Left  (-117.0697034,  11.8348180) (117d 4'10.93"W, 11d50' 5.34"N)
    Upper Right (-117.0047119,  11.8700676) (117d 0'16.96"W, 11d52'12.24"N)
    Lower Right (-117.0047119,  11.8348180) (117d 0'16.96"W, 11d50' 5.34"N)
    Center      (-117.0372077,  11.8524428) (117d 2'13.95"W, 11d51' 8.79"N)
    Band 1 Block=6681x1 Type=Byte, ColorInterp=Red
      NoData Value=nan
    Band 2 Block=6681x1 Type=Byte, ColorInterp=Green
      NoData Value=nan
    Band 3 Block=6681x1 Type=Byte, ColorInterp=Blue
      NoData Value=nan
```

The image mentioned above is available on [1].

In practice user might upload several custom maps and enable/disable or
rearrange them during operation.

Map upload dialog might have an image preview and display some parameters, such
as projection type, datum (geodetic coordinate system) name and extent (either
coordinates of upper-left and lower-right corners or coordinates of center).

In the data above the projection is not specified, datum is "WGS 84" (or in
long form "World Geodetic System 1984"). This stuff is mostly GIS-related, but
looking on some metadata before importing might save time. Importing procedure
contains converting the image to tiles and might take sometime (10'th of
seconds for large images).

In practice, such maps can be in a projection that can't be 1:1 converted to
tiles. In this case so called Warp procedure is necesssary. It distorts the map
in non-linear way and result can be wrong. Usually, the easiest way to verify
it -- find the same feature on other (e.g. network) map. For this purpose the
mapping widget itself is the best way to preview. This could mean that
Map-import dialog possibly needs to be a non-modal.

Additionally, it makes sense to store custom maps in general store of Triton
instead making ot a part of a Mission, such that multiple Missions can share
the same Map.

And final obvious thought about Custom maps: since user can add new maps, it
makes sense to let him delete or rename them :-)


### Vector Drawings

Georeferenced data can be represented in a vector form: contain Features with
its own metadata and georeferenced geometries. Geometry consists of:

* Points
* Lines
* MultiPoints
* Polygons
* MultiLines
* MultiPolygons

Such layers lets represent construction drawings on Mission Site, indicate
Points Of Interest or serve for other Purposes. There are several well known
formats for this purpose:

* GeoJSON -- simple and flexible way to represent geospatial data in browsers
  (this format will be used under the hood for all other imported vector layers)
* Shapefile -- container of georeferenced data (that might contain even raster
  images inside), typically produced by various GIS software
* DXF -- an open and documented format for CADs

Potentially all geometries that user created in Drawing mode in Triton, can be
exported as GeoJSON and later reimported into other Mission. This create
thoughts about UI that lets editing of any Vector Layer. And changing overall
logic of Drawing in order to make it layer-dependent.


### Graticule

Graticule is a set of lines of latitude and longitude. Ideally it is driven
above all Raster layers and below all Vector map layers.

A useful property of this layer is ability to tune color and transparency of
lines/text as well as background. Raster maps might have bunt colors which
makes difficult to see primitive geometries on a maps. Semi-transparent
background will suppress the palette and solve the issue.

From the point of view of UI/UX it can make sense to separate graticule and
shading layer.

The graticule layer might have parameters, such as Projection and Datum.

Technically all maps are visualized in WebMercator projection, but if user works
in UTM or in local Datum -- graticule helps to find mappings beween local view
and desired coordinate grid.

WebMercator is quite fast, simple and widely supported way to show Map data in
Web, but it doesn't support showing polar areas. In case of Triton, we need it.

Suggestion to handle that is introduce 3 modes of Maps visualization and let
user change the mode:

* Standard   -- for Latitudes in range 80°S..80°N
* North Pole -- for Latitudes > 70°N
* South Pole -- for Latitudes > 70°S

It is potentially possible to perforn maps warping into a desired projection on
the fly, but this will consume a lot of performance and battery.


### Telemetry layers

One of purposes of Vehicles is to collect georeferenced data. In such cases the
visualization of collected data helps user find whether additional action on
mission site are necessary.

Depending on kind of data it can be displayed as HeatMap (in case of scalar
observation such as Temperature, Salinity, Depth, etc.). In more complex data,
such as Side-Scan sonar a special logic is necessary which makes these special
layers look differently.

User needs to select the type of Telemetry to display on a layer. It is unclear
whether it makes sense let user have more than one Telemetry type at a time.

Optionally telemetry layer might produce either tooltip or show the data in
status for the location user holding mouse cursor over: e.g. depth at certain
location.


### Markers layers

During the operation additional georeferenced information may appear. It can be:
* Detected objects
* Acoustically observed position on the Vehicle underwater
* etc

Markers can be a simple icons (circle) as well as more complex objects:
thumbail of detected object.

These layers might require certain interactivity, such as tool-tips or even more
a tool-tips that can be pinned and will stay until closed. It will show some
information regarding corresponding marker (e.g. coordinates of the Vehicle).


### Projections and Datums

Datum is set of parameters that defines how to describe a point on a surface
Earth. It contains an Ellipsoid parameters: its center, axes dimensions and
rotation.

A global Datum that fits whole Geoid with minimal error and is used by GPS is
called WGS-84. However there are datums that fits Geoid much better in certain
areas (countries) and much worse in rest areas. Such Datums are called local.

Any geospatial data is represented in certain Datum. Using wrong Datum in
visualization introduces error (sometimes great).


Map projection defines how a point from a globe is transfered on a surface.
There are many projections exist. Most notable are:
* Mercator
* Universal Transverse Mercator (UTM)

In Triton a list of known Datums and Projections will be retrieved from Server.
This list is restricted by GDAL -- a library that performs various operations
with maps.


## Mission planning

### Definition of Maneuver

In Neptus, Plan is list of Maneuvers that are executed by the Vehicle
consequently. Mission might contain multiple Plans, but need to have at least
one Plan that is always known by the Vehicle and is being executed
automatically in case if the Vehicle losts communication with a base-station.

Every single Maneuver contains one of several georeferenced waypoints (in
geodetic coordinates) which have additional parameters such as speed. Examples
for Maneuvers are:
* "go-to" location
* Raster: a rows-maneuver of rectangular form with various raster parameters
* arbitary non-selfintersecting polygon that is filled with rows similarly to
  Raster Maneuver
* Station-keeping: similar to "go-to", but the Maneuver doesn't finishes when
  Vehicle reaches the point. It stays active and as soon the Vehicle leaves
  close area it executes another "go-to" to the same location

Maneuvers needs to have editing capabilities. At least the way to drag it over
the Map in simplest case. In more complex cases it might be a polygon editing
or rotation of the Maneuver.

_TODO: Exact set of parameters for each Maneuver is unknown to the author at
the moment of writing this document. It can be obtained from screencasts of
mission planning in Neptus. Looks like that every maneuver has Action-groups --
a set activities that are executed in the beginning and the end (and maybe in
case of termination) of the Maneuver: such as enable SideScan-Sonar or command
Winch to deploy/recover a probe. This things are tightly bounded to
architecture of DUNE which are unclear._


### Maneuvers reordering

Additionally it makes sense to make available reordering Maneuvers in Plan.
When one Maneuver is finished, the next is executed immediately. And if we
haveaplan that looks like a:
* Raster with side-scan
* Go-to back to launching point
user might notice that initial maneuver is Missing and want to add a go-to
Maneuver to launching point before executing the Raster.

In practive the Vehicle might be anywhere at the moment of Plan execution, and
in order to make reproducible datasets it makes sense to have a plan:
* Go-to launching point
* Raster with side-scan
* Go-to back to launching point

For whis operations User needs to either duplicate the last Maneuver or creare
a new one and then reoder it in Plan and make it first.

In some cases there is no need of such "first" Maneuvers and/or starting point
as it might have no sense: e.g. Maneuver that commands a Vehicle to move at 40%
Throttle on heading 30° during 20 seconds.


### Vehicle capabilities / Maneuver Requirements and Constraints

Some parameters might be available for Vehicles with certain capabilities. For
example, underwater maneuvers will have either depth below surface oraltitude
above ground for a transit in time when for surface Vehicles it is not
appliable.

This means, that every Maneuver creared for specific Vehicle type. Maybe it
makes sense to define Maneuvers for each Vehicle type even if most of them are
duplicated. From the point of view of UX, it will be the same, but technically
simplifies parameters logic.


### Maneuvers-groups / Common parameters and its inheritance

Maneuver's dragging and rotation over the Map might be useful to apply to a
group of maneuvers at the same time as well as its duplicate (copy-paste).

For this purpose a term "Maneuvers-group" can be introduced.

User can select some adjacent Maneuvers and group them. After this operation
this group will represent a single Maneuver in Plan.

Additionally Maneuverg-group might override parameters that are common to all
grouped Maneuvers (e.g. Speed). If user requests such override in a group, all
or selected submaneuvers will inherit this parameter of the group.

This solves the problem when on complex Plan, user needs to change the
parameter(s) on each Maneuver. At the same time this model leaves ability to
leave the parameters intact on some submaneuvers: e.g. the Speed in Group
defines how fast the Vehicle will collect telemetry, but will not affect
transitional Speed between more "important" parts of the Plan.

[1] https://nas.evologics.de/sharing/XEYZHkUoL
