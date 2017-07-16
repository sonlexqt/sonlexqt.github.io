---
layout: post
title: Replication in MongoDB
bigimg: /img/mongodb.png
---

A **replica set** in MongoDB is a group of [mongod](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) processes that maintain the same data set. Replica sets provide *redundancy* and *high availability*, and are the basis for all production deployments. With multiple copies of data on different database servers, replication provides a level of fault tolerance against the loss of a single database server.



## Members of a Replica Set

![](https://docs.mongodb.com/manual/_images/replica-set-read-write-operations-primary.bakedsvg.svg)

A replica set can have only one **primary** node, which receives all write operations. The primary records all changes to its data sets in its operation log, i.e. **oplog**. By default, an application redirects its read operations to the primary member.

The **secondaries** maintains a copy of the primary's data set. To replicate data, a secondary applies operations from the primary's oplog to its own data set in an *asynchronous* process.

![](https://docs.mongodb.com/manual/_images/replica-set-primary-with-secondary-and-arbiter.bakedsvg.svg)

**Arbiters** do not keep a copy of the data. However, arbiters play a role in the elections that vote for a primary if the current primary is unavailable. A replica set can have up to 50 members, but only 7 of them can vote.



## Replica Set Elections - Automatic Failover

Replica sets use **elections** to determine which set member will become primary. Elections occur after when:

- A replica set is being initiated
- A primary does not communicate with the other members of the set for more than 10 seconds

An eligible secondary will hold an election to elect itself the new primary. The first secondary to hold an election and receive a majority of the members’ votes becomes primary.

![](https://docs.mongodb.com/manual/_images/replica-set-trigger-election.bakedsvg.svg)

**While an election is in progress, the replica set has no primary therefore cannot accept writes and all remaining members become read-only.** MongoDB avoids elections unless necessary. Also, if a majority of the replica set (e.g 2 out of 3, 4 out of 7) is inaccessable/unavailable to the current primary, the primary will step down and become a secondary. The replica set cannot accept writes after this occurs, but remaining members can continue to serve read queries.



## Special Set Members

#### Priority 0 Replica Set Members

A **priority 0** member is a secondary that *cannot* become primary. Priority 0 members *cannot* trigger election. Otherwise these members function as normal secondaries. A priority 0 member maintains a copy of the data set, accepts read operations, and votes in elections. Configure a priority 0 member to prevent secondaries from becoming primary, which is particularly useful in multi-data center deployments.

Example use case of this is when you have two data centers: DC1 in California and DC2 in Oregon. DC2 contains a secondary member and has networking restraint as well as limited resources. Therefore we set the secondary member's priority in DC2 to 0 to prevent it from becoming a primary.

![](https://docs.mongodb.com/manual/_images/replica-set-three-members-geographically-distributed.bakedsvg.svg)

#### Hidden Replica Set Members

A **hidden** member maintains a copy of the primary’s data set but is **invisible** to client applications. Hidden members are good for workloads with different usage patterns from the other members in the replica set. Hidden members must always be priority 0 members and so cannot become primary. Hidden members, however, **may vote** in elections.

![](https://docs.mongodb.com/manual/_images/replica-set-hidden-member.bakedsvg.svg)

#### Delayed Replica Set Members

**Delayed** members contain copies of a replica set’s data set. However, a delayed member’s data set **reflects an earlier, or delayed, state of the set**. For example, if the current time is 09:52 and a member has a delay of an hour, the delayed member has no operation more recent than 08:52. Because delayed members are a running "historical" snapshot of the data set, they may help you recover from various kinds of human error. For example, a delayed member can make it possible to recover from unsuccessful application upgrades and operator errors including dropped databases and collections.

![](https://docs.mongodb.com/manual/_images/replica-set-delayed-member.bakedsvg.svg)

Delayed members:

- Must be priority 0 members (so that one of delayed members can't become primary)
- Should be hidden members (so that applications are prevented from seeing and querying delayed members)
- Do vote in elections for primary




## Read Operations

By default, clients read from the primary; however, clients can specify a **read preference** to send read operations to secondaries. Asynchronous replication to secondaries means that reads from secondaries may return data that does not reflect the state of the data on the primary.

![](https://docs.mongodb.com/manual/_images/replica-set-read-preference.bakedsvg.svg)

MongoDB drivers support five read preference modes.

| Read Pref. Mode                          | Description                              |
| ---------------------------------------- | ---------------------------------------- |
| [`primary`](https://docs.mongodb.com/manual/reference/read-preference/#primary) | Default mode. All operations read from the current replica set primary. |
| [`primaryPreferred`](https://docs.mongodb.com/manual/reference/read-preference/#primaryPreferred) | In most situations, operations read from the primary but if it is unavailable, operations read from secondary members. |
| [`secondary`](https://docs.mongodb.com/manual/reference/read-preference/#secondary) | All operations read from the secondary members of the replica set. |
| [`secondaryPreferred`](https://docs.mongodb.com/manual/reference/read-preference/#secondaryPreferred) | In most situations, operations read from secondary members but if no secondary members are available, operations read from the primary. |
| [`nearest`](https://docs.mongodb.com/manual/reference/read-preference/#nearest) | Operations read from member of the replica set with the least network latency, irrespective of the member’s type. |



## Replica Set Deployment

#### Consider Fault Tolerance

*Fault tolerance* for a replica set is the number of members that can become unavailable and still leave enough members in the set to elect a primary. In other words, it is the difference between the number of members in the set and the majority of voting members needed to elect a primary.

| Number of Members | Majority Required to Elect a New Primary | Fault Tolerance (the higher the better) |
| ----------------- | ---------------------------------------- | --------------------------------------- |
| 3                 | 2                                        | 1                                       |
| 4                 | 3                                        | 1                                       |
| 5                 | 3                                        | 2                                       |
| 6                 | 4                                        | 2                                       |

From this table, we can imply that:

* Adding a member to the replica set does not always increase the fault tolerance
* Deploy an odd number of members. Given `n >= 1`, we don't have to go for a `2n + 2` number of members because `2n + 1` and `2n + 2` have the same value of fault tolerance (which is `ceil(n/2) - 1` ). If you're actually having `2n + 2` members, deploy an *arbiter* so that the set has an odd number of voting members (`2n + 3`) and higher fault tolerance (`ceil((n + 1)/2) - 1`).

#### Use Hidden and Delayed Members for Dedicated Functions 

Add hidden or delayed members to support dedicated functions, such as backup or reporting.

#### Load Balance on Read-Heavy Deployments 

In a deployment with very high read traffic, you can improve read throughput by distributing reads to secondary members. As your deployment grows, add or move members to alternate data centers to improve redundancy and availability. Always ensure that the main facility is able to elect a primary.

#### Distribute Members Geographically

To protect your data in case of a data center failure, keep at least one member in an alternate data center. If possible, use an odd number of data centers, and choose a distribution of members that maximizes the likelihood that even with a loss of a data center, the remaining replica set members can form a majority or at minimum, provide a copy of your data.

##### Three-member Replica Set

For a replica set with 3 members, some possible distributions of members include:

- Two data centers: two members to Data Center 1 and one member to Data Center 2. If one of the members of the replica set is an arbiter, distribute the arbiter to Data Center 1 with a data-bearing member.

  * If Data Center 1 goes down, the replica set becomes read-only.

  - If Data Center 2 goes down, the replica set remains writeable as the members in Data Center 1 can hold an election.

- Three data centers: one members to Data Center 1, one member to Data Center 2, and one member to Data Center 3.

  - If any Data Center goes down, the replica set remains writeable as the remaining members can hold an election.

##### Five-member Replica Set

For a replica set with 5 members, some possible distributions of members include:

- Two data centers: three members to Data Center 1 and two members to Data Center 2.
  - If Data Center 1 goes down, the replica set becomes read-only.
  - If Data Center 2 goes down, the replica set remains writeable as the members in Data Center 1 can create a majority.
- Three data centers: two member to Data Center 1, two members to Data Center 2, and one member to site Data Center 3.
  - If any Data Center goes down, the replica set remains writeable as the remaining members can hold an election.

For example, the following 5 member replica set distributes its members across three data centers.

![](https://docs.mongodb.com/manual/_images/replica-set-three-data-centers.bakedsvg.svg)

#### Electability of Members

Some members of the replica set, such as members that have networking restraint or limited resources, should not be able to become primary in a failover. Configure members that should not become primary to have priority 0. 

In some cases, you may prefer that the members in one data center be elected primary before the members in the other data centers. You can modify the priority of the members such that the members in the one data center has higher priority than the members in the other data centers.

![](https://docs.mongodb.com/manual/_images/replica-set-three-data-centers-priority.bakedsvg.svg)

## References

1. [MongoDB’s official Replication documentation]([https://docs.mongodb.com/manual/replication/](https://docs.mongodb.com/manual/replication/))
2. [Replication Election and Consensus Algorithm Refinements for MongoDB 3.2]([https://www.mongodb.com/presentations/replication-election-and-consensus-algorithm-refinements-for-mongodb-3-2](https://www.mongodb.com/presentations/replication-election-and-consensus-algorithm-refinements-for-mongodb-3-2))
3. SOF: [Factors and Conditions that Affect Election in mongodb](https://stackoverflow.com/questions/28353164/factors-and-conditions-that-affect-election-in-mongodb)
4. SOF: [Voting in MongoDB](https://stackoverflow.com/questions/15597372/voting-in-mongodb)

