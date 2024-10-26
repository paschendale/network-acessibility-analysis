This tutorial uses PostgreSQL version 13, along with the latest versions of PostGIS and pgRouting. To install the dependencies, use your package manager (in Linux or Mac OS) or the installation packages from EnterpriseDB (in Windows OS). It is recommended to use pgAdmin 4 (installed along with the EnterpriseDB packages or separately on Linux and Mac) to run the code below in the database.

In the `sql` folder, there is a `routing.sql` file, containing a full dump of the database used in this tutorial; optionally, you can restore it through the psql command line.

### Database Preparation

1. **Create Database and Add Extensions to the Database**

   ```sql
   CREATE EXTENSION postgis;
   CREATE EXTENSION pgrouting;
   ```
2. Upload the Network to PostGIS and Configure Network Topologies

> NOTE: Rename the file to network_l before or after upload. The upload can be done either via QGIS by dragging it into the created database connection or using the shapefile upload tool for PostGIS. In the example folder, you will find a network shapefile that can be used for this tutorial.

```sql
-- Creating fields to store the IDs of the vertices at the beginning and end of the edges
ALTER TABLE network_l ADD source INT4;
ALTER TABLE network_l ADD target INT4;

-- Renaming the geometry column to the_geom to comply with the pgRouting standard. This step is optional if the geometry column name is already 'the_geom'
ALTER TABLE network_l RENAME COLUMN geom TO the_geom;

-- Creating network topology
SELECT pgr_createTopology('network_l', 1);
```

After the procedure, the query will display an "OK" message, indicating everything went successfully. If it returns a "FAIL" message, check the Messages tab in pgAdmin to understand the error reason.

NOTE: Running `pgr_createTopology` creates a secondary table with the template `table_name_edges_vertices_pgr`.

3. Find the Central Nodes of the Edges

In QGIS, open the toolbox using the shortcut `CTRL + Alt + T`. Look for the tool in `GDAL > Vector Geoprocessing > Points along the lines`.

Fill in the fields with the following parameters:

- Input Layer: `network_l`
- Geometry Column Name: `the_geom`
- Distance from Line: `0.5`
- Save the created layer as `network_mid_nodes_p` on disk in the `.gpkg` format, and upload it to the database later.

NOTE: Using `.gpkg` to save files on disk is essential to preserve column names and table encoding.

4. Split the Edges (`network_l`) at the Locations of the Central Nodes (`network_mid_nodes_p`)

NOTE: For this step, you will need the QGIS Networks plugin.

Save the `network_l` layer on disk in `.gpkg` format as `network_renoded_l`. In the QGIS toolbox, search for `Networks > Connect Nodes to Lines`.

Fill in the fields with the following parameters:

- Network: `network_renoded_l`
- Nodes: `network_mid_nodes_p`

The tool will edit the `network_renoded_l` file, creating vertices at the center of each edge and attempting to save the edits. QGIS will display an error indicating it couldn’t save the file due to primary key conflicts, but this can be ignored for now. Drag the `network_renoded_l` file, still in edit mode, into the database in QGIS.

5. Configure Topologies in the `network_renoded_l` Table
``` sql
-- Recreate the source and target fields for the new network
ALTER TABLE network_renoded_l DROP source;
ALTER TABLE network_renoded_l DROP target;
ALTER TABLE network_renoded_l ADD source INT4;
ALTER TABLE network_renoded_l ADD target INT4;

-- Recreate table IDs, retaining the network_l ID as id_edge
ALTER TABLE network_renoded_l RENAME COLUMN id TO id_edge;
ALTER TABLE network_renoded_l DROP IF EXISTS fid;
ALTER TABLE network_renoded_l DROP IF EXISTS id_0;
ALTER TABLE network_renoded_l ADD COLUMN id SERIAL PRIMARY KEY;

-- Rebuild topology
ALTER TABLE network_renoded_l RENAME COLUMN geom TO the_geom;
SELECT pgr_createTopology('network_renoded_l', 1);
```

6. Update the Original Edge Table (`network_l`) with the Central Vertices Representing Them
```sql
ALTER TABLE network_l ADD central_vertex INT4;
UPDATE network_l
SET central_vertex = subq.vertex_id
FROM (
   SELECT b.id AS vertex_id, a.id AS edge_id
   FROM network_mid_nodes_p a, network_renoded_l_vertices_pgr b
   WHERE a.geom = b.the_geom
) subq
WHERE subq.edge_id = network_l.id;
```

