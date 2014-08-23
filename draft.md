# Disambiguating Databases

The scope of databases is vast.  Technically speaking, anything that stores data for later retrieval is a database. Even by that broad definition, there is some common functionality that databases share.  The scope of this article is to enumerate the features common to most databases.  The intent is to provide the reader with a tool-set with which they might evaluate databases on their relative merits by feature. Applying this feature driven approach, the reader should see two benefits. Firstly, it will allow them to accurately assess their own needs. Secondly, it will provide a grounding to cut through the hype that seems to surround database technology and evaluate new technologies solely on their use cases. 

Typically there is one number that people look for when evaluating a database, and that is operations per second. Raw speed.  It's fun and all, but what exactly is an *operation*? Within the realm of databases, this could mean any number of things. Is that operation a transaction? Is it an indexing of data? A retrieval from an index? Does it store the data to a durable medium like a hard disk, or does it beam it by laser towards Alpha Centauri? It is this ambiguity that causes havoc in the software industry. Misunderstanding the features and guarantees of a database system can cause, at best, user consternation due to slowness or unavailability. At worst, it might cause loss of business or even jail time, due to loss data. 

Let's explore some of the operations that might be undertaken when we attempt to store some data for later retrieval. 

## A datum's trip through a database

We will briefly what happens to data from the moment it arrives at the database's query processing engine through its resting place in RAM or on Disk in a database. The intent is to provide the big picture before drilling down into the constituent parts in subsequent sections.

\#\#\# Insert Flow Chart Here \#\#\#

1. Protocol
When interacting with a database, the system will typically offer one of two choices for its protocol: Binary or Query Language.  

2. Transaction
After the request has been unpacked by the query processing engine, it might possibly begin a transaction. Transactions are intended to provide guarantees regarding data consistency and durability, but come and a significant cost.

3. Persistence
If the point of a database is to store data for later retrieval, then we have reached the entire reason for its existence. How (and even when) data is persisted can have the single biggest impact on the speed and success of data storage and retrieval operations. 

4. Indexing
Typically, storing the data isn't enough, it must also be retrieved. Indexing is a way of calculating and storing additional pathways with which persisted data might be quickly found and read.

> *Note: Relational Database Management Systems will offer an additional set of steps and features which can offer additional guarantees of data consistency at the cost of complexity for both the user and the DB engine, as well as increased CPU and disk load.* 


### Protocol 
The line between these two categories can sometimes be blurred but, in general, a **binary protocol** is accessed with functions in the API of the database, and those function calls are translated to a packed binary network protocol such as (BSON)[http://bsonspec.org/] 

The advantage of a binary protocol is that it is quite bandwidth efficient, and can be quickly serialized and deserialized by the client and the server, leading to less CPU and network usage. 

A **query language**, such as [SQL](http://en.wikipedia.org/wiki/SQL) or [Datalog](http://docs.datomic.com/query.html) will offer the user much more clear and concise interaction with their data. The trade-off is that a specialized parser must be built to interpret the language, which can add latency to the request. 

Unless the application is incredibly latency sensitive (e.g. measured in the 100's of microseconds or less) the protocol a database uses has a negligible impact on performance. For this reason, we will ignore the impact of communication protocols on database performance. 

### Transaction
Transactions are typically used in DBMS's as a boundary to the beginning and end to the set of operations on data.

#### ACID
Typically they are used to provide a set of guarantees commonly known as ACID. ACID stands for Atomicity, Consistency, Isolation and Durability.  

Let's briefly look at the ACID guarantees, and then what a DB might do to provide them. 

##### Atomicity
Within a transaction, there could be multiple operations. Atomicity guarantees that they will all succeed or fail together. Why is this necessary? Let's look at a simple social network database example: 
    

    BEGIN TRANSACTION;
        DELETE "http://pics.com/pic_of_dead_cat.jpg" from cover_page;
        INSERT "http://pics.com/pic_of_cheeseburger.jpg" into cover_page;
        INSERT "Hey all. Check out tonight's dinner!" into wall;
    END TRANSACTION;

In this not at all contrived example. We see a problem here if some of these instructions succeed without the others. 

If the insert instruction on line 4 succeeds, but the previous two instructions fail, then we might be giving viewers of this page some very incorrect information.

The atomicity guarantee doesnt allow this to happen, if any one instruction fails, then they all fail together.

##### Consistency
This guarantee states that the state of the database will be valid to all users before, during and after the transaction.  Databases may make certain guarantees about the data itself. Basic guarantees such as serializability mean that all operations will be processed in the order that they are applied. This might sound easy, but when many applications with many threads are operating on a system concurrently, (expensive) steps must be taken to ensure this is possible. 

Relational databases will often make an even larger set of consistency guarantees. This includes foreign key constraints, cascading operations on dependent types, or triggers that might be executed as part of this operation.

What this means, in terms of performance, is that all of these operations might be running while rows and tables are locked for editing, so no other clients will be able to use those parts of the system during that time. It also, clearly effects the round-trip-time of the request.

##### Isolation
Transactions don't happen immediately, they occur in steps, and, like in the Atomicity example, if an outsider were to see a partial set of completed steps, results would range from "amusing" to "horrible wrong". Isolation is the guarantee that say that this won't happen.  It hides away all of the operations from others until the transaction completes successfully. 

##### Durability
A very important trait indeed. Durability simply promises that when the transaction completes, the results of the operations will be successfully persisted on the storage  medium (typically the hard disk). 


> *A note on NoSQL:  The biggest "innovation" touted by most NoSQL databases was simply achieving faster operations by removing transactions. It has been stated that NoSQL should more correctly be termed NoACID. In fact, many nosql database now support SQL engines, even if they're not relational. *


#### Implementation of Transactions

How would a database go about executing a transaction? There are several methods that are common among databases, I will enumerate them now and attempt to detail the costs of each. 



