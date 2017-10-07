# Chapter 5: Performance on Clusters

## Perfomance Consideration

Mongo is designed to operate as a distributed system using:
* Replica Clusters - high availability
* Shard Clusters - scalability

### Things to consider
* Consider latency
* data is distributed across different nodes
* read implications
* write implications

### Replica Sets
* key to guarantee availability even when there is a node failure
* offload data to secondaries
* privilege your primary for operational workload
* have indexes/workloads which are targeted specifically on secondary loads

### Sharding
* horizontal scalability

consists of
* mongos
  * routes all of the incoming application queries
* config servers
 * communicate between mongos about shard configs
* shard nodes
  * do the brunt of the computational work
  * they are in themselves replica sets
    * if they weren't configured as a replica we could lose app data if it goes down

##### Before Sharding
* sharding is a horizontal scaling solution
* should ask yourself if you have reached a vertical scaling limit - cost/performance gains
* you need to understand how your data grows and how your data is accessed
* sharding works by defining key based ranges - our shard key - you must be able to identify a good one

#### Latency over Sharded cluster 
* Consider that for mongos, config servers, and shard nodes may need to communicate with eachother
* Co-locating application and mongos servers will help reduce latency
* Overall latency, if configured correctly should be negligible

#### Types of reads
* Scattered gathered: ping all nodes of shard cluster
* routed queries: we aska asmall numer of shard nodes to send us the data

these two have different performance profiles

* scattered gather pays a price of communicating with all of the shard nodes to gather data
* routed queries will only ask a limited number of the shard nodes

Which type of read is executed depends on if you are using the shard key which allows mongo to determine
which shard it needs to send the query to.

Sorting, limiting, and skipping data queries are more complicated in a sharded cluster because the data is split/sharded across mutliple machines.


## Increasing Write Peformance with Sharding

### Vertical vs. Horizontal Sharding

#### Vertical
Idea is you buy a bigger machine, more memory/ram/etc
At a certain point the return on performance to price will decrease
#### Horizontal
Idea is you just add more machines
Advantage is that you can use commodity hardware and costs tend to scale linearly

### Shard Key Rules
* All reads and writes go through a `mongos` in a sharded cluster
* a shard key defines how to partition data across shards
* A shard key is either an `id` or compound index which is included in every document in the collection
* data should be evenly distributed between shards
* Shard keys are broken into chunks ~64Mb chunks and distributed across each of the shards

#### Shard Key Considerations
* Cardinality: the key should have many distinct values
 * low cardinality keys will make it difficult or impossible to split the shard key into chunks
* Frequency: the values which define the shard key should come in an uniform frequency
 * similar to low cardinality if a particular value is disproportionately present it's chunk may become overgrown
* Rate of change: avoid monotonic values such ObjectId or date/timestamps as the first value of our shard key
 * all the newest data will end up being set to one shard

### Bulk Writes
* Ordered: wait for the current write to finish before writing next document
 * this type of writing does not take advantage of sharded environment because it will only write one at a time
 * additionally there may be noticeable network latency as the shards communicate with mongos on every write
* Unordered: write all in parallel even if there are some failures
 * mongos will have to serialize the data to it's respective shards but then the data can be written to it's corresponding shard in parallel with the other shards

Learn more about bulk writes [here](https://docs.mongodb.com/manual/core/distributed-write-operations/?jmp=university&_ga=2.65055918.1481678792.1507233934-692745340.1506849340)


## Reading from Secondaries
By deafault mongo always reads from primaries
```
db.people.find().readPref("primary")
```
However this can also be changed to read from the secondary like so
```
db.people.find().readPref("secondary") //read from secondary
db.people.find().readPref("secondaryPreferred") // read from secondary if available, otherwise primary
db.people.find().readPref("nearest") // read from machine with lowest network latency
```
Consider that secondaries may be holding stale data

### When it's a good idea
* stale data is acceptable
* offload data where the analytics job will be long running 
* when secondaries may be geographically distant from primary and the application needs low latency resources

### When it's a bad idea
* in general it's a bad idea
* providing extra capacity for reads when writes are overwhelming primaries
* reading from shards - don't know what state the shard is in, could be midwrite or moving/splitting chunks

#### m201 says never read from secondaries on a sharded cluster

## Replica Sets with Differing Indexes

### Disclaimer
* generally not very useful
* specific analytics secondary nodes
* reporting on delayed consistency data
* text search

### Secondary Node Considerations
* Should prevent these nodes from becoming a primary doing on of the following
 * priority=0
 * hidden node
 * delayed secondary

If you have decided to go forward with creating special indexes on a secondary node it is important to make sure you only add it to a single node instead of the whole cluster as it will have an impact on write performance.

## Aggregation Pipeline on a Sharded Cluster

### How it works

Aggregation pipelines tend to broken up into several stages such as `$match` and `$group`.
#### Using shard key
In the ideal case we will match based on the same field as our shard key so we do not have to query every node.
The aggregation will then also be completed on the single shard and then sent back.

#### Querying across entire database
Each shard will have to do it's own work and then one of the machines will randomly be chosen to aggregate and resolve the results of the entire database before send the results back to the application.

### Special cases
In the following cases the final results are always resolved on the primary:
* `$out`
* `$facet`
* `$lookup`
* `$graphLookup`

If they are used often, you can offset potential performance impacts by using a more powerful machine as your primary.

### Optimizations

Sometime the query planner will swtich the order, or combine steps in the aggregation pipeline to reduce the amount of data on the wire.
