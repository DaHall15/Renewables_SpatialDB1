> # Renewable Energy Group Project

**Analyzing siting options for renewables in the Massachusetts towns of Bernardston, Northfield, and Gill**

_Daniel Hall, Binghui Li, Fletcher Harrington_  
_4/3/2024_

## Objectives

The primary objectives for this project:

- Acquire spatial data for focus area describing land cover, wind speed, irradiance, and elevation
- Create database and table schema for data sources and normalize as necessary
- Analayze suitability of lands in focus area for renewable energy siting

## Focus Area

<img width="600px" src="DataScreenShots/locus.png" alt="locus"></img>\

## Data

Data sources so far are as follows:

| Name             | Source  | Data type | format      | Resolution     | link (if applicable)                                        |
| ---------------- | ------- | --------- | ----------- | -------------- | ----------------------------------------------------------- |
| Solar Irradiance | NREL    | raster    | jpg, 300dpi | 30m            | https://globalsolaratlas.info/download/usa                  |
| NLCD 2021        | USGS    | raster    | tif         | 30m            | mrlc.gov/                                                   |
| Wind Speed       | MassGIS | vector    | shp         | 250m x 250m    | mass.gov/info-details/massgis-data-modeled-wind-speed-grids |
| Land Elevation   | USGS    | raster    | tif         | 1/3 arc-second | https://apps.nationalmap.gov/downloader/                    |
| Town Boundaries  | MassGIS | vector    | shp         | n/a            | maps.massgis.digital.mass.gov/MassMapper/MassMapper.html    |

## General Data Processing

- All spatial data was reprojected to EPS:26986
- All spatial data was clipped to the merged town boundaries or a similar extent polygon
- Aspect generated from elevation data with QGIS Aspect tool
- Land cover data classes simplified with QGIS Raster Calculator
- Data normalized as necessary (no dependencies identified, unnecessary columns dropped)

## Data Normalization

### Wind Speed

- Once imported into database, non-essential fields were dropped using the three following queries in pgAdmin :

```
ALTER TABLE windspeed
DROP COLUMN x_coords;

ALTER TABLE windspeed
DROP COLUMN y_coords;

ALTER TABLE windspeed
DROP COLUMN objectid;
```

#### The Initial Windspeed table:

| GID (integer) | objectid | spd30 (numeric) | x_coords     | y_coords     | geom (geometry)                                |
| ------------- | -------- | --------------- | ------------ | ------------ | ---------------------------------------------- |
| 1             | 10608    | 4.4883          | x-coordinate | y-coordinate | "01060000206A69000001000000010300000001000..." |
| 2             | 11140    | 3.4000          | x-coordinate | y-coordinate | "01060000206A69000001000000010300000001000..." |
| 3             | 11670    | 5.0676          | x-coordinate | y-coordinate | "01060000206A69000001000000010300000001000..." |

#### Final Windspeed table:

| GID (integer) | spd30 (numeric) | geom (geometry)                                |
| ------------- | --------------- | ---------------------------------------------- |
| 1             | 4.4883          | "01060000206A69000001000000010300000001000..." |
| 2             | 3.4000          | "01060000206A69000001000000010300000001000..." |
| 3             | 5.0676          | "01060000206A69000001000000010300000001000..." |

---

Convert raster files into SQL format:

```
raster2pgsql -s 4324 -I -C -M DNI.tif public.DNI_focus_area.tif > DNI.sql
raster2pgsql -s 4326 -I -C -M elevation.tif public.elevation_focus_area.tif > elevation.sql
```

Import the raster data into databas:

```
psql -d groupProj -U postgres -h localhost -p 5433 -f DNI.sql
psql -d groupProj -U postgres -h localhost -p 5433 -f elevation.sql
```

#### DNI table:

| rid | rast                                                 |
| --- | ---------------------------------------------------- |
| 1   | 01000001000014AE47E17AA43F0014AE47E17AA4BFA4703D0... |

#### Elevation table:

| rid | rast                                                  |
| --- | ----------------------------------------------------- |
| 1   | 010000010048A1FB111111813F11111111111181BF1C276920... |

#### Aspect table (derived from elevation table):

| rid | rast                                                   |
| --- | ------------------------------------------------------ |
| 1   | 01000001009BA5516E884B874066F3716D884B87C02A3A92CB2... |

### Landcover Classification

```
raster2pgsql -s 26986 -I -C -M rcalc3.tif public.lc > lc.sql
```

```
psql -U postgres -d spatial_one -f lc.sql
```

```
SELECT rid, rast::varchar(64) from lc;
```

#### Land Cover table:

| rid | rast                                                             |
| --- | ---------------------------------------------------------------- |
| 1   | 0100000100BADBA08206F23D402F68308406F23DC02FDD24061F03FA40AF9465 |

#### Values Before Reclassification:

Total pixel count: 558976

NODATA pixel count: 0

