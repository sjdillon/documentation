# How many Cassandra nodes do we need running?
## Overview
When we have issues on Cassandra, I often get asked how many nodes we
need running in the cluster. The answer for for the application they
normally care about, is **9 out of 10 nodes**. The "asker" usually
thinks that’s terrible and moves on to bashing Cassandra.

They rarely are interested in **WHY** we need **90%** of the nodes
running. They wrongly assume this is a Cassandra limitation. It is not.
We could run with as few as **1 out of 10 nodes** if we chose to.

The explanation first requires a basic understanding of Cassandra.

## Replication Factors and Consistency Levels
### Replication Factor
Replication Factor is set at the keyspace level. A keyspace is basically
the same as a database in SQL Server or MySQL. It’s *sort of* like
database or a schema in oracle, but there isn’t an easy corollary.

We normally choose a **RF=3** for our keyspaces. This means that a
record will **eventually** be written to 3 nodes. The word
**"EVENTUALLY"** is important here.

Below is an example of a USER table with a primary key of loginname, and
a record for user **OZONE** written to 3 nodes.


| node | record |
|:-----|:-------|
| 1    | **OZONE**   |
| 2    | **OZONE**   |
| 3    | X      |
| 4    | **OZONE**   |
| 5    | X      |
| 6    | X      |
| 7    | X      |


## Consistency Level
Cassandra has configurable consistency. This consistency is determined
by the application, at the time data is written. This consistency can be
strong or eventually consistent. Strong would mean that it requires
records to be written to all the nodes (based on the RF, 3 nodes for us)
**before** acknowledging it as complete and successful.

Consistency is specified separately for both reads and writes. 

We specify a **QUORUM** for most of our apps.  So for a **RF of **3, that means a **2 nodes** must acknowledge **reads** and **writes**.  I’m going to stick with calling it **CL = 2** for the rest of this example.

**Read Consistency Level (RCL)**
```
cqlsh> CONSISTENCY TWO

cqlsh> SELECT * FROM USERS WHERE LOGINNAME=’OZONE’
```
 

This query will pull the **OZONE** record from two nodes and compare the results for consistency before returning the data.  If they don’t match, it has logic to determine and make the record consistent.

## Write Consistency Level (WCL)

```
cqlsh> CONSISTENCY TWO
cqlsh> INSERT INTO USERS (LOGINNAME) VALUES  (‘OZONE’);
```
 

This insert will write **OZONE** to 3 nodes (based on the RF defined at the keyspace) eventually, but for now, it will acknowledge success after getting inserts confirmed from just 2 nodes. 

Again, the insert is sent to 3 nodes, but the app can move on once two nodes acknowledge the insert (the app could specify 1, 2, Quorum, ALL, it’s up to up to the app designer’s needs).

 

## Node Down!
With our settings (**RF=3, RCL=2, WCL=2**) we can run with 1 node down, any more and some* transactions will begin to fail.

 

####  One node down 

| node | record    |
|:-----|:----------|
| 1    | **OZONE** |
| 2    | **OZONE** |
| 3    | X         |
| 4    | **~~OZONE~~** |
| 5    | X         |
| 6    | X         |
| 7    | X         |
| 8    | X         |
| 9    | X         |
| 10   | X         |

INSERTS with CL=2, can still complete.  All we need is 2 acknowledgements, the 3rd will be queued up for later.

READS – we can still read the OZONE record from 2 nodes, and that’s all we are asking for.

####  Two nodes down 


| node  | record   |
|:------|:---------|
| 1     | **OZONE**     |
| ~~2~~ | **~~OZONE~~** |
| 3     | X        |
| ~~4~~ | **~~OZONE~~** |
| 5     | X        |
| 6     | X        |
| 7     | X        |
| 8     | X        |
| 9     | X        |
| 10    | X        |




INSERTS with CL=2, cannot reach enough nodes, the insert fails (not written to any nodes).

READS – We cannot achieve the requested CL, so the read fails.  Again, the app specified this as a requirement. 



####  BUT, why do some transactions succeed?


| node  | record   |
|:------|:---------|
| 1     | **OZONE**     |
| 2     | **OZONE**     |
| 3     | X        |
| ~~4~~ | **~~OZONE~~** |
| 5     | CRUACHAN   |
| 6     | X        |
| 7     | CRUACHAN   |
| 8     | X        |
| 9     | X        |
| 10    | X        |



```
INSERT INTO USERS (LOGINNAME) VALUES (‘CRUACHAN’); 
SELECT * FROM USERS WHERE LOGINNAME=’CRUACHAN’;
```

**‘CRUACHAN’** maps to nodes that are up, so reads and writes to those nodes
will continue without any exceptions


####  The problem with BATCHES
Applications that batch up data, either in the query, insert or within the application logic, can amplify the impact of failure.  If the above query of CRUACHAN and OZONE were combined to request both records, the whole batch would fail, because OZONE was unavailable, even though CRUACHAN was available. 


#### Retry Policies
Cassandra is highly configurable. We also have retrypolicies in the
SDKs. One of these allows for a downgrading retry. This will start with
a specified CL and retry at a lower CL. IF a CL=2 fails, it will retry
with a CL=1.