7. Update the New Vertices Table with the Edges They Represent, If They Are Central Vertices
```sql
ALTER TABLE network_renoded_l_vertices_pgr ADD edge_id INT4;
UPDATE network_renoded_l_vertices_pgr
SET edge_id = network_l.edge_id
FROM (
   SELECT id AS edge_id, central_vertex
   FROM network_l
) network_l
WHERE central_vertex = id;
```

### Analysis

From this point, perform analyses on the cost matrix of `network_renoded_l`. The analyses are recombinations and joins of the resulting table from the `pgr_dijkstraCostMatrix` function (see reference) with other tables in the database. These can be done separately or in a single query that creates a materialized view called `network_l_analysis`, accessible via QGIS. If you’re only interested in the final result, skip to item 12.

8. Create the Proximity Matrix Between Edges (Resulting Matrix of the `pgr_dijkstraCostMatrix Function`)
```sql
SELECT start_vid.edge_id AS edge_id_start,
       end_vid.edge_id AS edge_id_end,
       agg_cost/2 AS distance
FROM pgr_dijkstraCostMatrix(
   'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM network_renoded_l',
   (SELECT array_agg(central_vertex) FROM network_l WHERE central_vertex IS NOT NULL)
) matrix
LEFT JOIN network_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
LEFT JOIN network_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid;
```

9. Extract the Direct Link Matrix for Each Edge
```sql
SELECT edge_id_start, SUM(distance) AS direct_links
FROM (
   SELECT start_vid.edge_id AS edge_id_start,
          end_vid.edge_id AS edge_id_end,
          agg_cost/2 AS distance
   FROM pgr_dijkstraCostMatrix(
      'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM network_renoded_l',
      (SELECT array_agg(central_vertex) FROM network_l WHERE central_vertex IS NOT NULL)
   ) matrix
   LEFT JOIN network_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
   LEFT JOIN network_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid
) subq
WHERE distance <= 1
GROUP BY edge_id_start;
```

10. Extract the Distance Matrix to the Farthest Edge in the Network
```sql
SELECT DISTINCT ON (start_vid.edge_id)
       start_vid.edge_id AS edge_id_start,
       end_vid.edge_id AS edge_id_end,
       agg_cost/2 AS distance
FROM pgr_dijkstraCostMatrix(
   'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM network_renoded_l',
   (SELECT array_agg(central_vertex) FROM network_l WHERE central_vertex IS NOT NULL)
) matrix
LEFT JOIN network_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
LEFT JOIN network_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid
ORDER BY start_vid.edge_id, agg_cost/2 DESC;
```

11. Extract the Weighted Connection Matrix for n Orders of an Edge (Obtain the Connection Matrix per Edge for Each Order)
```sql
SELECT edge_id_start, distance AS order,
       COUNT(edge_id_start) AS n_connections,
       COUNT(edge_id_start)*distance AS weighted_connections
FROM (
   SELECT start_vid.edge_id AS edge_id_start,
          end_vid.edge_id AS edge_id_end,
          agg_cost/2 AS distance
   FROM pgr_dijkstraCostMatrix(
      'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM network_renoded_l',
      (SELECT array_agg(central_vertex) FROM network_l WHERE central_vertex IS NOT NULL)
   ) matrix
   LEFT JOIN network_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
   LEFT JOIN network_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid
) subq
GROUP BY edge_id_start, distance
ORDER BY edge_id_start, distance;
```

12. Create a Materialized View with Edge Analysis Results
```sql
CREATE MATERIALIZED VIEW network_l_analysis AS
SELECT a.edge_id,
       b.direct_links,
       c.distance AS farthest_edge_distance,
       d.n_connections,
       d.weighted_connections
FROM network_l a
LEFT JOIN (
   -- query with direct links
) b ON b.edge_id_start = a.edge_id
LEFT JOIN (
   -- query with farthest distance
) c ON c.edge_id_start = a.edge_id
LEFT JOIN (
   -- query with weighted connections
) d ON d.edge_id_start = a.edge_id;
```

After running the network_l_analysis view, you can export it to disk in `.gpkg` format to upload to QGIS and complete your spatial analysis through the layers and results generated in the queries above.
