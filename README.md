# GISFeatures

Quite often in ML tasks, that involve geographical data, there is a need for geographical features, for instance, What is the number of restaurants in the 1km neighborshood? What is a distance to nearest bus stop? 

While there is a few good GIS systems, most of them cost money. A good alternative is OpenStreetMap - free GIS database.

*All the code below was run on Ubuntu 18.04.*

There are several ways to work with OSM data
- Get map dumps and parse them, since OSM files are just a plain XMLs
- Load dumps into Postgres database
- Use REST API which provides its own query language (OverpassQL)

If data is needed for just a limited area, it can be exported data using manually set bounding box.

Next we will load it into PostGIS, which an extension to Postgres DBMS, that allows to work with spatial data.

Before dat acan be loaded, we need to create a database. 

First, let's install an instance of Postgres DBMS

```sh
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt bionic-pgdg main" >> /etc/apt/sources.list'
wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -
sudo apt update

sudo apt install postgresql-10
sudo apt install postgresql-10-postgis-2.4 
sudo apt install postgresql-10-postgis-scripts
```

Create and configure a new database
```sql
sudo -u postgres psql
> CREATE EXTENSION adminpack;
> CREATE DATABASE gisdb;
> \connect gisdb;
> CREATE SCHEMA postgis;
> ALTER DATABASE gisdb SET search_path=public, postgis, contrib;
> \connect gisdb;  -- this is to force new search path to take effect
> CREATE EXTENSION postgis SCHEMA postgis;
> ALTER SYSTEM SET listen_addresses='*'; 
```

Enable remote access
```sh
sudo nano /etc/postgresql/10/main/pg_hba.conf
hostssl    all             all             0.0.0.0/0               md5
sudo service postgresql restart
sudo -u postgres psql
CREATE ROLE mysuperuser LOGIN PASSWORD 'password' SUPERUSER;
```

Finally we are ready to import geo data. A tool for loading GEO data into PostGIS is called **osm2pgsql**. It works with **.osm**, **.pbf**, **.osm.gz** file formats.

```sh
sudo apt-get install osm2pgsql
osm2pgsql --username mysuperuser --database gisdb -l map_dump.osm
```

Note the flag **"-l"**: it enables the "lonlat" format

By default it creates 5 tables for corresponding data types:
- planet_osm_point
- planet_osm_line
- planet_osm_polygon
- planet_osm_collection 
- spatial_ref (is a reference of available coordinate systems)

We will primarily focus on point objects. Each point tag is a separate column. 

Note that there can be multiple tags per one point (though its not very often). For example, it could be (leisure ='fitness-center', sport = 'yoga'). We could have flatten all tag columns into one column "point_type" if we define some tag priority, but instead we'll stick with a separate feature for each tag, wchich is more clear and straightfprward.


## Load training data

```python
# Import Points into Postgres DB
from sqlalchemy import create_engine
from tqdm import tqdm
engine = create_engine('postgresql+psycopg2://pguser:password@localhost/gisdb')
engine.execute("DROP TABLE IF EXISTS CIAN")
engine.execute("CREATE TABLE IF NOT EXISTS CIAN (LAT FLOAT, LON FLOAT, POINT Geography)")
for line in tqdm(df.iterrows(), total=df.shape[0], unit="rows"):
  lat = str(line[1].geo_lat)
  lon = str(line[1].geo_lon)
  result = engine.execute("INSERT INTO CIAN VALUES (" + lat + "," + lon + ", ST_POINT("+lon+","+lat+")::geography); COMMIT;")
```




## Generate distance features
Let's make some features. First, We want to calculate the number of objects of each type in training point neighborhood. Here comes a spatial join.

For a moment, let's use a tag called "amenity". It's value describes the type of amenity: cafe, shop and so on.

Spatial function **ST_DWithin(X,Y,R)** is a predicate that checks that the distance between X and Y is less then a radius R. We will use it as our join condition. Make sure X and Y are of type Geography not Geometry (or it will compute distance in coordinates instead of meters).

We join our training cases table on OSM table

Note that we want to compute counts for each amenity separately (number of cafes, number of restaurants and number of bus stops), so we add a **GROUP BY tag** statement.

Also for each training case we will compute several features: Number of objects of each type in radius 100, 500, 1000 meters.

To make code more consise, we parametrize an SQL query as a function:
```python
def create_sql(colname):
    return ('CREATE TABLE DF_FEATURES AS select c.POINT, p."'+colname+'", \
        SUM(CASE WHEN ST_DISTANCE(c.POINT, p.way::geography) <= 100 THEN 1 ELSE 0 END) as "'+colname+'_cnt_100", \
        SUM(CASE WHEN ST_DISTANCE(c.POINT, p.way::geography) <= 500 THEN 1 ELSE 0 END) as "'+colname+'_cnt_500", \
        SUM(CASE WHEN ST_DISTANCE(c.POINT, p.way::geography) <= 1000 THEN 1 ELSE 0 END) as "'+colname+'_cnt_1000" \
            from DF c join planet_osm_point p on ST_DWithin(c.POINT,p.way::geography,1000) where p."'+colname+'" is not null group by c.POINT, p."'+colname+'";')
```

If spatial index was created, the query can be quite fast. On my laptop it made a join of 20K training base on 140K data points in about 1 min.

We apply the same logic to other tags of interest and join the results.

```python
columns_to_process = ['amenity','leisure','']
            
```

