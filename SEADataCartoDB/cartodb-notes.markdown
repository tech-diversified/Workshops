# Carto DB Notes

## Adding Datasets

**Dataset with Separate Lat/Lon Columns**

CartoDB can consume links to `data.seattle.gov` **CSV** exports and **ZIP'd SHPs** directly.
For example, for the [My Neigbhorhood Map](https://data.seattle.gov/Community/My-Neighborhood-Map/82su-5fxf)
dataset, clicking on the `Export` button will open up a drop-down of available formats. The `CSV` link:
 (https://data.seattle.gov/api/views/82su-5fxf/rows.csv?accessType=DOWNLOAD) can be directly pasted into
CartoDB. CartoDB will pickup the separate Lat/Lon columns and automatically georeference.

**Dataset with Only a Shape or Location Column**

For datasets stored with a "location" column on `data.seattle.gov` but no separate `lat` `lon` columns,
You can use this workflow to import the dataset into CartoDB, but will need to manually create split `latitude` and `longitude`
columns. An example of this type of dataset is the [Landmarks](https://data.seattle.gov/Community/Landmarks/7nqc-eijt)
dataset. See the [CartoDB FAQ](https://docs.carto.pageospatial.com/faqs.html) For SQL snippets on creating split columns.

1. Add two new string type columns, `latitude` and `longitude`
2. UPDATE these columns with values from the `data.seattle.gov` `shape` or `location` column.

```
-- NOTE: `-?\d+[\.]?\d+` == "An optional '-' symbol, followed by 
-- one or more digits, followed by an optional '.' symbol
-- followed by one more more digits.
UPDATE landmarks
SET latitude = (
    regexp_matches(shape, '(-?\d+[\.]?\d+)?,\s(-?\d+[\.]?\d+)')
    )[1],
  longitude = (
    regexp_matches(shape, '(-?\d+[\.]?\d+)?,\s(-?\d+[\.]?\d+)')
    )[2]
```

3. Clicking on the `GEO` button next to the `the_geom` column allows
you to assign lat/lon columns to be used to construct the geometry. Select
out the columns you just populated.

## Symbolizing Data

- Use the wizard to create a categorical style map.
- One example of this is to symbolize the `seattle_cultural_space_inventory` against the `dominant_discipline` field.

## Spatial Analysis Example

An example creating a map using the `neighborhoods` and `seattle_cultural_inventory_dataset`.

Create two fields: A raw count of feature intersecting a neighborhood, and a density calculation.

Flipping between the `raw count` map and the `density` depicts the data in different ways. Compare small neighborhoods to large neighborhoods for example.

```
-- Update field `cultural_inventory_cnt` with
-- number of `seattle_cultural_space_inventory` features
-- Insersecting It.
UPDATE neighborhoods
SET cultural_inventory_cnt = (
  SELECT COUNT(seattle_cultural_space_inventory.cartodb_id)
  FROM seattle_cultural_space_inventory
  WHERE ST_Intersects(
    neighborhoods.the_geom,
    seattle_cultural_space_inventory.the_geom
  )
)

-- Landmark Density = Count of Landmarks / Area
UPDATE neighborhoods
SET cultural_inventory_density = (cultural_inventory_cnt/area)*1000
```

Another example showing new construction permits by neighborhood.

- Create a new map selecting the `neighborhoods` and `building_permits_current` layers.
- From the data view filter the `building_permits_current` layer to constrain:
  - `action_type` == `new`
  - `category` attribute to include only `SINGLE FAMILY/DUPLEX` and `MULTIFAMILY`
- Note how the `sql` pane of the `building_permit_current` table now has custom sql matching our filters

```sql
SELECT * FROM building_permits_current
WHERE action_type IN ('NEW') 
  AND action_type IS NOT NULL
AND category IN ('SINGLE FAMILY / DUPLEX','MULTIFAMILY') 
  AND category IS NOT NULL
```

- Add two new columns to `neighborhoods` dataset `bld_permit_cnt` and `bld_permit_density`.

Calculate
```sql
UPDATE neighborhoods
SET bld_permit_cnt = (
  SELECT COUNT(building_permits_current.cartodb_id)
  FROM building_permits_current
  WHERE ST_Intersects(
    neighborhoods.the_geom,
    building_permits_current.the_geom
  ) 
  -- APPLY Building Permit Filters
  AND action_type IN ('NEW') AND action_type IS NOT NULL
  AND category IN ('SINGLE FAMILY / DUPLEX','MULTIFAMILY') AND category IS NOT NULL
)

-- Density = Count / Area
UPDATE neighborhoods
SET bld_permit_density = (bld_permit_cnt/area)*1000
```