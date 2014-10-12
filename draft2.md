# Disambiguating Databases

The topic of data storage is one that doesn't need to be well understood until something goes wrong (data disappears), or something goes really right (too many customers). Because databases can be treated like black boxes with an API, their inner workings are often overlooked. They're often treated as magic things that just take data when offered, and supply it when asked. Since these two operations are the only understood activities of the technology, they are often the only feature points presented when one is comparing different technologies.

But what exactly is an *operation*? Within the realm of databases, this could mean any number of things. Is that operation a transaction? Is it an indexing of data? A retrieval from an index? Does it store the data to a durable medium like a hard disk, or does it beam it by laser towards Alpha Centauri? 

It is this ambiguity that causes havoc in the software industry. Misunderstanding the features and guarantees of a database system can cause, at best, user consternation due to slowness or unavailability. At worst, it could result in fiscal damage, or even jail time due to loss data. 

The scope of the term "database" is vast.  Technically speaking, anything that stores data for later retrieval is a database. Even by that broad definition, there is functionality that is common to most databases.  The scope of this article is to enumerate those features at a high level.  The intent is to provide the reader with a tool-set with which they might evaluate databases on their relative merits. 

Applying this feature driven approach, the reader should see two benefits. Firstly, it will allow them to accurately assess their own needs. Secondly, it will allow them to compare technologies by pairing up like features. When viewed with with this lens, comparative benchmarks will only be valid on databases that are performing equal work and providing the same guarantees.


## On Performance - or - The Hard Disk and the Page Cache

Before we dig into the features of databases, lets first discuss why one wouldn't just take all of the features.  The
short answer is that each feature typically comes with a performance cost, if not a complexity cost. 

Most of the functions performed by a database, as well as the algorithms that implement them, are built to work around the
performance bottleneck that is the hard disk.  If you have a requirement that your data (and metadata) be durable, then you must pay
this penalty one way or another. 

### The Hard Disk

