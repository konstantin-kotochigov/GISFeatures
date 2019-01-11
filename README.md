# GISFeatures

Quite often in Machine Learning tasks and competitions GEO features. i.e. number of restaurants

A good data source is OpenStreetMap, which is also free.

There are several ways to work with OSM data
- Get map dumps and parse them, since OSM files are just a plain XMLs
- Load dumps into Postgres database
- Use REST API which provides its own query language (OverpassQL)

If data is needed for just a limited area, it can be exported data using manually set bounding box.

Next we will load it into PostGIS, which an extension to Postgres DBMS, that allows to work with spatial data.

A tool for loading GEO data into PostGIS is called osm2pgsql. It works with .osm, .pbf, .osm.gz file formats.

Before dat acan be loaded, we need to create a database. Install Postgres, create new DB, enable GIS extension.

It creates 4 tables:
- planet_osm_point
- planet_osm_line
- planet_osm_plygon
- planet_osm_collection
for corresponding data types. The 5th table spatial_ref is a reference of available coordinate systems.

Challenge is that tags overlay sometimes. For example, (leisure ='fitness-center', sport = 'yoga'). So there must be set an order.
