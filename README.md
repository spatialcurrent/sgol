# Spatial Graph Operations Language (SGOL)

# Description

Spatial Graph Operations Language (aka **SGOL**) is a high-level language for describing graph operations, with a focus on spatial data.

This graph language follows many of the design conventions of the common [SQL](https://en.wikipedia.org/wiki/SQL).  The syntax will be familiar, but the actual reserved words are different.

This language describes a chain of graph operations that are executed as a stream.  A primary set of objects (usually Graph `elements`) are passed from clause to clause.  You can save elements to secondary sets while executing the operations by using `UPDATE` and `FETCH`.  This is somewhat similiar to how [jquery](https://jquery.com/), [gulp](http://gulpjs.com/), and more are designed for `chaining`.

**Why not just Gremlin?**

[Gremlin](http://tinkerpop.apache.org/gremlin.html) is the graph traversal language of Apache [TinkerPop](http://tinkerpop.apache.org/) and provides a common language to interface with multiple graph databases.  The key difference between **SGOL** and **Gremlin** is that an **SGOL** chain is a plain string easily built by client applications, with no dependencies.  As a plain string, it can be easily sent from web application front-ends, desktop applications, or manually input by users.  As with [CQL](http://docs.geoserver.org/latest/en/user/tutorials/cql/cql_tutorial.html), it be used directly in [QGIS](http://qgis.org/en/site/) and [OpenLayers](http://openlayers.org/) vector source urls.

**What's below?**

The language [specification](#specification), [compiler steps](#compiler-steps), and some [examples](#examples) are provided below.

# Contributing

[Spatial Current, Inc.](https://spatialcurrent.io) is currently accepting pull requests for this repository.  We'd love to have your contributions!  Please see [Contributing.md](https://github.com/spatialcurrent/sgol/blob/master/CONTRIBUTING.md) for how to get started.

# License 

This work is distributed under the **MIT License**.  See **LICENSE** file.

# Specification

## Core Clauses

The following are the `core` clauses implemented by this language: [ADD](#add), [DISCARD](#discard), [FETCH](#fetch), [HAS](#has), [INIT](#init), [LIMIT](#limit), [NAV](#nav), [OUTPUT](#output), [RELATE](#relate), [RUN](#run), and [SELECT](#select).

### ADD

`ADD` is used for adding new elements to the graph.  It is typically used as the terminal operation in a chain, since it's return value is `VOID`.  You can use it to commit elements that are created in-memory, but not commited.

**Examples**

```
... RELATE ADD
```

### DISCARD

`DISCARD` is an operation for discarding the `primary` set of elements.  You can use it to essentially start a new chain of clauses.  Importantly, you can still reference the **secondary** sets mentioned in `UPDATE`, `FETCH`, etc.  It is useful if you want to sequentially `add` or `upsert` data to the graph first and then navigate the graph with the new data.

**Examples**

```
... RELATE ADD DISCARD SELECT $PointOfInterest
```

### FETCH

`FETCH` is used for replacing the primary set of elements in the stream with those from a secondary set.  It is frequently used as a **penultimate** operation after a recursive search.

```
FETCH <id of secondary set>
```

### HAS

`HAS` is the operation for filtering the `current` set of entities to only those connected to another literal set of entities.  It is frequently used within a chain of statements.

```
... HAS INPUT $HasType pointofinteresttype_cafe OUTPUT entities
```

### INIT

`INIT` initializes a secondary set of objects, typically elements.  It is not strictly necessary if the `SGOL-compiler` automatically prepends `INIT` clauses as needed before executing a chain of operations.  See [Compiler Steps](#compiler-steps) below for more information.

```
INIT <id of secondary set>
```

### LIMIT

`LIMIT` is the operation for limiting the size of the `current` set of entities to the given value.  It is frequently used as a **penultimate** operation after a search, after **FETCH** and before **OUTPUT**.

```
... HAS INPUT $HasType pointofinteresttype_cafe LIMIT 10 OUTPUT entities
```

### NAV

`NAV` is the operation for `navigating` / `transversing` / `hopping` around the graph from entity to entity following certain edges.

```
NAV $PointOfInterest $HASTYPE pointofinteresttype_cafe
```


### OUTPUT

`Output` is used to specify the type of objects returned by the chain of operations.  This is used for encoding the response to SGOL clients.  **void** can be used to signal that there is no response (like a SQL update).  If the output type is missing or invalid for the chain, the SGOL server may throw an error.

```
OUTPUT [ bbox | edges | elements | entities | geojson | json | text | void ]
```

### RELATE

`RELATE` is used for calculating the geospatial relationship between two sets of entities, the first set being the **primary** set and the second being a **secondary** set referenced by id.  The total number of calculations is in `O(mn)` time, so your SGOL-server may cache results to increase performance.

```
... RELATE <id of secondary set> ADD OUTPUT void
```

**Full Example**

```
SELECT location_test UPDATE location DISCARD \
SELECT $PointOfInterest FILTER bbox(-180,-90,180,90) \
RELATE location ADD \
OUTPUT void
```

### RUN

`RUN` is used for directly executing graph operations known to the server.  It is frequently used at the end of a stream for transforming the output.

```
RUN GetConvexPolygon()
RUN GetConvexHull()
RUN GetExtent()
RUN CollectGeometries()
RUN UpsertElements()
```

`RUN` can also be used for calling custom operations.

```
RUN AddElementsFromGeoMesa(PointOfInterest, catalog, poi, "-180,-90,180,90")
```

### SELECT

`SELECT` is a basic operation, analagous to a SQL `SELECT`.  It simply filters all the elements in the graph by a group and optionaly other filters.  You can select an entity by it's id or select all the entities in a list of groups.  If selecting by group, then use a comma-separated list with each group prepended by a `$`.  You can optionally, also save the output to a secondary set by id.

**Specification**

```
SELECT [ <entity id> | $<group>[,$<group> ...] ] [UPDATE <id of secondary set> [WITH $<group>[,$<group> ...] ]
```

**Examples**
 
 ```
 SELECT $DATASET
 SELECT $PointOfIntereset,$PointOfInterestType RUN GetElementsWithinSet() OUTPUT elements
 ```

## Filters

Filters can be used in `SELECT` and `NAV` operations for filtering a set of elements (primary or secondary)

```
dwithin(geom_wkt, "Logan Circle, DC", 1000, meters)
icontains(attributes, name, "El Centro")
collectioncontains(aliases, "bar")
```

# Compiler Steps

SGOL-compilers can take a few basic steps to pre-process operation chains to decrease burden on users and client applications, including **initializing secondary sets**, **injecting discards**, and **appending output**.

**Initializing Secondary Sets**

Compilers can scan for references to **secondary** sets (e.g., `... FETCH b ...`) and prepend an **INIT** to the beginning of the chain if missing (e.g., `INIT b...`).

**Injecting Discards**

Certain clauses, by semantic definition, never take any input, such as **SELECT** and **FETCH**.  Compilers can inject **DISCARD** clauses into a chain when they can be inferred.  For example:

`SELECT A UPDATE c FETCH b...` becomes `SELECT A UPDATE c DISCARD FETCH b...`

**Appending Output**

If the final clause in a chain is not **OUTPUT**, then a compiler can attempt to infer the final output type.  For example, `SELECT $PointOfInterest` becomes `SELECT $PointOfInterest OUTPUT entities`.  However, the compiler may not be aware of the output type for **RUN** commands, e.g., `RUN XYZ(a, b, c)`.  If there is no final **OUTPUT**, then the response is up to the SGOL-comptaible service.

# Examples

These are some full real-world examples.

**Selecting Datasets**

Select all datasets where an alias for the dataset is "bank" and return as a geojson.

```
SELECT $Dataset FILTER collectioncontains(aliases, "bank") OUTPUT geojson
```

**Convex Polygon**

Find all medical facilities and then wrap a convex polygon around the set and export as geojson.

```
SELECT $PointOfInterest FILTER collectioncontains(keywords, medical) RUN GetConvexPolygon(geom_wkt, geojson) OUTPUT geojson
```

**Finding Points Of Interest**

```
NAV $PointOfInterest $HasType pointofinteresttype_cafe UPDATE collection WITH $PointOfInterest FILTER dwithin(geom_wkt, "Logan Circle, DC", 1000, meters)
```
