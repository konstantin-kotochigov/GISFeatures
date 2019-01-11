# GISFeatures

Quite often in Machine Learning tasks and competitions GEO features. i.e. number of restaurants

A good data source is OpenStreetMap, which is also free.

There are several ways to work with OSM data
- Get map dumps and parse them, since OSM files are just a plain XMLs
- Load dumps into Postgres database
- Use REST API which provides its own query language (OverpassQL)

If data is needed for just a limited area, it can be exported data using manually set bounding box.

Next we will load it into PostGIS, which an extension to Postgres DBMS, that allows to work with spatial data.

A tool for loading GEO data into PostGIS is called **osm2pgsql**. It works with **.osm**, **.pbf**, **.osm.gz** file formats.

Before dat acan be loaded, we need to create a database. 

First, install Postgres

```sh
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt xenial-pgdg main" >> /etc/apt/sources.list'
wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -
sudo apt update

sudo apt install postgresql-10
sudo apt install postgresql-10-postgis-2.4 
sudo apt install postgresql-10-postgis-scripts
```

Configurate DB
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

Next we are ready to import geo data using osm2pgsql
```sh
sudo apt-get install osm2pgsql
osm2pgsql --username mysuperuser --database gisdb -l map_dump.osm
```

By default it creates 5 tables for corresponding data types:
- planet_osm_point
- planet_osm_line
- planet_osm_plygon
- planet_osm_collection 
- spatial_ref (is a reference of available coordinate systems)

Note the flag **"-l"**: it enables the "lonlat" format



Challenge is that tags overlay sometimes. For example, (leisure ='fitness-center', sport = 'yoga'). So there must be set an order.
