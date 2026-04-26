# 🗺️ PostGIS in Arabic — Practical Training Manual

> **By Ahmad Saleh — April 2026**

This manual is designed to fill a major gap in Arabic technical content in the field of spatial databases.  
My goal is to empower the Arab community to confidently use GIS and spatial database technologies.

> 📌 **This manual works alongside the video lessons — it is not a textbook, but practical notes to follow while watching and learning.**

---

## 🔗 Important Links

| Resource | Link |
|---|---|
| 📁 GitHub — Data & Documents | [postgis-spatial-project-arabic](https://github.com/ahmadsaleh009/postgis-spatial-project-arabic) |
| 📖 PostGIS Workshop | [postgis.net/workshops/postgis-intro](https://postgis.net/workshops/postgis-intro/index.html) |
| 🐘 Download PostgreSQL | [postgresql.org/download](https://www.postgresql.org/download/) |

---

## 📚 Table of Contents

1. [PostgreSQL Quick Start](#1-postgresql-quick-start)
2. [PostGIS Introduction](#2-postgis-introduction)
3. [Real World Data — NYC Dataset](#3-real-world-data--nyc-dataset)
4. [Spatial Joins](#4-spatial-joins)
5. [Spatial Relationships](#5-spatial-relationships)
6. [Spatial Indexes](#6-spatial-indexes)
7. [KNN Search](#7-knn-search)
8. [ANALYZE and VACUUM](#8-analyze-and-vacuum)
9. [Clustering on Indices](#9-clustering-on-indices)
10. [PostGIS Validity](#10-postgis-validity)
11. [More Spatial Functions](#11-more-spatial-functions)
12. [DE-9IM — Spatial Relationship Model](#12-de-9im--spatial-relationship-model)
13. [Linear Referencing](#13-linear-referencing)
14. [PostgreSQL Functions & Procedures](#14-postgresql-functions--procedures)
15. [Triggers & Rules](#15-triggers--rules)
16. [Tracking Edit History](#16-tracking-edit-history)
17. [PostGIS Topology](#17-postgis-topology)

---

## 1. PostgreSQL Quick Start

### Basic Database & Table Operations

```sql
-- Create a database
CREATE DATABASE sales;

-- Create a table
CREATE TABLE customers (name TEXT, phonenumber TEXT);

-- Insert a row
INSERT INTO customers (name, phonenumber) VALUES ('ahmad', '123345678');

-- Query
SELECT * FROM customers;
```

### Schemas — Think of Them as Folders

Schemas help you organize tables, separate projects, control access, and backup/restore groups of tables.

```sql
CREATE SCHEMA myfirstschema;
CREATE SCHEMA app1;
```

### Users, Roles & Permissions

```sql
-- Create a user
CREATE USER adam WITH LOGIN PASSWORD '123';
GRANT CONNECT ON DATABASE mynewdb TO adam;
GRANT USAGE ON SCHEMA myfirstschema TO adam;

-- Read only
GRANT SELECT ON TABLE myfirstschema.customers TO adam;

-- Read and write
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE myfirstschema.customers TO adam;
```

**Better practice — use roles:**

```sql
-- Create roles
CREATE ROLE sales_readonly NOLOGIN;
CREATE ROLE sales_rw NOLOGIN;

-- Create users
CREATE USER readonlyuser WITH LOGIN PASSWORD '123';
CREATE USER readwriteuser WITH LOGIN PASSWORD '123';

-- Assign users to roles
GRANT sales_readonly TO readonlyuser;
GRANT sales_rw TO readwriteuser;

-- Grant permissions to roles
GRANT CONNECT ON DATABASE mynewdb TO sales_readonly, sales_rw;
GRANT USAGE ON SCHEMA public TO sales_readonly, sales_rw;
GRANT SELECT ON TABLE myfirstschema.customers TO sales_readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE myfirstschema.customers TO sales_rw;
```

### Common Column Types

| Category | Type | Example | Use Case |
|---|---|---|---|
| Primary Key (numeric) | `BIGINT GENERATED ALWAYS AS IDENTITY` | `id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY` | Internal IDs |
| Primary Key (public) | `UUID` | `id UUID DEFAULT gen_random_uuid()` | Public/shared IDs |
| Text | `TEXT` | `name TEXT` | Names, descriptions |
| Money/Decimal | `NUMERIC(12,2)` | `price NUMERIC(12,2)` | Currency |
| Integer | `INTEGER` | `qty INTEGER` | Counts |
| Boolean | `BOOLEAN` | `is_active BOOLEAN DEFAULT true` | Flags |
| Timestamp | `TIMESTAMPTZ` | `created_at TIMESTAMPTZ DEFAULT now()` | Time tracking |
| Date | `DATE` | `birth_date DATE` | Calendar date |
| Status | `TEXT + CHECK` | `status TEXT CHECK (status IN (...))` | Controlled values |
| Flexible Data | `JSONB` | `meta JSONB DEFAULT '{}'` | Extra attributes |
| Arrays | `TEXT[]` | `tags TEXT[]` | Small lists |
| Range Types | `TSTZRANGE` | `valid_range TSTZRANGE` | History tracking |

### SQL JOIN Types

| Join Type | Keeps |
|---|---|
| `INNER` | Only matching rows |
| `LEFT` | Everything from the left table |
| `RIGHT` | Everything from the right table |
| `FULL` | Everything from both tables |

---

## 2. PostGIS Introduction

### Enable PostGIS

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
SELECT postgis_full_version();
```

### Creating Geometries with WKT

WKT (Well-Known Text) is a standard way to describe spatial shapes using coordinates.

```sql
-- Point
SELECT ST_GeomFromText('POINT(-78 40)');

-- Line
SELECT ST_GeomFromText('LINESTRING(-78 40, -80 45)');

-- Polygon
SELECT ST_GeomFromText('POLYGON((-78 40, -80 45, -81 50, -78 40))');
```

### Create a Spatial Table

```sql
CREATE TABLE testgeo (name TEXT, geo GEOMETRY);

-- Insert a point using WKT
INSERT INTO testgeo (name, geo) VALUES ('point1', 'POINT(-78 40)');

-- Insert using ST_MakePoint + SRID
INSERT INTO testgeo (name, geo) VALUES ('point2', ST_SetSRID(ST_MakePoint(-77, 39), 4326));

-- Insert a polygon
INSERT INTO testgeo (name, geo) VALUES (
  'poly',
  ST_SetSRID(ST_MakePolygon(ST_GeomFromText('LINESTRING(-78 40,-79 39,-79 42,-78 40)')), 4326)
);

-- Insert a line
INSERT INTO testgeo (name, geo) VALUES (
  'line',
  ST_SetSRID(ST_MakeLine(ST_Point(-77, 39), ST_Point(-78, 40)), 4326)
);
```

### Exploring Geometry Data

```sql
-- List all spatial tables
SELECT * FROM geometry_columns;

-- Get geometry type
SELECT name, ST_GeometryType(geo) FROM testgeo;

-- Get WKT representation
SELECT name, ST_AsText(geo) FROM testgeo;

-- Get lat/long of a point
SELECT name, ST_X(geo) AS x, ST_Y(geo) AS y FROM testgeo WHERE name LIKE '%point1%';

-- Area of polygons
SELECT name, ST_Area(geo) FROM testgeo WHERE ST_GeometryType(geo) = 'ST_Polygon';

-- Length of lines
SELECT name, ST_Length(geo) AS length FROM testgeo WHERE ST_GeometryType(geo) = 'ST_LineString';
```

### SRID Management

```sql
-- Check SRID
SELECT name, ST_SRID(geo) AS srid, ST_GeometryType(geo) AS geometry FROM testgeo;

-- Set SRID for rows missing it
UPDATE testgeo SET geo = ST_SetSRID(geo, 4326) WHERE ST_SRID(geo) = 0;
```

### Geometry vs. Geography

| | Geometry | Geography |
|---|---|---|
| Math model | Flat (Euclidean) | Spherical (real Earth) |
| Units | Depends on projection | Meters |
| Use case | Drawing, shapes, spatial relationships, indexing | Distance calculations, "within X km", delivery radii |
| Performance | Faster | Slower |

**Best practice — store as geometry, calculate as geography:**

```sql
-- Store
geom GEOMETRY(Point, 4326)

-- Calculate distance in real-world meters
SELECT ST_Length(geo::geography) FROM testgeo WHERE name = 'line';
```

### Reprojecting with ST_Transform

```sql
SELECT name,
  ST_X(geo) AS old_x, ST_Y(geo) AS old_y,
  ST_X(ST_Transform(geo, 26918)) AS new_x,
  ST_Y(ST_Transform(geo, 26918)) AS new_y
FROM testgeo WHERE name = 'point 2';
```

---

## 3. Real World Data — NYC Dataset

### Loading the NYC Backup

```bash
# If using Docker
docker exec -it postgis16 pg_restore -U postgres -d nycdat /data/data/nyc_data.backup
```

> ⚠️ Make sure PostGIS extension is already enabled on the target database before restoring.

### Inspecting the Data

```sql
-- Check geometry types for all tables
SELECT * FROM geometry_columns;

-- Check columns for a specific table
SELECT column_name, data_type, udt_name
FROM information_schema.columns
WHERE table_name = 'nyc_census_blocks';
```

### SQL Queries on NYC Data

```sql
-- First 10 census blocks
SELECT * FROM nyc_census_blocks LIMIT 10;

-- Count of streets
SELECT count(*) FROM nyc_streets;

-- Streets starting with 'A'
SELECT count(*) FROM nyc_streets WHERE name LIKE 'A%';

-- Total population
SELECT sum(popn_total) FROM nyc_census_blocks;

-- Population by borough
SELECT sum(popn_total) AS pop, boroname AS name
FROM nyc_census_blocks
GROUP BY boroname;

-- Most populated borough
SELECT sum(popn_total) AS pop, boroname AS name
FROM nyc_census_blocks
GROUP BY boroname
ORDER BY pop DESC LIMIT 1;

-- Race % by borough
SELECT boroname,
  100.0 * (sum(popn_white)/sum(popn_total)) AS white_pct,
  100.0 * (sum(popn_black)/sum(popn_total)) AS black_pct,
  100.0 * (sum(popn_asian)/sum(popn_total)) AS asian_pct,
  100.0 * (sum(popn_other)/sum(popn_total)) AS other_pct
FROM nyc_census_blocks
GROUP BY boroname;
```

### Spatial Queries on NYC Data

```sql
-- Area of West Village neighborhood
SELECT ST_Area(geom) FROM nyc_neighborhoods WHERE name = 'West Village';

-- Largest neighborhood
SELECT name, ST_Area(geom) FROM nyc_neighborhoods ORDER BY ST_Area(geom) DESC LIMIT 1;

-- 3 longest streets
SELECT name, ST_Length(geom) FROM nyc_streets ORDER BY ST_Length(geom) DESC LIMIT 3;

-- Northernmost subway station
SELECT name, ST_AsText(geom), ST_Y(geom) AS y FROM nyc_subway_stations ORDER BY ST_Y(geom) DESC LIMIT 1;

-- Total length of all streets
SELECT sum(ST_Length(geom)) FROM nyc_streets;
```

---

## 4. Spatial Joins

A **spatial join** links rows from two tables based on **where** their geometries are — not on IDs.

### Join Types Recap

| Type | Keeps |
|---|---|
| `INNER` | Only matching rows |
| `LEFT` | Left table rows + NULLs for no match |
| `LATERAL` | Runs a subquery once per row of the left table (best for KNN/proximity) |

### Examples

```sql
-- Streets in Staten Island
SELECT * FROM nyc_streets s
JOIN nyc_neighborhoods n ON ST_Intersects(s.geom, n.geom)
WHERE n.boroname = 'Staten Island';

-- Subway stations in Manhattan
SELECT t.name, ST_AsText(t.geom), n.boroname
FROM nyc_neighborhoods n
JOIN nyc_subway_stations t ON ST_Intersects(n.geom, t.geom)
WHERE n.boroname = 'Manhattan';

-- Crimes within 200 units of each subway station
SELECT t.name, count(*) AS ccount
FROM nyc_subway_stations t
JOIN nyc_homicides h ON ST_DWithin(t.geom, h.geom, 200)
GROUP BY t.name
ORDER BY ccount DESC;

-- Total street length by borough
SELECT n.boroname,
  sum(ST_Length(ST_CollectionExtract(ST_Intersection(s.geom, n.geom), 2))) AS segment_len
FROM nyc_neighborhoods n
JOIN nyc_streets s ON ST_Intersects(n.geom, s.geom)
GROUP BY n.boroname
ORDER BY segment_len DESC;

-- Nearest street for each crime (within 50 units)
SELECT DISTINCT ON (c.id) c.id, s.name
FROM nyc_homicides c
JOIN nyc_streets s ON ST_DWithin(c.geom, s.geom, 50)
ORDER BY c.id, ST_Distance(c.geom, s.geom);
```

---

## 5. Spatial Relationships

| Function | Description |
|---|---|
| `ST_Contains` | Strict inside |
| `ST_Within` | Reverse of Contains |
| `ST_Intersects` | Any overlap |
| `ST_Disjoint` | No overlap |
| `ST_Touches` | Boundary only |
| `ST_Overlaps` | Partial overlap |
| `ST_Equals` | Exactly the same |
| `ST_Crosses` | Cross through |
| `ST_Distance` | Distance between geometries |
| `ST_DWithin` | Within a given distance |

### Test Data Example

```sql
WITH
poly AS (SELECT ST_GeomFromText('POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))', 4326) AS geom),
pt_inside AS (SELECT ST_GeomFromText('POINT(5 5)', 4326) AS geom),
pt_edge AS (SELECT ST_GeomFromText('POINT(0 5)', 4326) AS geom),
pt_outside AS (SELECT ST_GeomFromText('POINT(20 20)', 4326) AS geom)

-- Test: point inside polygon
-- SELECT ST_Intersects(pt_inside.geom, poly.geom) FROM poly, pt_inside;

-- Test: ST_Contains (edge point is NOT contained)
-- SELECT ST_Contains(poly.geom, pt_edge.geom) FROM poly, pt_edge;

-- Test: ST_Distance
-- SELECT ST_Distance(pt_inside.geom, pt_outside.geom) FROM pt_inside, pt_outside;
```

---

## 6. Spatial Indexes

### PostgreSQL Index Types

| Type | Used For |
|---|---|
| **B-Tree** | IDs, names, dates, numbers (default) |
| **GiST** | PostGIS geometries — indexes bounding boxes |
| **BRIN** | Big, naturally ordered / append-only data |
| **SP-GiST** | Optimized for large point-only datasets |

### Creating a Spatial Index

```sql
CREATE INDEX idx_geom ON table_name USING GIST (geom);
```

### Performance Impact

| Scenario | Comparisons |
|---|---|
| No index | N × M (e.g., 100,000,000) |
| With spatial index | N × small constant (e.g., ~20,000) |

### Spatial Functions That USE Indexes

`ST_Intersects`, `ST_Contains`, `ST_Within`, `ST_DWithin`, `ST_Covers`, `ST_CoveredBy`

> ⚠️ `ST_Distance` does **not** use indexes for filtering.

### Creating NYC Indexes

```sql
CREATE INDEX idx_nyc_census_blocks ON nyc_census_blocks USING GIST(geom);
CREATE INDEX idx_nyc_neighborhoods ON nyc_neighborhoods USING GIST(geom);
```

### Index-Only Spatial Query (Approximate, Very Fast)

```sql
-- Uses bounding box (&&) only — fast but approximate
SELECT sum(popn_total)
FROM nyc_neighborhoods n
JOIN nyc_census_blocks b ON n.geom && b.geom
WHERE n.name = 'West Village';

-- Uses ST_Intersects — exact result
SELECT sum(popn_total)
FROM nyc_neighborhoods n
JOIN nyc_census_blocks b ON ST_Intersects(n.geom, b.geom)
WHERE n.name = 'West Village';
```

---

## 7. KNN Search

K-Nearest Neighbor search using the `<->` operator — uses GiST index internally.

```sql
-- 5 nearest homicides to a point
SELECT * FROM nyc_homicides
ORDER BY geom <-> ST_SetSRID(ST_Point(592160, 4502211), 26918)
LIMIT 5;

-- Same but with a hard 200-unit cutoff
SELECT * FROM nyc_homicides
WHERE ST_DWithin(geom, ST_SetSRID(ST_Point(592160, 4502211), 26918), 200)
ORDER BY ST_Distance(geom, ST_SetSRID(ST_Point(592160, 4502211), 26918));
```

### LATERAL JOIN for Nearest-Neighbor Per Row

```sql
-- Nearest street for each crime
SELECT c.id, c.weapon, c.incident_d, near_street.name, c.geom
FROM nyc_homicides c
JOIN LATERAL (
  SELECT s.name
  FROM nyc_streets s
  WHERE ST_DWithin(s.geom, c.geom, 50)
  ORDER BY c.geom <-> s.geom
  LIMIT 1
) near_street ON true;

-- Total homicides per street
WITH crime_points AS (
  SELECT c.id, near_street.name
  FROM nyc_homicides c
  JOIN LATERAL (
    SELECT s.name FROM nyc_streets s
    WHERE ST_DWithin(s.geom, c.geom, 50)
    ORDER BY c.geom <-> s.geom
    LIMIT 1
  ) near_street ON true
)
SELECT count(*) AS number_of_crimes, name
FROM crime_points
GROUP BY name
ORDER BY number_of_crimes DESC;
```

> **LEFT vs LATERAL:**  
> `LEFT JOIN` = a row-preservation rule ("keep crimes even if no match").  
> `LATERAL` = a row-by-row execution rule ("for each crime, compute the best match").

---

## 8. ANALYZE and VACUUM

### ANALYZE — Update Planner Statistics

Run after bulk loads, large deletes, or major updates.

```sql
ANALYZE table_name;

-- Check table stats
SELECT * FROM pg_stat_user_tables WHERE relname = 'nyc_streets';
```

### VACUUM — Reclaim Dead Rows

Run after heavy updates/deletes.

```sql
VACUUM ANALYZE table_name;
```

> Autovacuum helps, but manual VACUUM is still important for spatial workloads.

---

## 9. Clustering on Indices

Clustering physically reorders table rows to match the index order — rows accessed together are stored close together on disk.

```sql
-- Create index first
CREATE INDEX nyc_census_block_idx_geom ON nyc_census_blocks USING GIST(geom);

-- Check performance before
EXPLAIN ANALYZE SELECT * FROM nyc_census_blocks;

-- Cluster
CLUSTER nyc_census_blocks USING nyc_census_block_idx_geom;

-- Check performance after
EXPLAIN ANALYZE SELECT * FROM nyc_census_blocks;
```

> ⚠️ Clustering is **not permanent** — new inserts won't follow the clustered order. Re-run CLUSTER periodically.

---

## 10. PostGIS Validity

### Why Validity Matters

- Points and lines are always valid.
- **Polygons and MultiPolygons can be invalid.**

A valid polygon must: close its rings, keep holes inside the outer ring, have no self-intersecting rings, and rings can only touch at points.

```sql
-- Find invalid polygons
SELECT name, boroname, geom FROM nyc_neighborhoods WHERE NOT ST_IsValid(geom);

-- Find the reason
SELECT name, boroname, ST_IsValidReason(geom) AS why FROM nyc_neighborhoods WHERE NOT ST_IsValid(geom);

-- Fix invalid geometry
SELECT name, ST_MakeValid(geom) FROM nyc_neighborhoods WHERE NOT ST_IsValid(geom);

-- Update in place
UPDATE nyc_neighborhoods SET geom = ST_MakeValid(geom) WHERE NOT ST_IsValid(geom);
```

### Check for Duplicate Geometries

```sql
SELECT a.name AS name_a, b.name AS name_b
FROM nyc_neighborhoods a
JOIN nyc_neighborhoods b ON ST_Equals(a.geom, b.geom);
```

---

## 11. More Spatial Functions

### ST_Union — Merge Geometries

```sql
-- Union all census blocks into one geometry
SELECT ST_Union(geom) FROM nyc_census_blocks;

-- Create a new table merged by borough
CREATE TABLE nyc_neighborhoods_by_boron AS
SELECT boroname, ST_Union(geom) AS geom
FROM nyc_neighborhoods
GROUP BY boroname;
```

### ST_Dump — Break Multi-Geometry into Singles

```sql
-- Break multi-line streets into individual segments
CREATE TABLE nyc_streets_dump AS
SELECT s.name, (ST_Dump(s.geom)).geom AS geom
FROM nyc_streets s;
```

### ST_Centroid & ST_Contains — Avoid Double Counting

> When joining census blocks to neighborhoods, use `ST_Centroid` to ensure each block is assigned to only **one** neighborhood:

```sql
-- ❌ Can double-count: a block touching two neighborhoods is counted twice
SELECT sum(b.popn_total)
FROM nyc_neighborhoods n
JOIN nyc_census_blocks b ON ST_Intersects(n.geom, b.geom);

-- ✅ Correct: block assigned to the neighborhood containing its center
SELECT sum(b.popn_total)
FROM nyc_neighborhoods n
JOIN nyc_census_blocks b ON ST_Contains(n.geom, ST_Centroid(b.geom));
```

### Example: % of Population Near a Subway Station

```sql
WITH blocks_with_neighborhood AS (
  SELECT b.blkid, b.popn_total, n.name AS neighborhood, b.geom
  FROM nyc_census_blocks b
  JOIN nyc_neighborhoods n ON ST_Contains(n.geom, ST_Centroid(b.geom))
),
blocks_near_subway AS (
  SELECT DISTINCT ON (b.blkid) b.blkid, b.popn_total, b.neighborhood
  FROM blocks_with_neighborhood b
  JOIN nyc_subway_stations s ON ST_DWithin(b.geom, s.geom, 500)
),
neighborhood_total AS (
  SELECT neighborhood, sum(popn_total) AS total_pop
  FROM blocks_with_neighborhood GROUP BY neighborhood
),
subway_total AS (
  SELECT neighborhood, sum(popn_total) AS subway_total
  FROM blocks_near_subway GROUP BY neighborhood
)
SELECT s.neighborhood, 100 * s.subway_total / n.total_pop AS percentage
FROM neighborhood_total n
JOIN subway_total s ON n.neighborhood = s.neighborhood
WHERE total_pop > 0
ORDER BY percentage DESC;
```

---

## 12. DE-9IM — Spatial Relationship Model

`ST_Relate` checks the exact spatial relationship between two geometries using a 9-character matrix (Dimensionally Extended 9-Intersection Model).

```sql
-- Get the DE-9IM string
SELECT ST_Relate(
  ST_GeomFromText('POINT(5 5)'),
  ST_GeomFromText('POLYGON((0 0,10 0,10 10,0 10,0 0))')
);
-- Result: 0FFFFF212

-- Test against a known pattern
SELECT ST_Relate(
  ST_GeomFromText('POINT(5 5)'),
  ST_GeomFromText('POLYGON((0 0,10 0,10 10,0 10,0 0))'),
  'T*F**F***'  -- Within pattern
);
```

### Common Patterns

| Relationship | Pattern |
|---|---|
| Contains | `T*****FF*` |
| Within | `T*F**F***` |
| Touches | `FT*******` |
| Crosses | `T*T******` |
| Disjoint | `FF*FF****` |

> Pattern characters: `T` = must intersect, `F` = must NOT intersect, `*` = don't care

---

## 13. Linear Referencing

Linear Referencing describes locations along a line using a **measure** instead of storing separate geometries.

> Example: "The accident happened at kilometer 42 along Highway 6."

### Key Functions

| Function | Purpose |
|---|---|
| `ST_LineLocatePoint` | Get measure (0–1) where a point falls along a line |
| `ST_LineInterpolatePoint` | Convert a measure back into a point |
| `ST_LineSubstring` | Extract a portion of a line |
| `ST_LocateAlong` | Get geometry at an M value |
| `ST_LocateBetween` | Get geometry between two M values |
| `ST_AddMeasure` | Add M values along a geometry |

```sql
-- Where does a point fall along a line? (returns 0.0 to 1.0)
SELECT ST_LineLocatePoint('LINESTRING(0 0, 2 2)', 'POINT(1 1)');

-- Convert measure back to a point
SELECT ST_AsText(ST_LineInterpolatePoint('LINESTRING(0 0, 2 2)', 0.5));

-- Extract a sub-line from 25% to 75%
SELECT ST_AsText(ST_LineSubstring('LINESTRING(0 0, 2 2)', 0.25, 0.75));

-- Add M values (0 to 100) along a line
SELECT ST_AsText(ST_AddMeasure('LINESTRING(0 0, 10 0)', 0, 100));
```

### Convert Subway Stations to Linear Events Along Streets

```sql
WITH near AS (
  SELECT streets.gid AS streets_gid, subways.gid AS subways_gid,
    ST_Distance(streets.geom, subways.geom) AS distance,
    subways.geom AS subways_geom,
    ST_GeometryN(streets.geom, 1) AS streets_geom
  FROM nyc_streets streets
  JOIN nyc_subway_stations subways ON ST_DWithin(streets.geom, subways.geom, 100)
  ORDER BY subways_gid, distance ASC
)
SELECT DISTINCT ON (subways_gid)
  subways_gid, streets_gid,
  ST_LineLocatePoint(streets_geom, subways_geom) AS measure,
  distance
FROM near;
```

---

## 14. PostgreSQL Functions & Procedures

### Function Syntax

```sql
CREATE OR REPLACE FUNCTION schema.function_name(param_name param_type)
RETURNS return_type
LANGUAGE sql | plpgsql
IMMUTABLE | STABLE | VOLATILE
STRICT
AS $$
  -- function body
$$;
```

### LANGUAGE sql vs. plpgsql

| | `LANGUAGE sql` | `LANGUAGE plpgsql` |
|---|---|---|
| IF/ELSE | ❌ | ✅ |
| Loops | ❌ | ✅ |
| Variables | ❌ | ✅ |
| Multiple statements | ❌ | ✅ |
| Speed | Very fast | Flexible |

### Volatility

| Level | Meaning | Examples |
|---|---|---|
| `IMMUTABLE` | Same input → same output forever. Safe for indexes. | `LOWER(text)`, `ST_Length(geom)` |
| `STABLE` | Same result within one query. Can read tables. | `NOW()`, aggregate queries |
| `VOLATILE` | Result may change every call. Writes data. | `random()`, `INSERT/UPDATE` |

### Example: Get Streets in a Neighborhood

```sql
CREATE OR REPLACE FUNCTION public.streets_in_neighborhoods(neighborhood_name TEXT)
RETURNS TABLE (street_id INT, street_name TEXT)
LANGUAGE sql STABLE STRICT
AS $$
  SELECT s.id, s.name
  FROM nyc_streets s
  JOIN nyc_neighborhoods n ON ST_Intersects(s.geom, n.geom)
  WHERE lower(n.name) = lower(neighborhood_name)
$$;

-- Test
SELECT * FROM streets_in_neighborhoods('east village');
```

### FUNCTION vs. PROCEDURE

| | Function | Procedure |
|---|---|---|
| Returns a value | ✅ | ❌ |
| Used in SELECT/WHERE | ✅ | ❌ |
| Can COMMIT transactions | ❌ | ✅ |
| Best for | Calculations, queries | Data migration, ETL, batch updates |

```sql
-- Stored procedure example
CREATE OR REPLACE PROCEDURE update_street_length()
LANGUAGE plpgsql AS $$
BEGIN
  UPDATE nyc_streets SET newlength = ST_Length(geom);
  COMMIT;
END;
$$;

CALL update_street_length();
```

---

## 15. Triggers & Rules

### What is a Trigger?

A trigger automatically runs a function when a specific table event occurs (INSERT, UPDATE, DELETE). It has two parts: a **trigger function** and a **trigger** that attaches it to a table.

### BEFORE vs. AFTER Triggers

| | BEFORE | AFTER |
|---|---|---|
| Purpose | Modify the row before saving | React to the change (logging, auditing) |
| Row access | `NEW.column` | `NEW.column` and `OLD.column` |

### Trigger Example — Auto-Calculate Street Length

```sql
-- 1) Trigger function
CREATE OR REPLACE FUNCTION update_streets_tgr_func()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  NEW.newlength := ST_Length(NEW.geom);
  RETURN NEW;
END;
$$;

-- 2) Attach trigger to table
CREATE TRIGGER trg_streets_update
BEFORE INSERT OR UPDATE ON nyc_streets
FOR EACH ROW EXECUTE FUNCTION update_streets_tgr_func();

-- Test
UPDATE nyc_streets SET geom = geom WHERE gid = 1;
SELECT gid, newlength FROM nyc_streets WHERE gid = 1;
```

### Rules

Rules silently rewrite SQL before PostgreSQL executes it.

```sql
-- Log all deletes to a separate table
CREATE RULE log_delete AS
ON DELETE TO products
DO ALSO
INSERT INTO delete_log(product_name, deleted_at) VALUES (OLD.name, now());
```

---

## 16. Tracking Edit History

Every edit creates a new version using `TSTZRANGE` (timestamp range). Query any point in history using `valid_range`.

### Setup

```sql
-- 1) Create history table from existing table
CREATE TABLE nyc_streets_history AS SELECT * FROM nyc_streets;

-- 2) Add history columns
ALTER TABLE nyc_streets_history
  ADD COLUMN created_by TEXT,
  ADD COLUMN deleted_by TEXT,
  ADD COLUMN hid SERIAL PRIMARY KEY,
  ADD COLUMN valid_range TSTZRANGE;

-- 3) Initialize existing rows
UPDATE nyc_streets_history
SET created_by = current_user, valid_range = tstzrange(now(), NULL);
```

### INSERT Trigger

```sql
CREATE OR REPLACE FUNCTION nyc_streets_insert()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO nyc_streets_history (gid, name, oneway, type, geom, valid_range, created_by)
  VALUES (NEW.gid, NEW.name, NEW.oneway, NEW.type, NEW.geom, tstzrange(now(), NULL), current_user);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_nyc_street_insert
AFTER INSERT ON nyc_streets FOR EACH ROW EXECUTE FUNCTION nyc_streets_insert();
```

### DELETE Trigger

```sql
CREATE OR REPLACE FUNCTION nyc_streets_delete()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE nyc_streets_history
  SET valid_range = tstzrange(lower(valid_range), now()), deleted_by = current_user
  WHERE gid = OLD.gid AND valid_range @> now();
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_nyc_streets_delete
AFTER DELETE ON nyc_streets FOR EACH ROW EXECUTE FUNCTION nyc_streets_delete();
```

### UPDATE Trigger

```sql
CREATE OR REPLACE FUNCTION nyc_streets_update()
RETURNS TRIGGER AS $$
BEGIN
  -- Close old version
  UPDATE nyc_streets_history
  SET valid_range = tstzrange(lower(valid_range), now()), deleted_by = current_user
  WHERE gid = OLD.gid AND valid_range @> now();

  -- Insert new version
  INSERT INTO nyc_streets_history (gid, name, type, geom, valid_range, created_by)
  VALUES (NEW.gid, NEW.name, NEW.type, NEW.geom, tstzrange(now(), NULL), current_user);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_streets_updates
AFTER UPDATE ON nyc_streets FOR EACH ROW EXECUTE FUNCTION nyc_streets_update();
```

### Querying History

```sql
-- All active (current) records
SELECT * FROM nyc_streets_history WHERE valid_range @> now();

-- All inactive (expired) records
SELECT * FROM nyc_streets_history WHERE NOT (valid_range @> now());
```

---

## 17. PostGIS Topology

### Why Topology?

Without topology, each polygon stores its own private copy of shared borders. Moving one border creates **gaps and overlaps** in neighboring polygons.

With topology, neighboring polygons **share the same edge** — move it once and both update automatically.

**Real-world use cases:** land parcels, neighborhood/borough boundaries, road networks.

### The Three Building Blocks

| Element | Symbol | Description |
|---|---|---|
| Node | ● | A corner, endpoint, or junction |
| Edge | ─ | A segment connecting exactly two nodes |
| Face | ▭ | A surface enclosed by edges |

Hierarchy: **Nodes → Edges → Faces → Polygons**

### Topology Storage (inside `nyc_topo` schema)

| Table | Contents |
|---|---|
| `edge_data` | The actual line segments |
| `edge` | A view on edge_data |
| `node` | All endpoints and isolated points |
| `face` | Polygon surfaces (stored as bounding boxes only) |
| `relation` | Links topogeometries to their elements |

### Building a Topology — Step by Step

```sql
-- Step 1: Enable extension and create topology
CREATE EXTENSION postgis_topology;
SELECT topology.CreateTopology('nyc_topo', 26918, 0.5);
SELECT * FROM topology.topology;

-- Step 2: Create table and add topogeometry column
CREATE TABLE nyc_neighborhoods_t (boroname VARCHAR, name VARCHAR, PRIMARY KEY (boroname, name));
SELECT topology.AddTopoGeometryColumn('nyc_topo', 'public', 'nyc_neighborhoods_t', 'topo', 'POLYGON');

-- Step 3: Populate from regular geometry
INSERT INTO nyc_neighborhoods_t
SELECT boroname, name, topology.toTopoGeom(geom, 'nyc_topo', 1)
FROM nyc_neighborhoods WHERE ST_IsValid(geom);

-- Step 4: Verify
SELECT 'edge', count(*) FROM nyc_topo.edge
UNION ALL SELECT 'node', count(*) FROM nyc_topo.node
UNION ALL SELECT 'face', count(*) FROM nyc_topo.face;

SELECT * FROM topology.TopologySummary('nyc_topo');
```

### Reading the Topogeometry Tuple

A `topo` column stores an **address**, not a shape: `(topology_id, layer_id, topogeom_id, type_code)`

| Position | Field | Meaning |
|---|---|---|
| 1st | Topology ID | Which named topology |
| 2nd | Layer ID | Which table/column |
| 3rd | Topogeom ID | Which specific feature |
| 4th | Type Code | 1=Point, 2=Line, 3=Polygon, 4=Collection |

```sql
-- Cast to geometry to see the actual shape
SELECT boroname, topo::geometry FROM nyc_neighborhoods_t;
```

### Detecting & Fixing Topology Conflicts

```sql
-- Find overlapping neighborhoods (shared faces)
SELECT array_agg(DISTINCT b.name) AS neighborhoods, te
FROM nyc_neighborhoods_t b,
     topology.GetTopoGeomElements(b.topo) AS te
GROUP BY te
HAVING count(DISTINCT b.name) > 1;

-- Inspect the shared area
SELECT array_agg(DISTINCT b.name) AS neighborhoods, te,
  topology.ST_GetFaceGeometry('nyc_topo', te[1]) AS geom
FROM nyc_neighborhoods_t b,
     topology.GetTopoGeomElements(b.topo) AS te
WHERE te[2] = 3
GROUP BY te
HAVING count(DISTINCT b.name) > 1;

-- Remove the shared face from the incorrect feature
UPDATE nyc_neighborhoods_t
SET topo = topology.TopoGeom_remElement(topo, ARRAY[83, 3]::topoelement)
WHERE name = 'Eastchester';
```

### Quick Reference — Topology Functions

| Function | Purpose |
|---|---|
| `CreateTopology()` | Create a new topology schema |
| `AddTopoGeometryColumn()` | Add a topogeometry column to a table |
| `toTopoGeom()` | Convert regular geometry to topogeometry |
| `TopologySummary()` | Human-readable element counts |
| `GetTopoGeomElements()` | Explode a topogeometry into element pairs |
| `ST_GetFaceGeometry()` | Reconstruct a face polygon from surrounding edges |
| `TopoGeom_remElement()` | Remove an element from a topogeometry |

---

## 📝 Notes

- This manual is designed to accompany video lessons, not as a standalone textbook.
- All SQL examples are written for **PostgreSQL + PostGIS**.
- The NYC dataset used throughout is available at the GitHub link above.

---

*PostGIS in Arabic — Ahmad Saleh © 2026*
