# Cálculo índice de accesibilidad por medio de transporte 
ESPECIFICACIONES: 

Postgis 3
Pgrouting 
PostgreSQL 13
Qgis 3.1.4

========================================
## ETAPA 1: GENERAR CLUSTERS DE HOSPITALES 
### NODO MAS CERCADO DE LA RED AL CENTROIDE DEL CLUSTER: Se asigna como primer parte del procesamiento el id del nodo  más cercano de la red a los puntos de áreas verdes 
``` sql
alter table areas_verdes add column closest_node bigint; 
update areas_verdes set closest_node = c.closest_node
from  
(select b.id as id_centroid, (
  SELECT a.id
  FROM ways_vertices_pgr As a
  ORDER BY b.geom <-> a.the_geom LIMIT 1
)as closest_node
from  areas_verdes b) as c
where c.id_centroid = areas_verdes .id
```
## ETAPA 2: MODELO DE COSTOS POR TIPO DE TRANSPORTE
En la siguiente tabla se muestran las categorías por tipo de vialidad utilizadas en cuanto a la velocidad máxima permitida. Para el modelo de costos en tiempo de traslado se utilizó un factor para priorizar las cirulación por por vialidades optimas por tipo de tranporte.

|class_id | type_id | name | priority | default_maxspeed|
|  :---:  | :---:   | :---: |     :---:      |    :---:    |   
|201 |       2 | lane              |        1 |               50|
|204 |       2 | opposite          |        1 |               50|
|203 |       2 | opposite_lane     |        1 |               50|
|202 |       2 | track             |        1 |               50|
|120 |       1 | bridleway         |        1 |               50|
|116 |       1 | bus_guideway      |        1 |               50|
|121 |       1 | byway             |        1 |               50|
|118 |       1 | cycleway          |        1 |               50|
|119 |       1 | footway           |        1 |               50|
|111 |       1 | living_street     |        1 |               50|
|101 |       1 | motorway          |        1 |               50|
|103 |       1 | motorway_junction |        1 |               50|
|102 |       1 | motorway_link     |        1 |               50|
|117 |       1 | path              |        1 |               50|
|114 |       1 | pedestrian        |        1 |               50|
|106 |       1 | primary           |        1 |               50|
|107 |       1 | primary_link      |        1 |               50|
|110 |       1 | residential       |        1 |               50|
|100 |       1 | road              |        1 |               50|
|108 |       1 | secondary         |        1 |               50|
|124 |       1 | secondary_link    |        1 |               50|

```sql
alter table ways add column longitud float;
update ways set longitud = st_length(geom);

alter table ways add column time float;
update ways set time = longitud/maxspeed; 
```

```sql
alter table ways add column costo_bici float;
update ways set costo_bici =
  case
    when class_id = '121' then time*0.5
    when class_id = '118' then time*0.5
    when class_id = '119' then time*0.5
    else time * 10
  end;
```

```sql
alter table ways add column costo_carro float;
update ways set costo_carro =
  case
    when class_id = '201' then time*0.5
    when class_id = '101' then time*0.5
    when class_id = '106' then time*0.5
    when class_id = '202' then time*0.5
    when class_id = '204' then time*0.5
    when class_id = '203' then time*0.5
    when class_id = '202' then time*0.5
    else time * 10
  end;
```

```sql
alter table ways add column costo_caminando float;
update table ways set costo_caminando = longitud/9;
```


## ETAPA 2: REGIONALIZACIÓN CON BASE EN CONECTIVIDAD 

#### NODOS CON MAYOR CENTRALIDAD: se calcula la centralidad de los nodos de la red, para filtrar por los que tienen mayore valores
* Calcular Medidas de centralidad de red
``` sql
SELECT  pgr_analyzeGraph('network.ways', .1 ,'the_geom','gid','source','target');
```


## ETAPA 3: CALCULAR EL COSTO AGREGADO DE TODOS LOS NODOS DE LA RED CON MEJOR CENTRALIDAD HACIA TODAS LAS ÁREAS VERDES DE LA CIUDAD

