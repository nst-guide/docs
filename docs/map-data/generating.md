# Generate Map Data

A map of the globe contains so much information that it would be impossible to
transmit all that information at high scale to a user at once. That's why [map
tiling schemes][maptiler-map-tile-def] [exist][wiki-tiled_web_map]. With the
standard [pseudo-mercator tiling scheme][wiki-web_mercator_proj], map zoom level 0 covers the globe; zoom level 1 is split into 4 tiles; zoom level 2 is split
into 16 tiles, and so on. Zoom level $n$ has $4^n$ tiles that cover the globe.

[maptiler-map-tile-def]: https://www.maptiler.com/google-maps-coordinates-tile-bounds-projection/
[wiki-tiled_web_map]: https://en.wikipedia.org/wiki/Tiled_web_map
[wiki-web_mercator_proj]: https://en.wikipedia.org/wiki/Web_Mercator_projection

## Raster vs Vector tiles

Historically, map tiles were PNG images. This meant that for a web browser or
mobile phone to display the map, all they had to do was display a collection of
images. That approach is pretty simple for the client, which allows it to work
on low-powered devices like those of the late 2000s.

Modern map technology is going away from raster tiles and toward vector tiles.
Instead of encoding RGB values of pixels, it encodes the _actual geometries_ of
the roads, streams, and points of interest to be displayed. This requires more
effort on the client, but opens up a whole host of opportunities:

- **Smooth zooming**. Since the tiles are rendered on the client, you can have fractional zooms without any pixelation.
- **Dynamic styling**. Change how the map looks without needing to download new data.
- **Smaller file sizes**. In general, vector tiles have smaller file sizes than their raster equivalents.
- **Extract data from the tile**. Since the geometries are encoded in the data received by the client, the client could potentially do cool stuff like basic routing without needing any new data.

Note that this isn't to say raster data is going away entirely! Some data is
still well-suited to rasters, like aerial imagery or hillshading.

## OpenMapTiles

The OpenMapTiles project defines a _schema_ for how to encode data from
OpenStreetMap into files conforming to the [Mapbox Vector Tile
specification][mvt-spec], and offers tools for converting OpenStreetMap data
into tiles of that schema.

[mvt-spec]: https://github.com/mapbox/vector-tile-spec

The schema defines a collection of tags for the geometries encoded in the vector
tile, so that _styles_ can be written for tiles with consistent layers. Note
that in order to use a Mapbox GL style, it must be written specifically for the
OpenMapTiles schema. A style written for the OpenMapTiles schema won't work with
vector tile data coming from Mapbox, and a style written for Mapbox's vector
tile schema won't work for OpenMapTiles. That means you can't just take a
default Mapbox style, and use it for OpenMapTiles.

### Dynamic tile generation

If you're trying to serve map tiles for a huge area, such as the entire globe,
or with constant updates, the standard procedure would be to generate tiles
dynamically.

The process for doing dynamic tile generation is roughly:

- Construct an OpenMapTiles Postgres database with all the data of your desired region
- Create vector tiles on the fly, with a project such as [`martin`][urbica-martin].

[urbica-martin]: https://github.com/urbica/martin

This approach, however, has higher costs because you need Postgres running on
the backend. Since my project only serves the US and doesn't need constant
updates, I use static tile generation instead.

### Static tile generation

The general process for making tiles is:

- Install Docker
- Clone the OpenMapTiles repository
- Set the desired min/max zoom levels in `.env`
- Run for a region(s) of interest
- Combine the output `.mbtiles` files into a single `.mbtiles` file
- [Serve it](hosting.md)

