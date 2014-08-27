# Disambiguating Databases

Typically there is one number that people look for when evaluating a database, and that is operations per second. Raw speed.  It's fun and all, but what exactly is an *operation*? Within the realm of databases, this could mean any number of things. Is that operation a transaction? Is it an indexing of data? A retrieval from an index? Does it store the data to a durable medium like a hard disk, or does it beam it by laser towards Alpha Centauri? 

It is this ambiguity that causes havoc in the software industry. Misunderstanding the features and guarantees of a database system can cause, at best, user consternation due to slowness or unavailability. At worst, it could result in fiscal damage, or even jail time, due to loss data. 

The scope of the term database is vast.  Technically speaking, anything that stores data for later retrieval is a database. Even by that broad definition, there is functionality that is common to all databases.  The scope of this article is to enumerate those features.  The intent is to provide the reader with a tool-set with which they might evaluate databases on their relative merits. 

Applying this feature driven approach, the reader should see two benefits. Firstly, it will allow them to accurately assess their own needs. Secondly, it will allow them to compare technologies by pairing up like features. When viewed with with this lens, comparative benchmarks will only be valid on databases that are performing equal work and providing the same guarantees.

The best way to illustrate the features of a database is step through a database, from query through storage and back, one operation at a time. 

## A datum's trip through a database

We will briefly what happens to data from the moment it arrives at the database's query processing engine through its resting place in RAM or on Disk in a database. The intent is to provide the big picture before drilling down into the constituent parts in subsequent sections.

\#\#\# Insert Flow Chart Here \#\#\# 

1. Query Processing
At the very least, a query engine must parse a query into a request data structure which reflects which sections of which tables need to be retrieved. In relational or distributed database management systems this process features a query planner. It is usually much more advanced.  

The query planner looks at the state of the indices that it must traverse in order to complete the query. It might also compare statistics such as table length, record size, or cardinality.  It also might change its search algorithm, or parallelize its operations when possible. For complex queries, the query itself may go through simplification and optimization steps.

The few microseconds it spends assessing this information that it keeps handy can shave off seconds or minutes of table and index traversal.  

2. Transaction
After the request has been unpacked by the query processing engine, it might possibly begin a transaction. Transactions are intended to provide guarantees regarding data consistency and durability, but come and a significant cost.

3. Persistence
If the point of a database is to store data for later retrieval, then we have reached the entire reason for its existence. How (and even when) data is persisted can have the single biggest impact on the speed and success of data storage and retrieval operations. 

4. Indexing
Typically, storing the data isn't enough, it must also be retrieved. Indexing is a way of calculating and storing additional pathways with which persisted data might be quickly found and read.

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

Note that these 4 steps are where every transactional database system keeps its secret sauce.  Transactions come at a significant cost. To be able to optimize this process even a little bit will provide customers with an advantage. Every DBMS will execute steps 1-4 in many different ways. They might try to execute all or some of the steps in parallel, they might leverage highly specialized data structures which reduce the need for locking.  In future articles, we will survey prevailing technologies and their approaches. 

> *A note on NoSQL:  The biggest "innovation" touted by most NoSQL databases was simply achieving faster operations by removing transactions. It has been stated that NoSQL should more correctly be termed NoACID. *


### Persistence

As was stated above, transactions and even indexing are completely optional within databases, depending on their scope. Persistence, however, is the raison d'etre of a database. 



#### Optimizing against the hard disk. 

It takes 10 milliseconds to find a location on a hard disk to read or write. This is what is commonly referred to ask a disk seek.  

Once a location has been found, sequentially reading from, or writing to the disk starting at that location is significantly cheaper. 

Algorithms regarding data storage and retrieval have been optimized against this fact since magnetic disks were invented.

So how do you find rows in a database on disk while minimizing disk seeks? You have two options: 

1. Read the entire database table into memory (not often possible, and not really recommended)

2. Store your record data in its own index. This is often referred to as a table index. While it is often a tree style index in its own right, it is also responsible for storing the record data itself. So it is not to be confused with the indexing step we'll be discussing in the next section.

> *SSDs have brought massive latency and throughput improvements to disks. The same seek on an SSD is about 60 times faster. They bring their own challenges, however, one interesting one is that the storage cells within an SSD have a fixed lifetime, that is, they can only handle so many writes to them before they fail.  So they have specialized firmware that spreads writes around the disk, garbage collects, and does other bookkeeping operations. Because of this, they have less predictable performance characteristics (though they are predictably faster than hard disks) *

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
