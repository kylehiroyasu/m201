# Chapter 1 Hardware and Configurations

## Performance and Hardware

MongoDB relies heavily on RAM:
* Aggregation
* Index Traversing
* Write operations - data is first written to pages in RAM
* Query engine - to store and send data
* Connections

CPU is used for:
* WiredTiger Storage Engine will use as many cores as possible to handle read/write requests
* Concurrency Level (number of copies of the data)
* Aggregation Framework
* Map Reduce

Disks
* Obviously store all of our data
* Disks with higher IOPs will help the overall system perform better
* The recommended disk Architecture is: RAID 10
 * provides redundancy across segments of a drive
 * also allows parallel read and write operations

Mongo features the ability to use many different disks. Networking is also a critical consideration for MongoDB because it scales horizontally (read:Nodes) this becomes more important as the database scales up

Other considerations include a given applications:
* Read concern
* Write concern
* Read Preference


## Lab
As Part of the lab be sure to install MongoDB and import the `people.json` file into `m201.people` then check the number of documents with the following:
```
db.people.count({ "email" : {"$exists": 1} })
```