I use a [modified fork](https://github.com/nst-guide/openmaptiles) of
OpenMapTiles, which

- Encodes waterways and trails into the vector tiles at a lower zoom than the original OpenMapTiles project, so that they can be displayed when more zoomed out.
- Adds `natural=spring` and `amenity=drinking_water` to the POI layer.

In code, the process is

```bash
git clone https://github.com/nst-guide/openmaptiles
cd openmaptiles
sudo docker-compose pull

# Make OpenMapTiles
regions="washington oregon"
mkdir -p mbtiles
# z expansion flag:
# https://stackoverflow.com/a/23164496
for region in ${(z)regions}; do
    sudo ./quickstart.sh $region && cp data/tiles.mbtiles mbtiles/$region.mbtiles;
done

# join tiles
tile-join joined.mbtiles mbtiles/*.mbtiles
```
The argument to `./quickstart.sh` is the region name. You can find valid region
names in the [OpenMapTiles documentation][openmaptiles-valid-region-names].

[openmaptiles-valid-region-names]: https://github.com/nst-guide/openmaptiles/blob/master/QUICKSTART.md#check-other-extracts

The regions are downloaded from [Geofabrik][geofabrik]. So if you wanted you
could run it on `north-america`, but I find it preferable to run smaller regions
at a time. By making a separate mbtiles for each state, you can update each
state individually. For example, if I generate individual `.mbtiles` files for
every state, and then want to regenerate the map tiles for the Pacific Crest
Trail, I only need to generate new data for California, Oregon, and Washington,
and then combine the existing data from all other states with
[`tile-join`][tile-join]. So I'm never forced to regenerate data for more than
one state at a time.

[geofabrik]: http://download.geofabrik.de/
[tile-join]: https://github.com/mapbox/tippecanoe#tile-join

Now that you've created a single `.mbtiles` file, you're [to host
it](hosting.md).

## Contours

I originally generated contours vector tiles straight from USGS vector data
([https://github.com/nst-guide/contours](https://github.com/nst-guide/contours)).
These downloads have contour lines pregenerated in vector format, and so are quite
easy to work with; just download, run `ogr2ogr` to convert to GeoJSON, and then
run [`tippecanoe`][tippecanoe] to convert to vector tiles.

[tippecanoe]: https://github.com/mapbox/tippecanoe

There are a couple drawbacks of using the USGS vector contour data:

1. Inconsistent vertical spacing. In some areas, I found that the data included
    10' contours, while others had a precision of only _80'_. Since I desired a
    40' contour, this meant that there were occasionally visual discontinuities
    in the map between tiles with source data of at least 40' precision and
    source data of less than 40' precision.
2. Metric contours. Although I'm currently making maps of the United States,
    where imperial measurements are the standard, it's desirable to provide
    metric measurements as well. With Mapbox GL, it's quite easy to write a
    style expression that converts feet into meters, but the _spacing_ of the
    contour lines will still be in feet. If neighboring imperial contour lines
    are 2000 feet and 2040 feet, displaying those same lines on a metric map
    would represent unintuitive values of 609.6 meters and 621.8 meters.

    In order to display metric contour lines, it is necessary to generate a
    separate set of contours with lines that represent 100 meters, 110 meters,
    etc. This must be generated from the original DEMs.

Because of these issues, I try to generate contour data from the original DEMs,
using code in [`nst-guide/terrain`][nst-guide/terrain].

[nst-guide/terrain]: https://github.com/nst-guide/terrain

## Hillshading

Historically, the way to make a hillshade is to generate raster images where
each cell stores the level of grayscale to display.

With Mapbox GL, you have a new option: [Terrain RGB
tiles](https://docs.mapbox.com/help/troubleshooting/access-elevation-data/#mapbox-terrain-rgb).
Instead of encoding the grayscale in the raster, it encodes the _raw elevation
value_ in 0.1m increments. This enables a whole host of cool things to do
client-side, like retrieving elevation for a point, or generating the viewshed
from a point.

I currently exclusively use Terrain RGB tiles in my projects because it allows for
the possibility of doing cool things client-side with elevation data.

I generate Terrain tiles in [`nst-guide/terrain`][nst-guide/terrain].
