# Disambiguating Databases

The topic of data storage is one that doesn't to be well understood until something goes wrong (data disappears), or something goes really right (too many customers). Because databases can be treated like black boxes with an API, their inner workings are often overlooked. They're often treated as magic things that just take data when offered, and supply it when I asked. Since these two operations are the only understood activities of the technology, they are the only feature points used when one is comparing different technologies.

But what exactly is an *operation*? Within the realm of databases, this could mean any number of things. Is that operation a transaction? Is it an indexing of data? A retrieval from an index? Does it store the data to a durable medium like a hard disk, or does it beam it by laser towards Alpha Centauri? 

It is this ambiguity that causes havoc in the software industry. Misunderstanding the features and guarantees of a database system can cause, at best, user consternation due to slowness or unavailability. At worst, it could result in fiscal damage, or even jail time due to loss data. 

The scope of the term "database" is vast.  Technically speaking, anything that stores data for later retrieval is a database. Even by that broad definition, there is functionality that is common to most databases.  The scope of this article is to enumerate those features.  The intent is to provide the reader with a tool-set with which they might evaluate databases on their relative merits. 

Applying this feature driven approach, the reader should see two benefits. Firstly, it will allow them to accurately assess their own needs. Secondly, it will allow them to compare technologies by pairing up like features. When viewed with with this lens, comparative benchmarks will only be valid on databases that are performing equal work and providing the same guarantees.

The best way to illustrate the features of a database is step through a database, from query through storage and back, one operation at a time. 

## A datum's trip through a database

We will briefly what happens to data from the moment it arrives at the database's query processing engine through its resting place in RAM or on Disk in a database. The intent is to provide the big picture before drilling down into the constituent parts in subsequent sections.

\#\#\# Insert Magnificent Flow Chart Here which shows the inner steps enveloped by a transaction \#\#\# 

1. Query Processing
At the very least, a query engine must parse a query into a request data structure which reflects which sections of which tables need to be retrieved. In relational or distributed database management systems this process features a query planner. It is usually much more advanced.  

The query planner looks at the state of the indices that it must traverse in order to complete the query. It might also compare statistics such as table length, record size, or cardinality.  It also might change its search algorithm, or parallelize its operations when possible. For complex queries, the query itself may go through simplification and optimization steps.

The few microseconds it spends assessing this information that it keeps handy can shave off seconds or minutes of table and index traversal.  

2. Transaction
After the request has been unpacked by the query processing engine, it might possibly begin a transaction. Transactions are intended to provide guarantees regarding data consistency and durability, but usually come and a significant cost.

3. Persistence
If the point of a database is to store data for later retrieval, then we have reached the entire reason for its existence. How (and even when) data is persisted can have the single biggest impact on the speed and success of data storage and retrieval operations. 

4. Indexing
Typically, storing the data isn't enough, it must also be retrieved. Indexing is a way of calculating and storing additional pathways with which persisted data might be quickly found and read.

## Some Note on Performance

Since this article is about comparing the performance characteristics of different database features, it is important to note the relative costs of the activities involved in persisting data within a database. 

### The Hard Disk

The highest latency operation you will encounter within a data center is a seek to a random location on a hard disk. For the purposes of this article, all timings presented will be for that of a 7200 RPM drive, which is typical of a multi-terabyte commodity drive found in data intensive applications.

> *SSDs have brought massive latency and throughput improvements to disks. The same seek on an SSD is about 60 times faster. They bring their own challenges, however, one interesting one is that the storage cells within an SSD have a fixed lifetime, that is, they can only handle so many writes to them before they fail.  So they have specialized firmware that spreads writes around the disk, garbage collects, and does other bookkeeping operations. Because of this, they have less predictable performance characteristics (though they are predictably faster than hard disks) *

It takes 4.17 milliseconds to seek to a random location on a hard disk. This is because the spindle on which the data is stored actually has to start rotating, spin to the correct location, then stop accurately to within a nanometer. Frankly it is amazing that they are as fast as they are. 

Once a location has been found, sequentially reading from, or writing to the disk starting at that location is significantly cheaper.  This is called a sequential read or write. 

Algorithms regarding data storage and retrieval have been optimized against this fact since magnetic rotating disks were invented.

### The network

The only other operation with latencies within 3 orders of magnitude of a disk seek is a network round trip. It takes about 0.5ms for a packet to go from machine A to machine B, and then for Machine B to turn around to send one back.  

