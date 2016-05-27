# Points of Interest Near Bus Stops

Objective: Let's find bus stops that have a large amount of
amenities within a half-kilometer radius.

We'll be making a map similar to this: https://td-data-workshop.cartodb.com/viz/67413342-2174-11e6-9958-0ecfd53eb7d3/public_map

## Sourcing Data

- `data.seattle.gov` actually doesn't contain datasets for either places
  of interest or bus stops/bus routes.
- Point of Interest and Bus Stop data available from [Mapzen
  Metro Extracts](https://mapzen.com/data/metro-extracts/)
- `OSM2PGSQL SHP` format contains more easy to access points
  of interest data (businesses, schools, places of worship, etc).
- `IMPOSM SHP` has easier to access bus stop information.

- The bounding box (extent) for the Seattle area is really large, so
  I've subset the data here: (https://github.com/mattmakesmaps/tech-diversified-workshop)


- Import `imposm_transportation_stuff.zip`, `osm_points.zip`, `osm_lines.zip`, and `osm_polys.zip` by copying and pasting the following links into cartodb.
- https://github.com/mattmakesmaps/tech-diversified-workshop/raw/master/imposm_transportation_stuff.zip
- https://github.com/mattmakesmaps/tech-diversified-workshop/raw/master/osm_points.zip

## Explore the data

The OSM Points dataset contains many different features.

To get a list of all values for a particular column, say "leisure",
use the following sql query:

```sql
SELECT DISTINCT leisure
FROM osm_points
```

To get COUNTS of features with that value:

```sql
SELECT leisure, COUNT(*) AS cnt
FROM osm_points
GROUP BY leisure
ORDER BY COUNT(*) DESC
```

The same queries can be written for the `amenity` and `shop` columns.

For the `seattle_imposm_transport_points` dataset, the main attribute
of interest is the `type` attribute. It's distinct values can be found
using a query similar to above, or by clicking on the `type` column, and 
by clicking the `filter by this column` button.

## Filtering the data

For the `seattle_imposm_transport_points` dataset, lets filter it to only
include `bus_stop` features, since we're only interested in bus stops. Save this dataset as `bus_stops`

Going back to the `osm_points` dataset, let's filter it to include only
the points of interest we are really interested in.

```sql
SELECT *
FROM osm_points
WHERE leisure IS NOT NULL
OR shop IS NOT NULL
OR (
  amenity IS NOT NULL
  AND amenity <> 'bicycle_parking'
)
```

NOTE in the query above we're excluding bicycle parking, which makes up over 1,200 records in the `amenity` column.

Create a dataset from the above query and name it `pois`

## Perform the Spatial Analysis

Create a new map in cartodb using the `bus_stops` and `pois` layers we've just created.

Our analysis will revolve around getting a count of ammentiy/shop/lesiure points
in the `pois` dataset that are within a half-kilometer radius of a bus stop.
In order to store the results of our analysis we'll need to add a new
column to the `bus_stops` table called `poi_count`.

- Select the `bus_stops` dataset and click the
`add column` button. Name the column `poi_count` and give it a data type
of number.

Now we can begin actually asking the question "how many points of interest are within 500m of a bus stop?"

To do so, we'll execute the following SQL statement on the `bus_stop` table:

```sql
  SELECT bus_stops.id, COUNT(*)
  FROM pois
  INNER JOIN bus_stops
  ON ST_DWITHIN(
    pois.the_geom::geography,
    bus_stops.the_geom::geography,
    500)
  GROUP BY bus_stops.id
```

These counts are interesting, but not meaningful unless they are
attached to actual geometries we can see on a map. To do this, we'll
update the `poi_count` attribute with these counts.

-- UPDATE
UPDATE bus_stops
SET poi_count = poi_count.count
FROM (
  SELECT bus_stops.id, COUNT(*)
  FROM pois
  INNER JOIN bus_stops
  ON ST_DWITHIN(
    pois.the_geom::geography,
    bus_stops.the_geom::geography,
    500)
  GROUP BY bus_stops.id
) AS poi_count
WHERE bus_stops.id = poi_count.id

## Making the Map

Now that we have the data prepared, we can focus on styling the map
to glean information from the data.

- Using the `map layer wizard` explore symbolizing the `poi_count`
  attribute using different styles. A choropleth style works well
  for this attribute. Different styles of classification can also
  be used.

- The `pois` table can be symbolized in many different ways as well.
  Just displaying the points is overwhelming. Try a cluster, heatmap,
  or density map to view the data.

- To add a popup use the `infowindow` wizard for both layers. A useful
  bit of information might be to display the `id`, `name`, and `poi_count`
  value for a bus stop of interest.

- Data can be filtered on the fly using the `filter` widget. An
  example of useful filtering would be to remove bus stops that
  don't contain any POIs within a 500m radius.

## Additional Questions

Now that we know the counts of POIs around all our bus stops, let's
ask the question, "What specific POIs are around a particular bus stop?"

```sql
  SELECT pois.amenity, pois.leisure, pois.shop, pois.name
  FROM pois
  INNER JOIN bus_stops
  ON ST_DWITHIN(
    pois.the_geom::geography,
    bus_stops.the_geom::geography,
    500)
  WHERE bus_stops.id = <INSERT ID>
  AND amenity <> 'bicycle_parking'
```
NOTE: To preserve your map styling, it's easiest to just open a new
browser tab and navigate to the `bus_stops` dataset directly to
execute the above query.