| Value | Pixel count | Area (m²)         |
| ----- | ----------- | ----------------- |
| 0     | 345257      | 309601513.0953814 |
| 11    | 6370        | 5712155.404286024 |
| 21    | 8479        | 7603354.108781978 |
| 22    | 7167        | 6426847.375591513 |
| 23    | 4236        | 3798538.507465557 |
| 24    | 646         | 579286.0896654272 |
| 31    | 1076        | 964878.9976470585 |
| 41    | 41654       | 37352295.32341131 |
| 42    | 56779       | 50915301.67974194 |
| 43    | 51206       | 45917838.2467614  |
| 52    | 1048        | 939770.6222436033 |
| 71    | 1456        | 1305635.520979663 |
| 81    | 16673       | 14951140.8250645  |
| 82    | 8243        | 7391726.373238571 |
| 90    | 7868        | 7055453.488370869 |
| 95    | 818         | 733523.2528580797 |

### Values After Reclassification:

Total pixel count: 558976

NODATA pixel count: 0

| Value | Pixel count | Area (m²)         |
| ----- | ----------- | ----------------- |
| 0     | 345257      | 309601513.0953814 |
| 11    | 15056       | 13501132.14551497 |
| 24    | 21604       | 19372905.07915153 |
| 43    | 150687      | 135125205.8721583 |
| 81    | 18129       | 16256776.34604416 |
| 82    | 8243        | 7391726.373238571 |

- 0 value represents null empty values surrounding image

## Post-processed data examples

#### Landcover Raster after reclassification

<img width="600px" src="DataScreenShots/lc1.png" alt="landcover1" style = "float: left, width: 600px, padding: 5px"></img>
<img src="DataScreenShots/lc1leg.png" alt="landcover1legend" style ="float: right, width: 60px, padding: 5px"></img>

<img width="600px" src="DataScreenShots/lc_areas.png" alt="lc_area" style ="width: 500px"></img>

#### Aspect derived from elevation

<img width="600px" src="DataScreenShots/aspect.png" alt="aspect" style ="float: left, width: 600px"></img>

#### Windspeed

<img width="600px" src="DataScreenShots/windspeed.png" alt="windspeed" style ="width: 600px"></img>

# Data Preparation and Creation

- import dem to QGIS, reproject, resample to larger cell size, and clip (used '100' cell size)
- create postgis script and import as raster table in db

```
raster2pgsql -s 26986 -I -C -M dem_resamp_26986.tif public.dem_rast > demr.sql
psql -U postgres -d SpatQuery -f demr.sql
```

<img width="600px" src="DataScreenShots/dem.png" alt="dem" style ="float: left, width: 300px"></img>\

- create aspect raster in QGIS using dem_rast table and GDAL Aspect tool
- load aspect raster as table

```
raster2pgsql -s 26986 -I -C -M aspect_hd_clip_26986.tif public.aspect_rast > aspr.sql
psql -U postgres -d SpatQuery -f aspr.sql
```

- create vector version of raster table in postgis using ST_PixelAsPolygons to create a polygon cell for each raster cell

```
CREATE TABLE aspect_vect AS SELECT val, geom As geom FROM (SELECT (ST_PixelAsPolygons(rast)).* FROM aspect_rast) As foo;
```

<img width="600px" src="DataScreenShots/aspect.png" alt="asp" style ="float: left, width: 300px"></img>\

- import dni file to QGIS, reproject and clip
- load dni raster into postgis raster table

```
raster2pgsql -s 26986 -I -C -M dni_hd_clip.tif public.dni_rast > dni.sql
psql -U postgres -d SpatQuery -f dni.sql
```

- create vector version of dni table

```
CREATE TABLE dni_vect AS SELECT val, geom As geom FROM (SELECT (ST_PixelAsPolygons(rast)).* FROM dni_rast) As foo ORDER BY val;
```

<img width="600px" src="DataScreenShots/dni.png" alt="dni"></img>\

- load landcover raster dataset into QGIS, remove extraneous bands using SAGA Rearrange Raster tool, clip to focus area
- load landcover raster table into postgis
- use raster calculator to join classes into more general categories (e.g. wetlands+water, all developed areas, pasture+ meadow)
- load resulting layer into database

```
raster2pgsql -s 26986 -I -C -M lc_1band_26986.tif public.lc_rast > lc.sql
psql -U postgres -d SpatQuery -f lc.sql
```

- create vector version of land cover table use ST_DumpAsPolygons to dissolve adjacent same value pixels (land cover types)

```
CREATE TABLE lc_vect AS SELECT val, geom As geom FROM (SELECT (ST_DumpAsPolygons(rast)).* FROM lc_rast) As foo WHERE val BETWEEN 11 and 82 ORDER BY val;
```

- open land cover vector table in QGIS and remove incompatible classes (crops, developed, water)

<img width="600px" src="DataScreenShots/lc_selected.png" alt="ssdni" style ="float: left, width: 300px"></img>\

- import windspeeds at 50m

<img width="600px" src="DataScreenShots/windspeed.png" alt="wind"></img>\

## Analyses

### Wind Analysis