To establish a TCP connection via the [3 way handshake](http://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment) it would take about 0.75 ms.  Subsequent requests (that fit into a packet) would then take 0.5ms (for the data and the ack) 

With this understanding, many data systems have taken to distributing data across a network, in the memory of other computers, as a low-latency replacement for local disk based storage. 

### Memory

Finding a byte of data in RAM on a machine can be done in approximately 100 nanoseconds, or 41 *thousand* times faster than a hard disk seek.  

This means that data structures in memory can be more complex and nuanced than data structures on disk. The design of every database has taken this into careful consideration. 

### The Page Cache

Because of the massive latency costs that hard disks impose, one optimization that is found in nearly every operating system is the page cache, or buffer cache.

The 
 
## The features of a database

We now know how the database operates at a high level, and we understand the relative cost of using the tools a computer provides for us. It's time to do a deep-dive into the core features while using our performance knowledge to attribute latency characteristics to each operation.

### Transaction
Transactions are typically used in DBMS's as a boundary to the beginning and end to the set of operations on data. In a busy, concurrent database, many things can go wrong, even when things are going well, writes can still fail simply because an earlier operation has locked or changed a cell.  Transactions provide many guarantees as well as a standard programmatic interface to eliminate the hassle of having to handle all potential failure modes in code. 

#### ACID
When transactions are provided, they are generally used to provide a set of guarantees commonly known as ACID. ACID stands for Atomicity, Consistency, Isolation and Durability.  

Let's briefly look at the ACID guarantees, and then what a DB might do to provide them. 

##### Atomicity
Within a transaction, there could be multiple operations. Atomicity guarantees that all operations will either succeed or fail together. Why is this necessary? Let's look at a simple social network database example: 
    

    BEGIN TRANSACTION;
        DELETE "http://pics.com/pic_of_dead_cat.jpg" from cover_page;
        INSERT "http://pics.com/pic_of_cheeseburger.jpg" into cover_page;
        INSERT "Hey all. Check out tonight's dinner!" into wall;
    END TRANSACTION;

In this not at all contrived example. We see a problem here if some of these instructions succeed without the others. 

If the insert instruction on line 4 succeeds, but the previous two instructions fail, then we might be giving viewers of this page some very incorrect information.

The atomicity guarantee doesn't allow this to happen, if any one instruction fails, then they all fail together.

##### Consistency
This guarantee states that the state of the database will be valid to all users before, during and after the transaction.  Databases may make certain guarantees about the data itself. Basic guarantees such as serializability mean that all operations will be processed in the order that they are applied. This might sound easy, but when many applications with many threads are operating on a system concurrently, (expensive) steps must be taken to ensure this is possible. 

Relational databases will often make an even larger set of consistency guarantees. This includes foreign key constraints, cascading operations on dependent types, or triggers that might be executed as part of this operation.

What this means, in terms of performance, is that all of these operations might be running while rows and tables are locked for editing, so no other clients will be able to use those parts of the system during that time. It also, clearly effects the round-trip-time of the request.

##### Isolation
Transactions don't happen immediately, they occur in steps, and, like in the Atomicity example, if an outsider were to see a partial set of completed steps, results would range from "amusing" to "horrible wrong". Isolation is the guarantee that say that this won't happen.  It hides away all of the operations from others until the transaction completes successfully. 

##### Durability
A very important trait indeed. Durability simply promises that when the transaction completes, the results of the operations will be successfully persisted on the specified storage medium (typically the hard disk). 

#### Implementation of Transactions

There are 4 steps that are common to an ACID transaction: 

1. Log the incoming request to persistent storage in a transaction log. This will protect the data in case of a system failure. In the worst case scenario, this transaction will be able to be re-started from the log upon start-up. 
2. Serialize the new values to the index and table data structures in a way that doesn't interfere with existing operations. 
3. Obtain write locks on all cells that need to be modified. Depending on the operation in question and the database. This might mean locking the entire table, the row, or possible the memory page. 
4. Move the new values into place. 
5. Flush all changes to disk. 

Note that these 5 steps are where every transactional database system keeps its secret sauce.  Transactions come at a significant cost. To be able to optimize this process even a little bit will provide customers with an advantage. Every DBMS will execute steps 1-4 in many different ways. They might try to execute all or some of the steps in parallel, they might leverage highly specialized data structure systems, such as MVCC, which reduce the need for locking.  In future articles, we will survey prevailing technologies and their approaches. 

> *A note on NoSQL:  The biggest "innovation" touted by most NoSQL databases was simply achieving faster operations by removing transactions. It has been stated that NoSQL should more correctly be termed NoACID. *

#### Transaction Performance (Summary) 


- Store transaction to transaction log - 1 disk seek. 1 sequential write
- Lock various rows - N memory seeks / modifications, disrupt concurrency
- (Optionally) Consistency and Integrity checks - N disk and/or memory seeks
- Move new structures into place - 1 to N memory seeks.
- Flush memory mapped changes to disk. 
Total cost: 

### Persistence

As was stated above, transactions and even indexing are completely optional within databases, depending on their scope. Persistence, however, is the raison d'etre of a database. 

---more stuff

So how do you find rows in a database on disk while minimizing disk seeks? You have two options: 

1. Read the entire database table into memory (not often possible, and not really recommended)

2. Store your record data in its own index. This is often referred to as a table index. While it is often a tree style index in its own right, it is also responsible for storing the record data itself. So it is not to be confused with the indexing step we'll be discussing in the next section.


There are two tree style on disk data structures that form the basis of almost all database storage. 

1. B+ Tree
A B+ Tree is a B-tree style index data structure that is optimized for, you guessed it, minimizing disk seeks.  It is one of the most common storage mechanisms in databases for table storage. It is also the data structure of choice for almost all modern filesystems.  


2. Log Structured Merge Tree
The LSM-tree is a newer disk storage structure that is optimized for a high volume of sequential writes.  It was designed to handle massive amounts of streaming events, such as for receiving web server access logs in real-time for later analysis.

Despite its origins in log style event collection, it is beginning to be adopted by relational databases as well. 

-- more details

3. Write Flush Queue

-- more details

### Indexing

The value found at the key in an index is actually a pointer back to where the row is physically stored. This can be done two ways:  

1. Storing the physical offset directly.  The advantage here is incredibly fast lookups.  The downside is that all write speed is reduced because any time more data is inserted, all effected values must have their offsets updated.  This effect can be mitigated in trees which store only a handful of rows per leaf. 

2. Storing the ID of the row.  This means that the database must, in turn, look up the row in the table index by ID. The upside of this extra layer of indirection is no modification is necessary when new data is added. 
