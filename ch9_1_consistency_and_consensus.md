## Chapter 9: Consistency and Consensus
- "It is better to be alive and wrong or right and dead": `CAP theorem`
- `algorithm` provides particular `guarantees` under particular `assumption`: `system model`
- The best way of building fault-tolerant systems (same approach as we used with `transaction`)
  - implement general-purpose `abstractions` with useful `guarantees`
  - let applications rely on those `guarantees`
- most important `abstractions` for distributed system is `consensus`
  - fail-over & elect a new leader on database with single-leader replication (preventing `split brain`)


### Consistency Guarantees
- ex) timing issues with replication lag
- `eventual consistency`
  - eventually converge to the same value (=`convergence`)
  - `weak guarantee`: doesn't say about `when`
- `stronger` consistency models
  - they don't come for free
  - appealing because they are easier to use correctly.
  - `strongest consistency models` = `linearizability`
- similarities between `distributed consistency models` and `hierarchy of transaction isolation levels`
  - `transaction isolation`: avoiding race conditions due to `concurrently executing transactions`
  - `distributed consistency`: `coordinating the state of replicas` in the face of delays and faults

### Linearizability
- `linearizability` (=`atomic consistency`=`strong consistency`=`immediate consistency`=`external consistency`)
- make a system appear `as if there were only one copy of data(~only one replica)`, and all operations on it are `atomic`
- `linearizability` is a `recency guarantee`: guaranteeing that the value read is the most recent, up-to-date value
- with this guarantee, even though there may be multiple replicas in reality, the application does not need to worry about them.

#### What Makes a System Linearizable?
- `regular register`
  - read may return either the old or the new value if they are `concurrent` with a write
  - not yet sufficient to linearizability: readers could see a value flip back and forth between the old and the new value several times while a `concurrent` write is going on.
- after any one read has returned the new value, all following reads (on the same or other clients) must also return the new value (even if the write operation has not yet completed).
- recency guarantee & always move forward in time!

#### Linearizability versus Serializability
- `Serializability`
  - isolation property of transaction(every transaction may read and write multiple objects)
  - guarantee that transaction behave the same `as if they had executed in some serial order` (each transaction running to completion before the next transaction starts)
  - it is okay for that `serial order` to be different from the order in which transactions were actually run
- `Linearizability`
  - recency guarantee on read and write of a register(an individual object)
  - it doesn't group operations together into transactions
- combination of `serializabilty` and `linearizability`: `strict serializabilty` = `strong one-copy serializability(strong-1SR)`
  - serializability based on `two-phase locking` & `actual serial execution`

#### Relying on Linearizability
- Locking and Leader election
  - Coordination services (like Apache `ZooKeeper` and `etcd`)
    - are often used to implement `distributed locks` and `leader election`
    - use `consensus algorithms` to implement `linearizable operations` in a fault-tolerant way
  - fencing issues (-> `fencing token`)
  - libraries like `Apache Curator` help by providing higher-level recipes on top of `ZooKeeper`
  - `linearizable storage service` is the basic foundation for these `coordination` tasks.
  - `distributed locking` usage example: lock per disk page on distributed databases.
- Constraints and uniqueness guarantees
  - similar to a `lock` & similar to an `atomic compare-and-set`
  - these constraints all require there to be a `single up-to-date value that all nodes agree on`
- Cross-channel timing dependencies
  - the linearizability violation was only noticed because there was an `additional communication channel` in the system(ALice's voice to Bob's ears)
  - Image resizer examples
    - race conditions between two different communication channels(`file storage service` & `message queue`) between the web server and the resizer.
  - `Linearizability` is not the only way of avoiding this race condition, but it's the simplest.

#### Implementing Linearizable Systems
- "behave as though there is only a single copy of the data, and all operations on it are atomic"
- simplest answer: really only use a single copy of the data -> `NOT FAULT-TOLERANT`
- Linearizability by fault-tolerant replication methods
  - single-leader replication (potentially linearizable): ISR
  - consensus algorithms (linearizable): contain measures to prevent `split brain` and `stale replicas`, consensus algorithms can implement `linearizable storage`(like `ZooKeeper` & `etcd`) safely
  - multi-leader replication (NOT linearizable): conflicts are an artifact of the lack of a single copy of the data
  - leaderless replication (probably NOT linearizable): even with strict quorums, nonlinearizable behavior is possible

#### Linearizability and quorums
- when we have variable `network delays`, it is possible to have race conditions, even with `strict quorum`.
- it is possible to make dynamo-style quorums linearizable at the cost of reduced performance
  - reader must perform read repair synchronously (`synchronous read repair`)
  - writer must `read` the latest state of a quorum of nodes `before` sending its `writes`
  - (linearizable `compare-and-set` operation requires a `consensus` algorithm)

#### The Cost of Linearizability
- `network interruption` forcing a choice between `linearizability` and `availability`
- `CAP theorem`
  - `Consistency, Availability, Partition tolerance: pick 2 out of 3`
  - `either Consistent or Available when Partitioned`
- Linearizability and Performance(`Network delays`)
  - the reason for dropping `linearizability` is `performance`, not `fault tolerance`
  - ex) RAM on a modern multi-core CPU is not linearizable
    - every CPU core has its own momory cache(L1, L2) and store buffer.
    - momory access first goes to the cache by default, and any changes are `asynchronously` writeen out to main memory.
    - `severral copies of the data`: one in main momory, and perhpas several more in various caches

### Ordering Guarantees
- `linearizable register`: behaves as if there is only a single copy of the data, and that every operation apeears to take effect atomically at one point in time.
- `well-defined-order`
  - leader in `single-leader replication` determine the `order of writes` in the replication log-that is, the `order` in which followers apply those writes.
  - `serializability` is about ensuring that `transactions` behave as if they were executed in `some sequential order`
  - the use of `timestamps and clocks` in distributed system

#### Ordering and Causality
- `ordering` helps preserve `causality` (~`causl dependency`)
- these chains of causally dependent operations define the `causal order` in the system. (~`what happened before what`)
- `causally consistent`: system obeys the `ordering` imposed by `causality`
  - `snpashot isolation` provides causal consistency
- `causal order` is not a `total order`
  - ex) natural numbers are `totally ordered` / `mathematical sets` are `partially ordered` with incomparable pairs
  - Linearizability: have a `total order` of operations (no `concurrent` operations in a linearizable datastore)
  - Causality: define a `partial order` (incomparable if they are `concurrent`, `concurrency` would mean that the timeline branches and merges again)
- `linearizability` is stronger than `causal consistency`
  - `linearizability` implies `causality`, but is NOT the only way of preserving `causality`
  - the good news is that a `middle ground` is possible.
  - `causal consistency` is the strongest possible consistency model
- capturing causal dependencies
  - we need some way of describing the `knowledge` of a node in the system.
  - `version vectors`: track causal dependencies across the entire database
  - database needs to know which `version` of the data was read by the application for `causal ordering`
    - `version number` from the prior operation is passed back to the database on a `write`
    - conflict detection of `SSI(Serializable Snapshot Isolation)`

#### Sequence Number Ordering
- actually keeping track of all causal dependencies can become `impractical` ~ `overhead`
- BETTER WAY: use `sequence numbers` or `timestamps(physical clock or logical clock)` to `order` events!
- `Noncausal` sequence number generators
- `Lamport timestamps`
  - simply a pair of `(counter, node ID)`
- `Timestamp ordering` is not sufficient
  - ex) `RIGHT NOW`

#### Total Order Broadcast
- using total order broadcast
- implementing `linearizable storage` using `total order broadcast`
- implementing `total order broadcast` using `linearizable storage`