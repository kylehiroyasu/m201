# Chapter 3: Index Operations

## Query Plans



### Tradeoffs between foreground and background indexes

* Foreground indexes

 * are fast but block all incoming operations on the collection until it's complete
 * are only acceptable to use on production environments during a maintenance window
 
* Background indexes

 * Are nonblocking but they are slower to complete than foreground indexes
 * The speed to build an Index in the background will depend on the load of read/write operations happening in the foreground
 * Should be used as the deferred way of adding indexes to production machines if you want to avoid maintenance windows
 
Regardless of the type of index being built, it will impact the read/write perfomance as the index building process consumes resources.

Use the following command to import the new `restaurants` dataset:

```javascript
mongoimport -d m201 -c restaurants --drop restaurants.json
```

### How to use Background Indexes

To create an index in the background we can use the following:

```javascript
db.restaurants.createIndex({"cuisine":1, "name":1, "address.zipcode":1}, {"background": true})
```

By default `"background"` is set to `false`

### Using `currentOp()` and `killOp()`

To check how a process is going/if an index is finished building you can use the following:

```javascript
db.currentOp()
```

Or to specifically search for index building operations use the following:

```javascript
// from another shell, switch to the m201 database, and find all index-build
// operations
use m201
db.currentOp(
  {
    $or: [
      { op: "command", "query.createIndexes": { $exists: true } },
      { op: "insert", ns: /\.system\.indexes\b/ }
    ]
  }
)

```
Using these commands you should see a key called `opid` which is the unique identifier for the running process.  If you need to prematurely kill an operation you can use the following command to kill the operation:

```javascript
db.killOp([the_op_id])
```
more information about index building is available [here](https://docs.mongodb.com/manual/core/index-creation/?jmp=university&_ga=2.247858895.99457314.1507576023-692745340.1506849340)

## Query Plans

### What are query plans

* A query plan is a series of stages that feed into eachother
* There can be many different query plans generated trying to utilize different indexes


### How the query optimizer works with them

The query optimizer tries to find the 'best' possible query plan to return data

1. The query optimizer examines all available query plans to use
2. Identifies 'candidate indexes' which are viable for the given query
3. The query optimizer then generates 'candidate plans' which can be tested
4. Mongo has an 'empirical query planner' where each of the query plans is executed for a brief period of time
5. Finally the 'best' performing plan over this trial period is elected as the winning query plan

The definition of 'best' is not clearly defined, it could be things like all the documents returned fastest, or some portion of documents returned in a sort order, or other metrics. 

### How they're cached

Winning queries plans get cached and associated with queries of a given 'shape'. In this way similar queries to not have to be optimized every time they are ran.

Over time the queries used and the indexes on collections will change thus the cache may be flushed under any of the following reasons:

* Server restart
* When the work done is 10x the work performed by the winning plan (a threshold to indicate a bad query plan)
* Indexes are rebuilt
* Indexes are created/dropped

more info on query plans [here](https://docs.mongodb.com/manual/core/query-plans/?jmp=university&_ga=2.251416526.99457314.1507576023-692745340.1506849340)

## Understanding Explain

### What information is provided by `explain()`

* What index is being used for the query plan
* Is the query using an index to do the sorting
* is your query using an index to provide the projection (covered query)
* How selective is your index
* Which part of your query plan is the most expensive


### How `explain()` works

To start using explain you can create an "explainable object":

```javascript
exp = db.people.explain() // explainable object

exp.find({"address.city":"Lake Meaganton"})
```
This will given an explain output of what would happen if you ran this query. That is to say: the query will not be ran!

However if we run the following:

```javascript
exp = db.people.explain("executionStats")

exp.find({"address.city":"Lake Meaganton"})
```
This will execute the query and give us back perfomance stas

The last example:

```javascript
// and one final explainable object with the 'allPlansExecution' parameter
expRunVerbose = db.people.explain("allPlansExecution")
```

Will show also what other "rejected" execution plans were considered. This will execute the given query.

### What the output looks like in a sharded cluster

On a sharded cluster the explain output is similar to a single database, however there will be a query plan for each individual shard instead of a single winning query.

More information about running a sharded environment can be found in the handout.

More information about understanding explain output can be found in this video [here](https://www.mongodb.com/presentations/deciphering-explain-output?_ga=2.21849075.99457314.1507576023-692745340.1506849340)

### `explain()` in MongoDB Compass

Just go try a query in Compass, will resemble the explain output we've seen in it's raw json format

## Forcing Indexes with `hint()`

Can use `hint()` to override MongoDBs default index selection.

Even if we've designed the perfect index, it is possible MongoDBs query optimization may not use our perfect index for one reason or another.

The usage of `hint()` is as follows:

```javascript
db.people.find(
		{ name:"John Doe", zipcode: { $gt: "6300"} }
	).hint(
		{name:1, zipcode:1}
	)

db.people.find(
		{ name:"John Doe", zipcode: { $gt: "6300"} }
	).hint(
		"name_1_zipcode_1"
	)
```

By providing the shape or name of the index we want mongo to use it will use our desired index.

In general use caution when using the `hint()` method. If you find yourself needing to use this it may be an indication you have to many indexes and you would likely benefit from doing some cleanup.

## Resource Allocation for Indexes

Using indexes helps better utilize our resources by:

* optimizing out queries
* reducing our response time of queries

However Indexes are data structures which also require available resources to usable. These include considerations of:

* Index Size
* Resource allocation to hold index in memory
* Edge cases

### Index Size

This can be easily checked using compass or using `db.stats()`

### Resource Allocation

We need the following:

* Disk space - we will assume that we have plenty of disk space for indexes

 * however keep in mind if you're storing indexes and data on separate physical drive you should ensure you have adequate disk space
 
* RAM

 * Our deployments should be sized such that we can fit all of our indexes into memory
 * If we don't have adequate RAM we'll have to use a lot of disk IO to travers/read the entire index file
 
We can check on how much RAM we're utilizing by running something like the following:

```javascript
db.rides_other.stats({indexDetails:true})
```
There will be information regarding every index. You can find details about an index you've recently created including `cache` information like

* "bytes currently in the cache"
* "bytes read into cache"
* "pages written from cache"
* "pages read into cache"

## Basic Benchmarking


