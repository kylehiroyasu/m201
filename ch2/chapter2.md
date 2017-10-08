# Chapter 2 : MongoDB Indexes

## What are Indexes
Indexes are used to improve the speed of slow queries like an index/glossary in a book

Without an index we would have to search through every document to complete queries AKA `collscan`
* this kind of query is O(n) or linear in performance

Indexes can be created using one or more fields as a key to allow our query to identify the correct values quickly, think of something like a telephone book and searching for people based on first and last name.

* Multiple indexes can be created on a single collection to support various queries
* `_id` is automatically indexed in all collections
* Indexes are stored as `B-trees`
* Although Indexes speed up read performance they come at the cost of write perfomance.

More information is found in the [Indexes Section](https://docs.mongodb.com/manual/indexes/?jmp=university&_ga=2.58433578.1481678792.1507233934-692745340.1506849340)

## How Data is Stored on Disk

There will be differences in how data is stored depending on the selected storage engine (MMAPv1, Wired Tiger, other...)

By default `mongod` will store all data related to its databases in `/data/db`. This includes collection and index information.

This can be tested running the following:
```bash
# start a mongod
mongod --dbpath /data/db --fork --logpath /data/db/mongodb.log

# immediately shut down the server
mongo admin --eval 'db.shutdownServer()'
```

You can also configure this differently to run store each database in its own folder
```bash
# this time, start the server with the --directoryperdb option
mongod --dbpath /data/db --fork --logpath /data/db/mongodb.log --directoryperdb

# write a single document into the 'hello' database
mongo hello --eval 'db.a.insert({a:1}, {writeConcern: {w:1, j:true}})' 

# then, shutdown the server
mongo admin --eval 'db.shutdownServer()'
```

It's also possible to specify Wired Tiger to store indexes in their own folder as shown here:
```bash
# this time, start the server with the --directoryperdb and
# --wiredTigerDirectoryForIndexes options
mongod --dbpath /data/db --fork --logpath /data/db/mongodb.log \
       --directoryperdb --wiredTigerDirectoryForIndexes

# write a single document into the 'hello' database
mongo hello --eval 'db.a.insert({a:1}, {writeConcern: {w:1, j:true}})' 

# then, shutdown the server
mongo admin --eval 'db.shutdownServer()'
```

This means we could link our indexes to this directory and store our indexes on a separate disk. In terms of performance this means:
* IO for indexes can be independent of normal DB IO


MongoDB also supports document compression which implies less IO which implies faster queries as a tradeoff for more CPU cycles.

How data makes it to disk:
* Application/users can specify a write concern which will force a sync
* Periodic checkpoint/sync periods
* All writes are saved in a compressed journaled format which are processed in batches
 * if for whatever reason a db has a fault these journals will be used to recover to an atomic state
 * this can also be specified in application write concern

## Single Field Indexes

### Key Features
* Keys from only one field
* can find a single value for the indexed field
* can find a range of values
* can use dot notation to index fields in subdocuments
* can be used to find several distinct values in a single query

Using the `people` dataset we can load the data
```bash
# import the people dataset
mongoimport -d m201 -c people --drop people.json

# connect to the m201 database
mongo m201
```
and then create some indexes

```javascript
db.people.find({ssn:"720-38-5636"}) // compare query with no index
db.people.createIndex( { ssn : 1 } )

exp = db.people.explain("executionStats")
exp.find({ssn:"720-38-5636"}) // query with an index

```
Keep in mind the following:
* indexes can be only used to query on the fields they contain
* it is a bad idea to index on a key which points to an object, instead index on the values stored inside the object as shown below

```javascript
// create an index using dot-notation
db.examples.createIndex( { "subdoc.indexedField" : 1 } )
```

## Sorting with Indexes

Sorting can be accomplished one of two ways:
* In memory: the servers ram will have to sort the everytime the query is called and will abort if it's too big (>64Mb)
* Using an index: if your query uses an index then the results will be ordered by fields the index was built with

The query planner can choose an index which fits either the predicate or the sort best. Keep in mind with a single field index we can traverse the index either forwards or backwards


## Quering on Compound Indexes

### Structure
* B-trees are stored in a flat file, thus your indexes, even multi-indexes are flat 
* The order of the keys is very important to the usefulness of the index

### Index Prefixes
Index prefixes are the beginning subsets of the indexed field. For example with the following index:
```javascript
{ "item": 1, "location": 1, "stock": 1 }
```
The index has exactly two prefixes:
```javascript
{ item: 1 }
{ item: 1, location: 1 }
```

These would not be considered index prefixes:
```javascript
{ location: 1 }
{ item: 1, stock: 1 }
```
These are useful because prefixes can be used exactly the same way the full indexed would be used

You can read more about [compound indexes](https://docs.mongodb.com/manual/core/index-compound/?jmp=university&_ga=2.258679690.1481678792.1507233934-692745340.1506849340) and [creating indexes](https://docs.mongodb.com/manual/tutorial/create-indexes-to-support-queries/?jmp=university&_ga=2.258679690.1481678792.1507233934-692745340.1506849340)

## When you can use sort with Indexes

To simply sort documents in a collection, an index can be used under the same conditions as a query. That isas long as the sort uses either an index or an index prefix then Mongo can utilize the index.

To use an index to query and sort the query must use equality conditions on all of the prefix keys that precede the sort keys.

## Multikey Indexes
Creating an index on an array or object

For example using this document:
```javascript
{
	productName: "MongoDB Short Sleeve T-Shirt",
	categories: ["T-Shirts", "Clothing", "Apparel"],
	stock: [
	       { size: "L", color: "green", quantity: 100 },
	       { size: "L", color: "blue", quantity: 10 }
	]
}
```
We could create an index on `categories` or on `stock.quantity`, or in addition we could also `productName`
However we should avoid creating an index on `categories` and on `stock.quantity`, and we should also keep in mind how large the arrays might grow as that will impact the overall size and if it fits in memory.

Multikey Indexes don't support covered queries

## Partial Indexes

When you only want to create an index on a subset of documents in a collection.

Use cases:
* Perhaps you have a collection of restaurants but 90% of queries only look at restaurants with more than 4 stars
* If the full size of an index is too big to fit into memory
* When you want to use multikey indexes with large arrays

Note sparse indexes are a special case of partial indexes.

* In order to make sure you use your partial index the query has to guarantee to return a subset of the documents defined by the filter expression used to create the partial index

### Restrictions

* you cannot specify both partialFilterExpression and the sparse option
* _id indexes cannot be partial indexes
* Shard key indexes cannot be partial indexes

Read more about partial indexes [here](https://docs.mongodb.com/manual/core/index-partial/?jmp=university&_ga=2.32092094.1481678792.1507233934-692745340.1506849340)

## Text Indexes

Rather than using an exact string match or using a slow REGEX expression we can also generate a text index

### How to create indexes
```javascript
// create a text index on "statement"
db.textExample.createIndex({ statement: "text" })

// Search for the phrase "MongoDB best"
db.textExample.find({ $text: { $search: "MongoDB best" } })
```

### Implications on number of index keys
This text search feature works very similarly to how a multikey index works, splitting text into basically an array of individual, case insensitive words.

### Costs associated with text searches
Keep in mind the longer the piece of text:
* more keys to query
* increased index size
* takes longer to build index
* decreased write performance

To reduce the number of index keys examined you can also make a compound index with a scalar value.

Also keep in mind text searches basically or the words you are searching for. So the following example:
```javascript
// Display each document with it's "textScore"
db.textExample.find({ $text: { $search : "MongoDB best" } }, { score: { $meta: "textScore" } })
```

could show any documents whose statements include the word "MongoDB" or "best" or both, the "textScore" will give an indication of how well the document matches our search.

## Collations

Define the string ordering. They can be defined on the following levels:
* collection
* index
 * to utilize an index with a different collation than the underlying collection, you must specify the matching collection in the query

Why use collations:
* Correctly sort/search text based on a given locale
* Marginal performance impact
* Case insensitive indexes

More on collations [here](https://docs.mongodb.com/manual/reference/collation/?jmp=university&_ga=2.27373372.1481678792.1507233934-692745340.1506849340) 
