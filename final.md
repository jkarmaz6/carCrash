### Spatial Database Design Final Report
by Jeff Karmazin

## Analysis of Car Accidents in Pennsylvania
### Pennsylvanie Department of Transportation Car Accident Data
The main dataset used in this analysis has been provided through the Pennsylvania Department of Transportation, representing all car accidents in PA for the year of 2019. The purpose of this dataset it to provide information on car accident locations and more data regarding these accidents. PennDOT's goal is to provide this maintain and provide this information in order to utlize law enforcement, engineering, and education to main safe highway systems.

This dataset is quite extensive, breaking accident data into many different categories and providing much specific information. For further analysis, the tiger line roads data for PA has also been utilized provided by the US Census Bureau. The car accident dataset will be used to determine and locate the most dangerous roads in Pennsylvania. Some questions to be answered include 
* What roads have the most car accident fatalities?
* Where are the highest amount of vehicles involved in car accidents located?
* How many car accidents have occured on some of the most used roadways in Pennsylvania?


### Data Structure and Normalization
The data provided has been restructured for use in this analysis. It has been formatted to 3NF and reduced to information of use regarding the relevant spatial analysis questions. Original datasets broke accident information into weather realted, specific flags (such as drunk driving or speeding), commercial vehicles, motorcycles, person, roadway, and trailervehicles. This is way more than need in this anaysis, so it will be reduced to 3 tables. These tables include a main crash table, injury look up table, and roadway table.

The main crash table will include 7 columns: 
    *gid, fatal_count, injury_count, person_count, max_injury_id, vehicle_count, and geom*

*gid* is the primary key for the crash table. It is sourced from the original dataset's crn(crash record number). It is of integer data type, and is uniquely specific to each car accident. The original dataset kept the crn in each table.

*fatal_count, injury_count, person_count, vehicle_count* are all carried over from the original dataset in their original form. These data types are integers.

*max_injury_id* is also of integer data type, but unique codes to different injury levels. The injury id serves as the primary key to the injury lookup table, which contains the injury id and description of each unique id. The injury look up table is as follows:
    0 - 'no injury'
    1 - 'killed'
    2 - 'suspected serious injury'
    3 - 'suspected minor injury'
    4 - 'possible injury'
    8 - 'injury severity unknown'
    9 - 'unknown if injured
    
The *geom* column utilizes longitude and latitude coordinates provided with the original dataset. This data coordinate system needed to first be set as point data with the original SRID (4269). It is then transformed into the desired SRID code (2272) of Pennsylvanie State Plane South. 

The *crash_road* data table contains 6 columns: *id, gid, lane_count, speed_limit, street_name, county*. *id* is the primary key in this table, with a foreign key *gid* to the main crash table. *lane_count, speed_limit, county* all are of integer data types and are selected exactly from their original form in the original dataset. *street_name* is of variable character form for reference to real world street names. *county* is a lookup table to specific county codes for a second location look up in the case of duplicate street names across PA regions.

[ERD](ERD - ERD.pdf)

TigerLine Road data is used in its original form provided from the US Census Bureau and not further normalized. For the purpose of this analysis, it is used as a second source of geometry to cross reference the car accident data. Columns utilized here are the *geom* and *full_name* to determine road locations.

## Optimization
The car accident data is a very extensive dataset, with over 125,000 recorded car accident for the year of 2019. To further optimize SQL DB queries, a few indexes were created. These include indexes on both the geomtries for car accidents and the geometry from the TigerLine Roads data, as well as on the *full_name* column from the Roads data. See Appendix C for inde scripts.


### Analysis










#### Appendix A - Data Source

All datasets can be found through PennDOT's Car Crash Portal:  https://crashinfo.penndot.gov/PCIT/welcome.html

Here, you can access reports and summaries, utilize interactive maps to find data yourself, or download original datasets for years 2000 - 2019.

Data was loaded in a computer shell with a ogr2ogr code. TigerLine shapefile data was loaded through QGIS data management.

#### Appendix B - Normalization 
```
---- CREATE look up tables  ----
--- Injury Table ---
drop table if exists injury cascade;
create table injury (
	injury_id int primary key,
	description varchar
);

insert into injury(injury_id, description)
 values 
 (0, 'no injury'),
 (1, 'killed'),
 (2, 'suspected serious injury'),
 (3, 'suspected minor injury'),
 (4, 'possible injury'),
 (8, 'injury severity unknown'),
 (9, 'unknown if injured');
 
---- create road table ----
drop table if exists crash_road cascade;
create table crash_road(
	id int primary key,
	gid int references crash(gid),
	lane_count int,
	speed_limit int,
	street_name varchar,
	county int
	);

insert into crash_road(id, gid, lane_count, speed_limit, street_name, county)
select ogc_fid ::int,
		crn::int,
		lane_count::int,
		speed_limit::int,
		street_name::varchar,
		county::int
from crash_road_data ;

--- create main Crash table
drop table if exists crash cascade;
create table crash (
	gid int primary key,
	fatal_count int,
	injury_count int,
	person_count int,
	max_injury_id int references injury(injury_id),
	vehicle_count int,
	geom geometry(point, 2272)
);

	

---- INSERT info into tables  ----

insert into crash(gid, fatal_count, injury_count, person_count, max_injury_id, vehicle_count,  geom)
select 
		crn::int,
		fatal_count::int,
		injury_count::int,
		person_count::int,
		max_severity_level::int,
		vehicle_count::int,
		st_transform(st_setSRID(st_point(dec_long::float, dec_lat::float), 4269), 2272)::geometry
		from import_crash_data icd; 
```
### Appendixe C - Index Scripts

```
create index road_transform on tl_2020_42_prisecroads
using gist (st_transform(geom, 2272));

create index geom_ind on crash
using gist(geom);

create index name_ind on tl_2020_42_prisecroads(fullname);

```