- It is known that the prevailing winds in Massachusetts are westerly (originating from the west). Suitable wind farm areas would have slopes that face the west, with aspect values ranging from 202.5 to 337.5 (west, southwest and northwest). Therefore, the threshold value range for aspect is 202.5 to 337.5.

- The minimum average wind speed at 50 meters elevation is six meters per second. Therefore, the threshold value of 6 m/s was chosen for wind.

- The threshold value for elevation was chosen to be 100 meters given previous wind speed and suitability analyses for Massachusetts.

- This preliminary analysis identifies suitable areas for wind farm placement in the area of interest using wind speed, elevation, and aspect data.

#### Workflow

- Wind speed data (vector) including wind speed at 30-, 50-, 70- and 100-meters elevation was imported into QGIS. All extraneous fields were deleted (wind speed at 30, 70 and 100 meters) since we were only interested in wind speed at 50 meters elevation. Data was projected in EPSG 26986. Wind speed loaded into postgis database using vector2pgsql.

- Elevation data (originally raster) was converted into a vector using folloing SQL command after first loading it into postgis:

```
CREATE TABLE dem_vect AS SELECT val, geom As geom FROM (SELECT (ST_PixelAsPolygons(rast)).* FROM dem_rast) As foo;
```

- Aspect data (originally raster) was created and processed in QGIS (process is described below in the solar section). Using SQL, aspect was converted into a vector file and reprojected (also described in detail in the solar analysis section below).

Once all layers were vectorized – aspect, elevation and wind speed – and loaded into the postgis database, the SQL query was run to determine the interesection of areas with suitable wind speed, aspect, and elevation:

```
CREATE TABLE wind_intersect_1 AS
SELECT
  ST_Intersection(wind_intersect_1.geom, lc_selected.geom) AS geom,
  wind_intersect_1.aspect_val AS aspect_val,
  wind_intersect_1.dem_val AS dem_val,
  wind_intersect_1.windspeed_val AS windspeed_val
FROM
  wind_intersect_1
  JOIN dem_vect ON ST_Intersects(aspect_vect.geom, dem_vect.geom)
  JOIN windspeed_50 ON ST_Intersects(aspect_vect.geom, windspeed_50.geom)
WHERE
  aspect_vect.val BETWEEN 202.5 AND 337.5
  AND dem_vect.val >= 100
  AND windspeed_50.spd50 >= 6;

CREATE TABLE wind_intersect_2 AS
SELECT
  ST_Intersection(wind_intersect_1.geom, lc_selected.geom) AS geom,
  lc_selected.land_class AS land_class,
  wind_intersect_1.aspect_val AS aspect_val,
  wind_intersect_1.dem_val AS dem_val,
  wind_intersect_1.windspeed_val AS windspeed_val
FROM
  wind_intersect_1, lc_selected
```

#### PostGIS intersects and threshold values were used to create sites that met the above threshold values for elevation, apsect, windspeed, and land cover types

<img width="600px" src="DataScreenShots/wind_sites.png" alt="wsites"></img>\
_Wind Sites - polygon layer of potential wind power sites_

## Solar Analysis

- In the Northern Hemisphere, facing solar panels to the south (180°) is ideal because it ensures the panels get the most direct sunlight throughout the day
- The preliminary analysis used a slighltly westward biased angle, final analysis will use more southerly direction
- A minimum threshold of 6.0 kWh/m2/day in DNI is a good general guideline as suggested [here](https://www.esmap.org/sites/default/files/esmap-files/ESMAP_IFC_RE_CSP_Training_World_Bank_Romero.pdf)
  - Note: we discovered late in the process that our dni data units were out of range for unkown reasons. We used an upper quartile of existing values as a proxy for good solar values

### Workflow

- Use aspect and dni vector tables to create a vector table of their intersection at thresholds chosen for solar

```
CREATE TABLE ad_intersect
AS SELECT dni_vect.val as dni_val,
  aspect_vect.val as asp_val,
  st_intersection(aspect_vect.geom, dni_vect.geom) as geom
from aspect_vect, dni_vect
where (dni_vect.val > 1450) = true
  and (aspect_vect.val >= 130 AND aspect_vect.val <= 230)
  and st_intersects(aspect_vect.geom, dni_vect.geom)

```

<img width="600px" src="DataScreenShots/solar_sites_adintersect.png" alt="ssad" style ="float: left, width: 300px"></img>\

```
CREATE TABLE solar_sites
AS SELECT
  ad_intersect.dni_val as dni_val,
  ad_intersect.asp_val as asp_val,
  lc_selected.land_class as land_class,
  st_intersection(lc_selected.geom, ad_intersect.geom) as geom
from ad_intersect, lc_selected
where
  st_intersects(ad_intersect.geom, dni_vect.geom)
```

- intersect resulting table with selected land classes retaining aspect, dni, and land class values

#### Selected Solar Sites Shaded by DNI

<img width="600px" src="DataScreenShots/solar_sites_satellite.png" alt="ssdni" style ="float: left, width: 300px"></img>\
