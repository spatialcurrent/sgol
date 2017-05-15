# Spatial Graph Operations Language (SGOL)

# Description

Spatial Graph Operations Language (aka **SGOL**) is a high-level language for describing graph operations, with a focus on spatial data.

This graph language follows many of the design conventions of the common [SQL](https://en.wikipedia.org/wiki/SQL).  The syntax will be familiar, but the actual reserved words are different.

This language describes a chain of graph operations that are executed as a stream.  A primary set of objects (usually Graph `elements`) are passed from clause to clause.  You can save elements to secondary sets while executing the operations by using `UPDATE` and `FETCH`.  This is somewhat similiar to how [jquery](https://jquery.com/), [gulp](http://gulpjs.com/), and more are designed for `chaining`.

The language [specification](#specification), [compiler steps](#compiler-steps), and some [examples](#examples) are provided below.

# Contributing

[Spatial Current, Inc.](https://spatialcurrent.io) is currently accepting pull requests for this repository.  We'd love to have your contributions!  Please see [Contributing.md](https://github.com/spatialcurrent/sgol/blob/master/CONTRIBUTING.md) for how to get started.

# License 

This work is distributed under the **MIT License**.  See **LICENSE** file.

# Specification

## Core Clauses

The following are the `core` clauses implemented by this language: [ADD](#add), [DISCARD](#discard), [SELECT](#select), [NAV](#nav), [HAS](#has), [FETCH](#fetch), [RUN](#run), and [OUTPUT](#output).

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

### SELECT

`SELECT` is a basic operation, analagous to a SQL `SELECT`.  It simply filters all the elements in the graph by a group and optionaly other filters.

**Examples**
 
 ```
 SELECT $DATASET
 SELECT $ORIGIN
 ```

### NAV

`NAV` is the operation for `navigating` / `transversing` / `hopping` around the graph from entity to entity following certain edges.

```
NAV $PointOfInterest $HASTYPE pointofinteresttype_cafe
```

### HAS

`HAS` is the operation for filtering the `current` set of entities to only those connected to another literal set of entities.  It is frequently used within a chain of statements.

```
... HAS INPUT $HasType pointofinteresttype_cafe OUTPUT entities
```

### FETCH

`FETCH` is used for replacing the primary set of elements in the stream with those from a secondary set.  It is frequently used as a **penultimate** operation after a recursive search.

```
FETCH collection
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

### OUTPUT

`Output` is used to specify how to encode the response to the query.  Be sure to use a output schema compatible with the last operation.

```
OUTPUT [elements | entities | edges | geojson | json | bbox]
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