The highest latency operation you will encounter within a data center is a seek to a random location on a hard disk. At
present, 7200 RPM disks have a seek time of about 4ms.  The SATA Bus of a typical server (Ivy Bridge Architecture) has a
 theoretical max bandwidth of 750MB/s.  That seems high, but compared to the Ivy Bridge PCI 3 Bus, which has 40 channels of approximately 1GB/s it is tiny.  Compared to its memory bus, which can do 14.9GB/s per channel (with at least 4 channels), it is minuscule.  [Source: Ivy Bridge Architecture](http://en.wikipedia.org/wiki/LGA_2011)

> *SSDs have brought massive latency and throughput improvements to disks. A seek on an SSD is about 60 times faster than a hard disk. They bring their own challenges, however, one interesting one is that the storage cells within an SSD have a fixed lifetime, that is, they can only handle so many writes to them before they fail.  For this reason, they have specialized firmware that spreads writes around the disk, garbage collects, and does other bookkeeping operations. Because of this, they have less predictable performance characteristics (though they are predictably faster than hard disks) *

Once a location has been found, successive append-to, or read-from operations at that location are significantly cheaper.  This is called a sequential read or write. 
Algorithms regarding data storage and retrieval have been optimized against this fact since magnetic rotating disks were invented.

### The Page Cache

Because of the massive latency, and relatively low throughput of hard drives, one optimization that is found in nearly every operating system is the page cache, or buffer cache.

As its name implies, the purpose of the page cache is to optimize disk access by transparently storing contents of files in memory pages mapped to disk by the operating system's kernel.  The idea is that the same local parts of a disk or a file will be read or written many times in a short period of time.  This is usually true for databases. 

When a read occurs, if the file contents are not in the cache, it will simultaneously load the data into the cache, and return the data. A write will modify the contents of the cache, but not necessarily write to the hard disk itself.  This is to eliminate as many disk accesses as possible. Assuming that writing a record of data takes 5ms, and you have to write 20 different records to disk, performing these operations in the page cache and then flushing to disk would cost you only a single disk access, rather than 20.  The performance saving add up quickly.  

> **CAUTION** : The page cache is a significant source of optimization, but can also be a source of danger. If writes to the page cache are not flushed to disk, and a power, disk or kernel failure occurs, you will lose your data.  Be mindful of this when analyzing database solutions which leverage the page cache exclusively for their durability operations.


## Database Features

There are dozens of classifciations of databases. Each of the hundreds of commercially or freely available database
sytems out there likely fall into several of these classes.  We'll try to skip past the classification and instead
try to provide a framework through which we can evaluate each database by their features.

The five categories of features we'll be exploring are: 

* Data Model
* API
* Transactions
* Persistence
* Indexing

### Data Model

There are fundamentally three categories of data models. They are:  Relational, Hierachical, and Key-Value. 
Most database systems fall distinctly into one camp, but might offer features of the other two. 

#### Relational Models

Relational databases have enjoyed the most popularity throughout recent history. Early on, it was because they made
efficient use of a very rare and valuable resource, hard-disk space.  Lately, however, hard-disk space is incredibly
cheap, they are used widely today because they are very flexible and their models are well understood. Also, SQL, the
lingua franca of relational data, is commonly known among programmers.  

The one downside of relational databases is their storage models don't lend themselves well to storing or retrieving
huge amounts of data.  Query operations against relational tables typically require accessing multiple indexes and
sophisticated schemes which work well for 1GB of data, but not so well for 1TB of data.  

Relational database allow arbitrary dissection of data into "tables" which are defined by a schema. A schema defines the valid data types which will be the "columns" of the data. Columns are the segments of "rows" of data inserted into the database.  The advantage of relational databases is they can store data in a
compact way. By logically breaking up data sets into "logically atomic" parts, data can be referenced instead of
replicated.  When retrieving these objects, the database engine or the client application must fetch these parts and
re-assemble them.  This typically involves joins to additional tables and/or indexes, which increased the overhead and
the overall work done by the database system.  

The fundamental trade-off a relational database makes is that of saving
disk space in return for greater CPU and disk load. 

**Benefits of this model**
* Lowest disk space utilization
* Well understood model and query language. 
* Flexible model to support a wide varity of use cases.
* Schema enforced data consistency

**Downsides of this model**
* Typically the slowest
* Schemas mean higher programmer overhead for iterating changes. 
* High degree of complexity with many tuning knobs


#### Key-Value Models

Key-value stores have been around since the beginning of persistent storage. They have often been used when the
complexity and overhead of relational systems were not required.  Because of their simplicity, efficient storage
models and low runtime overhead, they can usually manage orders of magnitude more operations per second than relational
databases. Lately they are used as a replacement for log collectors, since they can provide basic (usually time based)
indexing. 

Key value stores operate quite simply by having a single data type, typically a chunk of bytes, in a tree, which in turn
points to a single, opaque record. With this simplified model, the system can be heavily optimized towards reading or
writing.  Also, because records are often homogenous is size, and have replicated data, they can be heavily compressed
before being stored on disk. This can drastically reduce the bandwidth required across the SATA bus, which can provide
big gains. 

Through clever row and column creation, and even schema application, key-value stores can be made to simulate some
relational features. They are typically less flexible, however.  If multiple indexes are needed, typically the 2nd index
is just more key-value records.  This means that the application itself must complicate its logic by performing multiple
lookups. 

**Benefits of this model**
* Fast
* Fairly flexible
* Simple, therefore the executables are small with limited dependencies. 

**Downsides of this model**
* Often no schema support, so no consistency checks
* More complicated application logic.

#### Hierarchical or Document Data Models

A model that has achieved popularity relatively recently is the hiercharchal model.  The major advantage of the
hierarchical model is that of ergonomics.  The data is stored and retrieved from the database in the way it stored
within objects, which is typically the way the data is represented in real life.

The hiercharchical model stores all relevant data in a single record, which has delineations for multiple keys and
values, where the values could, themselves be additional associations of keys and values.  

In the general case, all of the data of a real-world object would be found within a single record. This means that it
will necessarily use more storage space than the relational model, because it is replicating the data instead of
referencing it.  It also simplifies the query model, since only a single record needs to be retrieved from a single
table. 

Because the data being stored is very heterogeneous in nature, compression is can provide limited gains and is typically
not used. 

Hierarchical databases typically do offer some relational features, such as foreign references and multiple indexes.
Many such databases do not offer any schema support, as data structure is arbitrary. 

**Benefits of this model**
* The most flexible
* Arbitrary indexes support easy access to data*
* Highest fidelity between application data structures and on-disk data structures. 

**Downsides of this model**
* The highest disk-space usage.
* Without a schema, data-layout is arbitrary so no schema or consistency checks. 


### API

API stands for Abstract Programming Interface, and, in short, it is how you and your program will interact with a
database.  The interface can be diced in many different dimensions, but we'll start with two:

#### In-Process vs Out of Process

If the database itself is running in the same process (at least partially) as the client application, typically a
library of function calls are provided which invoke methods in the database engine directly. This tight coupling results
in the lowest possible latency, and the highest possible bandwidth (memory).
However, it reduces flexibility, since it means that only a single client application can access
the data at one time.  It also poses additional risks: Since they share the same process, if the client application
crashes, so does the database. 

If the Database runs in a seperate process, typically a protocol over TCP/IP is used.  Many RDBMSs (and recently, other
datbases) support either the ODBC or JDBC protocols, which are standardized database connectivity protocols.  This
simplifies the creation of client applications, as the number of libraries which can leverage these protocols are
plentiful.  A network protocol does drastically improve flexibility of a database, but TCP ip carries with it latency
and bandwidth penalties vs those of memory. 

#### SQL vs Not

SQL is a declarative language that was designed originally as a mechanism to simplify storage and retrieval of
relational data.  Its usage is ubiquitous, and as such, many developers speak the language fluently, this can aid the
adoption of a database within an organization and at large. 

> *A note on NoSQL:  The biggest "innovation" touted by most NoSQL databases was simply achieving faster operations by removing transactions and relational tables.  Many of those databases began to support SQL as an API language, even though they didn't need the relational features. Some of the features of SQL such as querying, filtering and aggregating were quite useful.  So it was said that that No-SQL should be renamed to No-ACID, because of their lack of transaction support.  Now many of those same databases have transactional support.  These days, NoSQL might be more accurately called No-Relational, but NoSQL sounds better and is close enough. 

The downside of SQL is that it must be parsed and compiled by the database engine in order to be used.  This imposes a
runtime hit. Most database engines or client APIs work around this by pre-compiling the SQL based function calls into
prepared statement, which are pre-compiled, or compiled on first run, then the compiled version is saved and used for
future calls. 

SQL cannot effectively describe all data relationships.  Hierarchical data, for instance, can not leverage SQL so
database which are hierarchical in nature. "Document", JSON or XML databases fall into this category. 

In many cases the features of databases are so sparse, lacking features such as indexing or aggregation, there is is simply no reason to
support the complexity of a SQL parsing and execution engine. Many in key-value stores fall into this category. 


### Transactions

A database transaction, by definition, is a unit of work treated in a coherent and reliable way.  The most common recipe
for database transactions is ACID, which stands for Atomic, Consistent, Isolated, and Durable. We will discuss each of
  these in-depth below. 

> **NOTE** ACID is just a recipe. Many database systems claim support for transactions or "lightweight" transactions,
> but they may not provide all of the features of ACID, but only those that are convenient/efficient to support. For
> instance, many distributed databases offer the concept of transactions without the Isolation step. This means that the
> data is being modified in-place, and other transactions see that data while its being modified.  One can work around
> this if they know that it is there, if they don't however, the results could be disastrous. 

Let's briefly look at the ACID guarantees, and then what a DB might do to provide them. 

##### Atomicity
Within a transaction, there could be multiple operations. Atomicity guarantees that all operations will either succeed or fail together. Why is this necessary? Let's look at a simple social network database example: 
    

    BEGIN TRANSACTION;
        DELETE "MY_WALL_PIC|http://pics.com/pic_of_dead_cat.jpg" from cover_page;
        INSERT "MY_WALL_PIC=http://pics.com/pic_of_cheeseburger.jpg" into cover_page;
        INSERT "Hey all. Check out tonight's dinner!" into wall;
    END TRANSACTION;

In this not at all contrived example. We see a problem here if some of these instructions succeed without the others. 

If the last instruction succeeds, but the previous two instructions fail, then we might be giving viewers of this page some very incorrect information.

The atomicity guarantee doesn't allow this to happen, if any one instruction fails, then the transaction fails and all
data remains at the state in which it existed before the transaction.

##### Consistency
This guarantee states that the state of the database will be valid to all users before, during and after the transaction.  Databases may make certain guarantees about the data itself. Basic guarantees such as serializability mean that all operations will be processed in the order that they are applied. This might sound easy, but when many applications with many threads are operating on a system concurrently, (expensive) steps must be taken to ensure this is possible. 

Relational databases will often make an even larger set of consistency guarantees. This includes foreign key constraints, cascading operations on dependent types, or triggers that might be executed as part of this operation.

What this means, in terms of performance, is that all of these operations might be running while rows and/or pages are locked for editing, so no other clients will be able to use those parts of the system during that time. It also, clearly effects the round-trip-time of the request.

##### Isolation
Transactions don't happen immediately, they occur in steps, and, like in the Atomicity example, if an outsider were to see a partial set of completed steps, results would range from "amusing" to "horribly wrong". Isolation is the guarantee that say that this won't happen.  It hides away all of the operations from others until the transaction completes successfully. 

##### Durability
A very important trait indeed. Durability simply promises that when the transaction completes, the results of the operations will be successfully persisted on the specified storage medium (typically the hard disk). 

#### Implementation of Transactions

There are 4 steps that are common to an ACID transaction: 

1. Log the incoming request to persistent storage in a transaction log (also known as a Write Ahead Log) This will protect the data in case of a system failure. In the worst case scenario, this transaction will be able to be re-started from the log upon start-up. 
2. Serialize the new values to the index and table data structures in a way that doesn't interfere with existing operations. 
3. Obtain write locks on all cells that need to be modified. Depending on the operation in question and the database. This might mean locking the entire table, the row, or possible the memory page. 
4. Move the new values into place. 
5. Record the transaction as completed in the transaction log.
6. Flush all changes to disk. 

#### Performance Implications of Transactions

**Benefits** 
* Transactions can lead to speed-ups over performing the operations piece-meal, since all of the disk operations are
  batched into a single set of operations. 
* Transactions, if ACID, are a form of concurrency control. Since they sit at the data itself, can often be more
  efficient than custom built concurrency solutions in the application itself. 
 
**Downsides of Transactions**
* Not good for highly concurrent applications. Highly contentious operations will generate excessive replays and aborts (which result in more replays) 
* Complexity - All of the moving parts required to provide transactions add to larger and less maintainable code bases 


### Persistence Models

As was stated above, transactions and even indexing are completely optional within databases. Persistence, however, is the raison d'etre of a database. 

Since the performance game in a database is to minimize disk seeks, we must devise a plan to find the home of a piece of data quickly.  This includes minimizing disk seeks, and also storing data with maximum locality so as to minimize cache load times, and maximize chances for cache hits. 

The performance costs associated with disks (and the data-loss risk associated with the page cache) requires us to make
trade offs with respect to how we store our data.  It is hoped that you select the storage model that is optimized for
your application.  

The different mechanisms for storage are, row store, columnar, distributed or memory-only. 

#### Row Based

To do this, we'll need to store the record data in its own tree based data structure. This is often referred to as a table index. It is technically an index, because trees are tools for efficient look-up.  But we'll use a data structure designed to store large amounts of data while minimizing disk seeks. 

There are two tree style on disk data structures that form the basis of almost all database storage. 

#### B+ Tree
A B+ Tree is a B-tree style index data structure that is optimized for, you guessed it, minimizing disk seeks.  It is one of the most common storage mechanisms in databases for table storage. It is also the data structure of choice for almost all modern filesystems.  

#### Log Structured

The LSM-tree is a newer disk storage structure that is optimized for a high volume of sequential writes.  It was designed to handle massive amounts of streaming events, such as for receiving web server access logs in real-time for later analysis.

Despite its origins in log style event collection, it is beginning to be considered for relational databases as well.  There is a major trade-off in an LSM tree, you cannot delete or update in an LSM data structure. Such events are recorded as new records in the log. When reading an LSM tree, one typically starts from the back in order to read the newest version of the data.  

Periodically, the records which have been made obsolete by subsequent deletes or updates must be garbage collected. 

#### Columnar

#### Distributed 

#### Memory




#### Relational

#### Persistence Performance Summary

To find a location in the tree, it is log base B of N number of searches, where B is the branching factor of the tree and N is the number of items in the tree.   For instance, if it had a branching factor of 10, it would take 6 searches, worst case, to find a specific node in a tree. This could mean 6 disk seeks, compare that with 19.9 for a binary tree, or 2.8 for for a tree with a branching factor of 128!. The default branching factor in many on-disk database b-trees is 128, so we'll assume that for the rest of this article. 

In addition to the insert itself, the tree nodes might need to be split and rebalanced, which could result in additional seeks, although the items in question are likely now in the cache. 

If the database uses MVCC, the tree might be a bit more complex, as it would store and replace nodes which represent versions of the data, as well as the data itself. 

For LSM trees, lets assume that the write to the memory-log causes it to pass a threshold and it needs to be merged to the on-disk log. The write to disk is similar to the B+Tree write above, but it is done in large batches, and since all of the writes are ordered, it is typically done in a small set of sequential writes.  

The LSM tree has two background tasks which must occur frequently which can put additional load on the server and the disk.  It has to frequently flush the data in the memory portion onto the disk. Secondly, it must perform its compacting step against the on-disk portion. The second step can invoke significant overhead. 

B+Tree
lookup or update :  log base 128 of N disk seeks
rebalance on update : log base 128 of N disk seeks

LSM Tree
lookup+rebalance every M operations, where M is the maximum population of the in-memory structure.




### Indexing

Data is rarely stored as isolated values. Typically heterogeneous collections of fields which make up a record. In relational databases, those fields are called columns, and they are fixed to the schema that defines the tables. 

In non-relational databases, heterogeneous fields are still often accommodated and even indexed. When you want to look up a table by specifying one of the fields in a record, that field needs to be part of an index.  

An index is just a data structure for performing random lookups given a specified field (or, where supported, a tuple of specified fields). To avoid ambiguity with the table index described in Persistence, we will call these lookup indices. 

#### To Tree, or not to Tree

Since these are on-disk data structures as well, a B-tree style index is the the tool of choice for most lookup indices since they efficiently support hard disks. B-trees can accommodate inserts efficiently without having to (re)allocate storage cells for each operation. They also tend to be flat in structure, which reduces the number of nodes which need to be searched, therefor reducing the number of potential disk seeks. 

There are other options, however.  For instance, a bitmap index is a data-structure that provides very efficient join queries of multiple tables. 

Tree style indexes grow linearly for the number of items items in the tree, and search time grows with the depth of the tree (a logarithmic function of the total depth) 

Bitmap indices, on the other hand, grow with the number of *different* items in a column.  As the name implies, they build bitmaps which represent the membership of values for all relevant columns.  Multiple boolean operations against bitmap indexes are very fast, they themselves produce new bitmaps which can be cached efficiently as search results. 

One of the other major innovations of bitmap indices is that they can be compressed, and they can even perform query operations while compressed. This makes storage, retrieval faster. It also makes them more CPU cache friendly, which can further reduce latencies. 

#### Lookup Index to Table Index Referencing

The value found at the key in an index is actually a pointer back to where the row is physically stored. This can be done two ways:  

1. Storing the physical offset directly.  The advantage here is incredibly fast lookups.  The downside is that write speed is reduced. Whenever more data is inserted into the table index, all effected storage locations must have their new offsets updated within the lookup index.  

2. Storing the ID of the row.  This means that the database must, in turn, look up the row in the table index by ID. The upside of this extra layer of indirection is no modification of the lookup index is necessary when new data is added to the table index.

#### Index Performance Summary

The cost of B-Tree style indices are discussed above. In the case of lookup-indices, they store less data, and are often optimized to better fit into the page cache, and even the CPU cache.  

Bitmap indices often have a much smaller memory footprint, and can sit in memory. They have one major drawback, however, and that is that they cannot be efficiently updated. Since the data is stored, quite literally as a matrix of bits packed into memory, large updates would require a full scan and modification of the index. 

costs: 
lookup or update :  log base 128 of N disk seeks
re-balance on update : log base 128 of N disk seeks

Odds are that if a re-balance is required, the initial traversal through the tree loaded all relevant data into the page cache, so there would be no additional disk seeks. 

### Pulling it all together

If there is one take-away here, it is that there are many knobs to turn on a database and an operating system to optimize performance. The trade-offs all revolve around the disk.  If you don't need guaranteed, immediate durability for every operation, you can delay persisting the operation to disk and leverage an in-memory data structure temporarily. This can be accomplished either manually in your own batches, such as the in-memory store of the LSM tree, or automatically, via the page cache. 

Understand of course that the risk of data-loss is present any time you rely on memory to speed things up. If a failure occurs, those pending writes can disappear.  If you are operating in an environment where data loss is acceptable, then by all means use in-memory structures, or don't bother flushing the page cache to disk. 

Regardless of your application, you should take time to understand the page cache in your OS. Writes that you think are safe may not be.  It, too, has many settings for fine-tuning performance.  It can be set to be highly paranoid, but busy, or care-free and fast. It is worth verifying that your expectations match reality. 

We now know that write benchmarks for an ACID, b-tree style database should come nowhere close to those of an LSM style log event collector. It would be silly to compare such things. We also know that it would be folly to use an in-memory database to run bank transactions. 

As with all software systems, check to see if a database's features match your requirements before comparing its speed.

It may just save your data. 


