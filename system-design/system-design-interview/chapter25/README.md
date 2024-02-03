# S3-like Object Storage
In this chapter, we'll be designing an object storage service, similar to Amazon Simple Storage Service(S3).

Storage systems fall into three broad categories:
 * Block storage
 * File storage
 * Object storage

Block storage are devices, which came out in 1960s. HDDs and SSDs are such examples.
These devices are typically physically attached to a server, although they can also be network-attached via high-speed network protocols.Block storage presents raw blocks to the server as a volume. Most flexible and versatile form of storage.
Servers can format the raw blocks and use them as a file system or it can hand over control of them to applications directly.

File storage is built on top of block storage. It provides a higher level of abstraction, making it easier to manage folders and files (hierarchical directory structure).

Object storage sacrifices performance for high durability, vast scale and low cost.
It targets "cold" data and is mainly used for archival and backup.
There is no hierarchical directory structure, all data is stored as objects in a flat structure.
It is relatively slow compared to other storage types. Most cloud providers have an object storage offering - Amazon S3, Google GCS, etc.
![storage-comparison](images/storage-comparison.png)

|                 | Block Storage                    | File Storage                            | Object Storage                 |
|-----------------|----------------------------------|-----------------------------------------|--------------------------------|
| Mutable Content | Y                                | Y                                       | N (has object versioning, in place update not supported）     |
| Cost            | High                             | Medium to high                          | Low                            |
| Performance     | Medium to high, very high        | Medium to high                          | Low to medium                  |
| Consistency     | Strong consistency               | Strong consistency                      | Strong consistency [5]         |
| Data access     | SAS/iSCSI/FC                     | Standard file access, CIFS/SMB, and NFS | RESTful API                    |
| Scalability     | Medium scalability               | High scalability                        | Vast scalability               |
| Good for        | Virtual machines (VM), high performance applications like databases | General-purpose file system access      | Binary data, unstructured data |

Some terminology, related to object storage:
 * Bucket - logical container for objects. Name is globally unique. To upload an object, we must first create a bucket. Multiple objects can be stored in a bucket.
 * Object - An individual piece of data, stored in a bucket. Contains object data (any sequence of bytes) and metadata (set of name-value pairs that describe the object).
 * Versioning - A feature keeping multiple variants of an object in the same bucket.Enabled at bucket level and makes it easier to recover objects that are deleted or overwritten by accident.
 * Uniform Resource Identifier (URI) - each resource (buckets and objects) is uniquely identified by a URI.
 * Service-level Agreement (SLA) - contract between service provider and client. 

Amazon S3 Standard-Infrequent Access storage class SLAs:
 * Durability of 99.999999999% across multiple Availability Zones
 * Data is resilient in the event of entire Availability Zone being destroyed
 * Designed for 99.9% availability

# Step 1 - Understand the Problem and Establish Design Scope
 * C: Which features should be included?
 * I: Bucket creation, Object upload/download, versioning, Listing objects in a bucket
 * C: What is the typical data size?
 * I: We need to store both massive objects and small objects efficiently
 * C: How much data do we store in a year?
 * I: 100 petabytes
 * C: Can we assume 6 nines of data durbility (99.9999%) and service availability of 4 nines (99.99%)?
 * I: Yes, sounds reasonable

## Non-functional requirements
 * 100 PB of data
 * 6 nines of data durability
 * 4 nines of service availability
 * Storage efficiency. Reduce storage cost while maintaining high reliability and performance

## Back-of-the-envelope estimation
Object storage is likely to have bottlenecks in disk capacity or IO per second (IOPS).

Assumptions:
 * we have 20% small (less than 1mb), 60% mid-size (1-64mb) and 20% large objects (greater than 64mb),
 * One hard disk (SATA, 7200rpm) is capable of doing 100-150 random seeks per second (100-150 IOPS)

Given the assumptions, we can estimate the total number of objects the system can persist.
 * Let's use median size per object type to simplify calculation - 0.5mb for small, 32mb for medium, 200mb for large.
 * Given 100PB of storage (10^11 MB) and 40% of storage usage results in 0.68bil objects (10^11 x 0.4/ (0.2 x 0.5 MB + 0.6 x 32 MB + 0.2 x 200 MB) = 0.68 Bil)
 * If we assume metadata is 1kb, then we need 0.68tb space to store metadata info

# Step 2 - Propose High-Level Design and Get Buy-In
Let's explore some interesting properties of object storage before diving into the design:
 * Object immutability - objects in object storage are immutable (not the case in other storage systems). We may delete them or replace them, but no in-place update.
 * Key-value store - an object URI is its key and we can get its contents by making an HTTP call
 * Write once, read many times - data access pattern is writing once and reading many times. According to some Linkedin research, 95% of operations are reads
 * Support both small and large objects

Design philosophy of object storage is similar to UNIX - when we save a file, it creates the filename in a data structure, called inode and file data is stored in different disk locations.
The inode contains a list of file block pointers, which point to different locations on disk. 

When accessing a file, we first fetch its metadata from the inode, and then read the file data by following the file block pointers to the actual disk locations.

Object storage works similarly - metadata store is used for file information, but contents are stored on disk/object store:
![object-store-vs-unix](images/object-store-vs-unix.png)

By separating metadata from file contents, we can scale the different stores independently. Data store contains immutable data, metadata store contains mutable data.:
![bucket-and-object](images/bucket-and-object.png)

## High-level design
![high-level-design](images/high-level-design.png)
 * Load balancer - distributes API requests across service replicas
 * API service - Stateless server, orchestrating calls to metadata and object store, as well as IAM service.
 * Identity and access management (IAM) - central place for authentication (who you are), authorization (what operations you can do based on who you are), access control.
 * Data store - stores and retrieves actual data. Operations are based on object ID (UUID).
 * Metadata store - stores object metadata
Metadata and data stores are logical components and can be implemented in different ways (eg:Ceph's Rados Gateway uses multiple Rados objects).

## Uploading an object
![uploading-object](images/uploading-object.png)
 * Create a bucket named "bucket-to-share" via HTTP PUT request
 * API service calls IAM to ensure user is authorized and has write permissions
 * API service calls metadata store to create a bucket entry. Once created, success response is returned.
 * After bucket is created, HTTP PUT is sent to create an object named "script.txt"
 * API service verifies user identity and ensures user has write permissions
 * Once validation passes, object payload is sent via HTTP PUT to the data store. Data store persists it and returns a UUID.
 * API service calls metadata store to create a new entry with object_id, bucket_id and bucket_name, among other metadata.

| object_name                   | object_id                            | bucket_id                 |
|-------------------------------|--------------------------------------|---------------------------|
| script.txt                    | 2345df78a                            | 43785DA8345|

Example object upload request:
```
PUT /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 17:51:00 GMT
Authorization: authorization string
Content-Type: text/plain
Content-Length: 4567
x-amz-meta-author: Alex

[4567 bytes of object data]
```

## Downloading an object
Buckets have no directory hierarchy, buy we can create a logical hierarchy by concatenating bucket name and object name to simulate a folder structure.

Example GET request for fetching an object:
```
GET /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 18:30:01 GMT
Authorization: authorization string
```

![download-object](images/download-object.png)
 * Client sends an HTTP GET request to the load balancer, ie `GET /bucket-to-share/script.txt`
 * API service queries IAM to verify the user has correct permissions to read the bucket
 * Once validated, UUID of object is retrieved from metadata store
 * Object payload is retrieved from data store based on UUID and returned to the client

// sprint 1
# Step 3 - Design Deep Dive
## Data store
Here's how the API service interacts with the data store:
![data-store-interactions](images/data-store-interactions.png)

The data store's main components:
![data-store-main-components](images/data-store-main-components.png)

The data routing service provides a RESTful or gRPC API to access the data node cluster.
It is a stateless service, which scales by adding more servers.

It's main responsibilities are:
 * querying the placement service to get the best data node to store data
 * reading data from data nodes and returning it to the API service
 * Writing data to data nodes

The placement service determines which data nodes should store an object.
It maintains a virtual cluster map, which determines the physical topology of a cluster.The virtual cluster map contains location information for each data node which the lacement service users to make sure the replicas are physically separated. This separation is key to high durability.
![virtual-cluster-map](images/virtual-cluster-map.png)

The service also receives consistent heartbeats within configurable 15 second intervals from all data nodes to determine if they are alive or should be removed from the virtual cluster.

Since this is a critical service, it is recommended to maintain a cluster of 5 or 7 replicas, synchronized via Paxos or Raft consensus algorithms. The consensus protocol ensures that as long as more than half of the nodes are healthy, the service as a whole continues to work. 
Eg a 7 node cluster can tolerate 3 nodes failing.

Data nodes store the actual object data.
Reliability and durability is ensured by replicating data to multiple data nodes, also called the replication group.

Each data node has a data service daemon running, which sends heartbeats to the placement service.

The heartbeat includes:
 * How many disk drives (HDD or SSD) does the data node manage?
 * How much data is stored on each drive?
When the placement service receives the heartbeat for the first time, it assigns an ID for this data node, adds to the virtual cluster map and returns the unique ID for the data node, virtual cluster map and where to replicate data.

### Data persistence flow
![data-persistence-flow](images/data-persistence-flow.png)
 * API service forwards the object data to data store
 * Data routing service creates UUID for data object, queries placement services for data node (which checks virtual cluster map and returns the info) and sends the data to the primary data node with UUID.
 * Primary data node saves the data locally and replicates it to two secondary data nodes. Response is sent after successful replication on all secondary nodes.
 * The UUID of the object is returned to the API service.

Caveats:
 * Given an object UUID, it's replication group is deterministically chosen by using consistent hashing
 * In step 4, the primary data node replicates the object data to all secondary nodes before returning a response. This favors strong consistency over higher latency.
![consistency-vs-latency](images/consistency-vs-latency.png)
Both 2 and 3 are forms of eventual consistency.

### How data is organized
One simple approach to managing data is to store each object in a separate file. 

This works, but is not performant with many small files in a file system:
 * Data blocks on HDD are wasted, because every file uses the whole block size. Typical block size is 4kb.
 * Many files means many inodes. Operating systems don't deal well with too many inodes and there is also a max inode limit.

These issues can be addressed by merging many small files into bigger ones via a write-ahead log (WAL). Once the file reaches its capacity (typically a few GB), a new file is created:
![wal-optimization](images/wal-optimization.png)

The downside of this approach is that write access to the file needs to be serialized. Multiple cores accessing the same file must wait for each other.
To fix this, we can confine files to specific cores to avoid lock contention.

### Object lookup
To support storing multiple objects in the same file, we need to maintain a table, which tells the data node:
 * `object_id`
 * `filename` where object is stored
 * `file_offset` where object starts
 * `object_size`

We can deploy this table in a file-based db like RocksDB or a traditional relational database using a B+ tree based storage engine.
Since the access pattern is low write+high read, a relational database works better.

How should we deploy it?
We could deploy the db and scale it separately in a cluster, accessed by all data nodes.

Downsides:
 * we'd need to aggressively scale the cluster to serve all requests
 * there's additional network latency between data node and db cluster

An alternative is to take advantage of the fact that data nodes are only interested to data related to them, 
so we can deploy the relational db within the data node itself. 

SQLite is a good option as it's a lightweight file-based relational database.

### Updated data persistence flow
![updated-data-persistence-flow](images/updated-data-persistence-flow.png)
 * API Service sends a request to save a new object
 * Data node service appends the new object at the end of a file, named "/data/c"
 * A new record for the object is inserted into the object mapping table

### Durability
Data durability is an important requirement in our design. In order to achieve 6 nines of durability, every failure case needs to be properly examined.

First problem to address is hardware failures. We can achieve that by replicating data nodes to multiple hard disks to minimize probability of failure.
But in addition to that, we also ought to replicate across different failure domains (cross-rack, cross-dc, separate networks, etc). A failure domain is a physical or logical section of the environment that is negatively affected when a critical service experiences problems.
A critical event can cause multiple hardware failures within the same domain and data can be replicated in different Availability Zones (AZ):
![failure-domain-isolation](images/failure-domain-isolation.png)

The choice of failure domain level doesn't directly increase the durability of data, but it will result in better reliability in extreme cases, such as large-scale power outages, colling system failures, natural disasters, etc. 

Assuming annual failure rate of a typical HDD is 0.81%, making three copies gives us 6 nines of durability.

Replicating the data nodes like that grants us the durability we want, but we could also leverage erasure coding to reduce storage costs.

Erasure coding enables us to use parity bits, which allow us to reconstruct lost bits in the event of a failure:
![erasure-coding](images/erasure-coding.png)

Imagine those bits are data nodes. If two of them go down, they can be recovered using the remaining four ones.

There are different erasure coding schemes. In our case, we could use 8+4 erasure coding, split across different failure domains to maximize reliability:
![erasure-coding-across-failure-domains](images/erasure-coding-across-failure-domains.png)

An (8+4) erasure coding setup breaks up the original data evenly into 8 chunks and calculate 4 paities. All 12 pieces of data have the same size. All 12 chunks of data are distributed across 12 different failure domains. Compared to replciation whree the data router only needs to read data for an object from one healthy node, in erasure coding the data router has to read data from at least 8 healthy nodes. This is an architectural design tradeoff. We use a more complex solution with a slower access speed for higher durability and lower storage cost.

Erasure coding enables us to achieve a much lower storage cost (50% more) at the expense of access speed due to the data routing service having to collect data from multiple locations:
![erasure-coding-vs-replication](images/erasure-coding-vs-replication.png)

Other caveats:
 * Replication requires 200% storage overhead (in case of 3 replicas) vs. 50% via erasure coding
 * Erasure coding [gives us 11 nines of durability](https://github.com/Backblaze/erasure-coding-durability) vs 6 nines via replication
 * Erasure coding requires more computation to calculate and store parities
 * For write performance, replicating data does not need computation while erasure coding has increased write latency because we need to calculate parities before data is written to disk.
 * For read performance, for replication, in normal operation, reads are served from the same replica. Reads under a failure mode are not impacted because reads can be served from a non-fault replica. For erasure coding, in normal operation, every read has to come from multiple nodes in the cluster. Reads under a failure mode are slower because the missing data must be reconstructed first. 

In summary, replication is more useful for latency-sensitive applications, whereas erasure coding is attractive for storage cost efficiency and durability.
Erasure coding is also much harder to implement as it greatly complicates the data node design.

### Correctness verification
If a disk fails entirely, then the failure is easy to detect. This is less straightforward in case a part of the disk memory gets corrupted.

To detect this, we can use checksums - a hash of the file contents, which can be used to verify the file's integrity.
If we know the checksum of the original data, we can compute the checksum of the data after transmission. If they are dfiferent, data is corrupted. If they are same, very high chance of data not being corrupted.Some checksum algorithms are MD5, SHA1, HMAC, etc. A good checksum algorithm usually otuputs a significantly different value even for a small change made to the input. 
In our case, we'll store checksums for each file and each object:
![checksums-for-correctness](images/checksums-for-correctness.png)

In the case of erasure coding (8+4), we'll need to fetch each of the 8 pieces of data separately and verify each of their checksums.

## Metadata data model
Table schemas:
![metadata-data-model](images/metadata-data-model.png)

Queries we need to support:
 * Find an object ID by name
 * Insert/delete object based on name
 * List objects in a bucket sharing the same prefix

There is usually a limit on the number of buckets a user can create, hence, the size of the buckets table is small and can fit into a single db server.
But we still need to scale the server for read throughput.

The object table will probably not fit into a single database server, though. Hence, we can scale the table via sharding:
 * Sharding by bucket_id will lead to hotspot issues as a bucket can have billions of objects
 * Sharding by object_id makes the load more evenly distributed, but our queries will be slow
 * We choose sharding by `hash(bucket_name, object_name)` since most queries are based on the object/bucket name.

Even with this sharding scheme, though, listing objects in a bucket will be slow.

## Listing objects in a bucket
In a single database, listing an object based on its prefix (looks like a directory) works like this:
```
SELECT * FROM object WHERE bucket_id = "123" AND object_name LIKE `abc/%`
```

This is challenging to fulfill when the database is sharded. To achieve it, we can run the query on every shard and aggregate the results in-memory.
This makes pagination challenging though, since different shards contain a different result size and we need to maintain separate limit/offset for each.

We can leverage the fact that typically object stores are not optimized for listing objects, so we can sacrifice listing performance.
We can also create a denormalized table for listing objects, sharded by bucket ID. 
That would make our listing query sufficiently fast as it's isolated to a single database instance.

## Object versioning
Versioning keeps multiple versions of the object in the bucket and is used to restore objects that are accidentally deleted or overwritten. works by having another `object_version` column which is of type TIMEUUID, enabling us to sort records based on it. See phone for diagram.

Each new version produces a new `object_id`:
![object-versioning](images/object-versioning.png)

Deleting an object creates a new version with a special `object_id` indicating that the object was deleted. Queries for it return 404 Object Not Found error:
![deleting-versioned-object](images/deleting-versioned-object.png)

## Optimizing uploads of large files
Uploading large files can be optimized by using multipart uploads - splitting a big file into several chunks, uploaded independently:
![multipart-upload](images/multipart-upload.png)
 * Client calls service to initiate a multipart upload
 * Data store returns an uploadID which uniquely identifies the upload
 * Client splits the large file into several chunks, uploaded independently using the upload id
 * When a chunk is uploaded, the data store returns an etag, which is a md5 checksum, identifying that upload chunk
 * After all parts are uploaded, client sends a complete multipart upload request, which includes uploadID, part numbers and all etags
 * Data store reassembles the object from its parts. The process can take a few minutes. After that, success response is returned to the client.

Old parts, which are no longer useful can be removed at this point. We can introduce a garbage collector to deal with it.

## Garbage collection
Garbage collection is the process of reclaiming storage space, which is no longer used. There are a few ways data becomes garbage:
 * lazy object deletion - object is marked as deleted without actually getting deleted
 * orphan data - eg an upload failed mid-flight and old parts need to be deleted
 * corrupted data - data which failed checksum verification

The garbage collector does not remove objects from data store right away. Deleted objects will be periodically cleaned up with a compaction mechanism. The garbage collector is also responsible for reclaiming unused space in replicas. 
With replication, data is deleted from both primaries and replicas. With erasure coding (8+4), data is deleted from all 12 nodes.

To facilitate the deletion, we'll use a process called compaction:
 * Garbage collector copies objects which are not deleted from "data/b" to "data/d"
 * `object_mapping` table is updated once copying is complete using a database transaction
 * To avoid making too many small files, compaction is done on files which grow beyond a certain threshold
![compaction](images/compaction.png)

# Step 4 - Wrap Up
Things we covered:
 * Designing an S3-like object storage
 * Comparing differences between object, block and file storages
 * Covered uploading, downloading, listing, versioning of objects in a bucket
 * Deep dived in the design - data store and metadata store, replication and erasure coding, multipart uploads, sharding