#### AUTOMOVIL
``` sql
CREATE TABLE accesibilidad_carro AS 
SELECT b.the_geom, a.*
FROM
(SELECT DISTINCT ON (start_vid)
       start_vid, end_vid, agg_cost
FROM   (SELECT * FROM pgr_dijkstraCost(
    'select gid as id, source, target, costo_carro as cost from network.ways',
    array(select distinct(id) from (select * from ways_vertices_pgr where cnt > 40) as net ), --destino 
	array(select distinct(closest_node) from áreas_verdes), --origen
directed:=false)
) as sub
ORDER  BY start_vid, agg_cost asc) as a
JOIN network.ways_vertices_pgr b
ON a.start_vid = b.id
```
#### BICICLETA
``` sql
CREATE TABLE accesibilidad_bicicleta AS 
SELECT b.the_geom, a.*
FROM
(SELECT DISTINCT ON (start_vid)
       start_vid, end_vid, agg_cost
FROM   (SELECT * FROM pgr_dijkstraCost(
    'select gid as id, source, target, costo_bici as cost from network.ways',
    array(select distinct(id) from (select * from ways_vertices_pgr where cnt > 40) as net ), --destino 
	array(select distinct(closest_node) from áreas_verdes), --origen
directed:=false)
) as sub
ORDER  BY start_vid, agg_cost asc) as a
JOIN network.ways_vertices_pgr b
ON a.start_vid = b.id
```
#### CAMINANDO
``` sql
CREATE TABLE accesibilidad_caminando AS 
SELECT b.the_geom, a.*
FROM
(SELECT DISTINCT ON (start_vid)
       start_vid, end_vid, agg_cost
FROM   (SELECT * FROM pgr_dijkstraCost(
    'select gid as id, source, target, costo_caminando as cost from network.ways',
    array(select distinct(id) from (select * from ways_vertices_pgr where cnt > 40) as net ), --destino 
	array(select distinct(closest_node) from áreas_verdes), --origen
directed:=false)
) as sub
ORDER  BY start_vid, agg_cost asc) as a
JOIN network.ways_vertices_pgr b
ON a.start_vid = b.id
```

#### METRO
``` sql
CREATE TABLE accesibilidad_node_est AS 
SELECT b.the_geom, a.*
FROM
(SELECT DISTINCT ON (start_vid)
       start_vid, end_vid, agg_cost
FROM   (SELECT * FROM pgr_dijkstraCost(
    'select gid as id, source, target, costo_caminando as cost from network.ways',
    array(select distinct(id) from (select * from ways_vertices_pgr where cnt > 40) as net ), --destino 
	array(select distinct(closest_node) from estaciones_metro), --origen
directed:=false)
) as sub
ORDER  BY start_vid, agg_cost asc) as a
JOIN network.ways_vertices_pgr b
ON a.start_vid = b.id
```
``` sql
CREATE TABLE accesibilidad_est_av AS 
SELECT b.the_geom, a.*
FROM
(SELECT DISTINCT ON (start_vid)
       start_vid, end_vid, agg_cost
FROM   (SELECT * FROM pgr_dijkstraCost(
    'select gid as id, source, target, costo_caminando as cost from network.ways',
    array(select distinct(closest_node) from estaciones_metro), --destino 
	array(select distinct(closest_node) from areas_verdes), --origen
directed:=false)
) as sub
ORDER  BY start_vid, agg_cost asc) as a
JOIN network.ways_vertices_pgr b
ON a.start_vid = b.id
```
**Para representar el costo agregado en metro, sumar las columnas agg_cost de accesibilidad_est_av y accesibilidad_est_av
__NOTA:De la tabla resultante la columna start_vid pertenece a los nodos de la red y end_vid los nodos de destino

Por cada índice se crea un raster de costos con la herramienta IDW interpolando con la columna agg_cost. 
A partir de la herramienta point density se genera con las capa de delitos el raster de zonas peligrosas y se genera una realción para incrementar el costo en la intersección entre el rster de costos obtenido por el IDW

