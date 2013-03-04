---
layout: blog_entry
title: Raster-based Transit Accessibility
---

One common way to visualize transportation networks is with a directed
acyclic graph. To determine how widespread transit is we start from a
node in the graph (bus or train stop), doing a simple graph traversal.

However, loading and traversing the graph can take a long time. Depending
on the complexity of the transit system it can make it hard to run
certain real-time analyses. For instance [http://www.transitheatmap.com/heat.html?philadelphia](http://www.transitheatmap.com) can show heatmaps from around
the country in a reasonable amount of time, but they are far from
instantaneous.

Another way to qualitatively analyze these data is to use a raster based
solution. We're going to use a cost-distance operation to efficiently calculate
a transitshed for a given transit system. The basic idea is:

1. Create a blank raster covering Philadelphia
2. Assign each cell a "crossing time" based on walking speed
3. Rasterize bus/rail lines using the stop-to-stop time
4. Run a cost-distance operation from a starting point on the raster

### Tools

We're going to use GeoTrellis, as raster processing library that is
being built by Azavea.

### Creating the Base Raster

The general idea is to assign each cell in the raster a value
that is the time it takes to traverse the cell via walking or
transit. The first thing we need to do is define how big our raster
should be. Note that processing time for this algorithm is
**linear** in the size of the raster, regardless of the complexity
of the network.

{% highlight scala %}
  val phillyExtent = Extent(-8383693, 4845038, -8341837, 4884851)
  val cellWidthTgt = 30.0 // meters
  val cellHeightTgt = cellWidthTgt

  // We want to have an integer number for rows
  // and cols so we recalc cell width and height
  val yspan = phillyExtent.ymax - phillyExtent.ymin
  val rows = (yspan / cellHeightTgt).toInt
  val cellWidth = yspan / rows

  val xspan = phillyExtent.xmax - phillyExtent.xmin
  val cols = (xspan / cellWidthTgt).toInt
  val cellHeight = xspan / cols

  // Create a blank raster
  val raster = CreateRaster(phillyRasterExtent)

{% endhighlight %}

### Assigning the Walking Values

The walking raster that we're building specifically
assumes that each square can be traversed in the same amount of
time. This includes water and buildings. A great extension
would be to use the Philly land-use layer to cordon these areas.
Now we assume the average walking speed is 3.5 mph and assign it
based on our cell size:

{% highlight scala %}

  val walkingSpeedMph = 3.5
  val walkingSpeedMetersPerSec = walkingSpeedMph * 0.44704

  // Truncate to integers in seconds
  val walkingTimeForCell =
    (cellWidth / walkingSpeedMetersPerSec).toInt

  val phillyRasterExtent =
    RasterExtent(phillyExtent, cellWidth, cellHeight, cols, rows)

  // Assign a 'friction value' of walking to all of the cells
  val rasterWalks = local.DoCell(raster, _ => walkingTimeForCell)

{% endhighlight %}


### Rasterize Bus/Rail Lines

#### Extracting GTFS Data

Now that we have walking speeds we need to grab the bus routing
speeds and times. Once you grab the [http://www2.septa.org/developer/](SEPTA GTFS data) unzip it and take a look at the two files we're going to use:

{% highlight bash %}
$ head -n 3 gtfs/stop_times.txt
trip_id,arrival_time,departure_time,stop_id,stop_sequence
3509653,04:33:00,04:33:00,1721,1
3509653,04:35:00,04:35:00,17842,2
$ head -n 3 gtfs/stops.txt
stop_id,stop_name,stop_lat,stop_lon,location_type,parent_station,zone_id,wheelchair_boarding
2,Ridge Av & Lincoln Dr  ,40.014986,-75.206826,,31032,1,0
4,Roosevelt Blvd & Broad St - FS,40.018136,-75.148887,,,1,
{% endhighlight %}

Basically, we're going to load the *stop_times.txt* and then update
each *stop_id* with the latitude and longitude from the *stops.txt*
file. The GTFS object provides some useful parsing methods. We can
use the *stopstoCoords* methods to extra JTS *Coordinates*, used
to assemble the linestrings later.

{% highlight scala %}
// A stop list is a list of (seconds since midnight,
// Stop IDs) pairs
type Stops = Seq[(Double, Int)]

val trips:Seq[Stops] =
    GTFS.parseTripsFromFile("gtfs/stop_times.txt")
      .right
      .get
      .values

val stops = GTFS.parseStopFromFile("gtfs/stops.txt")
              .right
              .get

def stopsToCoords(trip: Stops) =
  trip.map(s => stops.get(s._2) match {
    case Some((lng,lat)) => {
      val m = latLonToMeters(lat, lng)
        new Coordinate(m._1,m._2)
      }
    case None => sys.error(s"bad stop $s")
  })

{% endhighlight %}

To determine the average speed of the route we can
calculate the total time taken to traverse the trip
and the total distance of any given sequence of points:

{% highlight scala %}
def stopsToTimeInterval(trip: Stops) =  {
  val t = trip.map(_._1)
  t.max - t.min
}

def dist(s: Seq[Coordinate]) =
  s.zip(s.tail)
   .map(a => a._1.distance(a._2))
   .reduceLeft(_+_)

{% endhighlight %}

Then apply it the GTFS data and create GeoTrellis features
(the LineString objects). Note that *epsg3857factory* creates
geometries projected with Web Mercator.

{% highlight scala %}

def stopsToLineString(trip: Seq[(Double, Int)]) = {
  val coords = stopsToCoords(trip)
  val distance = dist(coords)
  val time = stopsToTimeInterval(trip)
  val speed = distance / time   // meters per second
  val cellTime = (cellWidth / speed).toInt  // seconds

  LineString(
    util.epsg3857factory.createLineString(coords.toArray),
    cellTime)
}

val lines:Seq[LineString[Int]] =
  tripsInRange.toSeq.map(tripStops => stopsToLineString(tripStops))

{% endhighlight %}

The end result (*lines*) is an array of *Feature* objects where
each object contains the time to traverse the cell.

#### Rasterizing It

GeoTrellis has a built-in rasterizer that we can use. The basic
use case is to feed it a bunch of *Features* and a per-cell
callback to use whenever a cell is covered by a feature. The
callback provides the mutable raster data as well as the feature
and cell info (GeoTrellis uses a *callback type* instead of a
standard scala function *Function3* and higher are not specialized)

{% highlight scala %}

val cb = new Updater {
  def apply(c: Int, r: Int, g: LineString[Int], data: IntArrayRasterData) {
    val prev = data.get(c,r)
    data.set(c, r,
      if (prev == NODATA) {
        g.data
      } else {
        math.min(g.data, prev)
      })
  }
}

val busTimeRaster =
  server.run(
    RasterizeLines(
      phillyRasterExtent,
      cb,
      lines))

{% endhighlight %}

#### Finishing the Transit Times Layer

The last thing we need to do is to merge the walking layer
and the bus layer and run it through the server:

{% highlight scala %}
val rasterBothOp = AMin(rasterWalks, busTimeRaster)
val renderedRaster = server.run(rasterBothOp)
{% endhighlight %}


### Calculate the TransitShed

Calculating the actual transitshed just requires us to
run a cost distance operation. The operation will automatically
find the shortest path between a starting point and all cells
on the raster.

{% highlight scala %}
val costsOp = focal.CostDistance(Context.rasterWalks, Seq((x,y)))
{% endhighlight %}

### Example

For example:

![transitheapmap.com example](/images/transitheapmap.png "transitheapmap.com example")
