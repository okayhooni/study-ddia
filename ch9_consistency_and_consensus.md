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
  - you also need to know `when that order is finalized`

#### Total Order Broadcast
- Knowing when your `total order is finalized` is captured in the topic of `total order broadcast`(=`atomic broadcast`)
- `Total order broadcast` is usually described ad a `protocol` for exchanging messages between nodes
- required safety properties
  - reliable delivery
  - totally ordered delivery
- using total order broadcast
  - `Consensus services` such as `ZooKeeper` and `etcd` actually implement `total order broadcast`
    - strong connection between `total order broadcast` and `consensus`
  - `total order broadcast` is exactly what you need for `database replication`
    - `state macine replication`: if every `message` represents a `write` to the database, and every replica processes the same `writes` in the same `order`, then the replicas will remain consistent with each other.
  - `total order broadcast` can be used to implement `serializable` transactions
    - if every `message` represents a `deterministic` transaction to be executed as a `stored procedure`, and if every node processes those `messages` in the same `order`, then the `partitions` and `replicas` of the database are kept consistent with each other.
  - the `ORDER` is `FIXED` at the time `the messages are delivered` -> this fact makes `total order broadcast` STRONGER than `timestamp ordering`
  - another way of looking at `total order broadcast` is that it is a way of creating a `LOG`! (as in a `replication log`, `transaction log`, or `write-ahead log`)
    - `delivering a message` is like `APPENDING to the LOG`
    - all nodes must `deliver` the same `message` in the same `order`
  - `total order broadcast` is also useful for implementing a `lock` service that provides `fencing tokens`
    - `every request` to acquire the `lock` is `appended` as a message to the log.
    - all messages are sequentially numbered in the `order` they appear in the log.
    - the `sequence number` can then serve as a `fencing token`, because it is `monotonically increasing` (In `ZooKeeper`, this `sequence number` is called `zxid`)
- implementing `linearizable storage` using `total order broadcast`
  - `total order broadcast` is `asynchronous`
    - messages are guarantedd to be delivered reliably in a `fixed order`
    - but there is `no` guarantee about `WHEN` a message will be delivered (so one recipient may `lag` behind the others)
    - By contrast, `linearizability` is a `recency` guarantee
  - If you have `total order broadcast`, you can build `linearizable storage` on top of it!
    - you can implement such a `linearizable compare-and-set operation` by using `total order broadcast` as an `APPEND-ONLY log`
    - choosing the `first` of the conflicting writes as the `winner` and `aborting later ones` ensures that `all nodes agree` on whether a write was committed or aborted
    - similar approach can be used to implement `serializable` multi-object transactions on top of a `LOG`
  - while this procedure ensures `linearizable writes`, it doesn't guarantee `linearizable reads` (=`sequential consistency`=`timeline consistency` : a slightly weaker guarantee than `linearizability)
    - if you read from a store that is `asynchronously` updated from the `log`, it may be stale.
  - to make `reads linearizable`..
    - `Quorum reads` in `etcd`
      - you can sequence `reads` through the `log` by `APPENDING` a message, reading the `log`, and performing the actual read when the message is delivered `back to you`.
      - the `message's position in the log` thus defines the `point in time` at which the read happens
    - `ZooKeeper's sync()`
      - if the `log` allows you to fetch the `position` of the `latest log message` in a `linearizable` way
      - you can query that `position`, `wait` for all entries up to that `position` to be delivered to you, and then perform the read
    - chain replication (~ `Microsoft Azure Storage`)
      - make your read from `ISR(a replica that is synchronously updated on writes)`, and is thus sure to be up to date.
- implementing `total order broadcast` using `linearizable storage` (TURN IT AROUND!)
  - for every message you want to send through `total order broadcast`, you perform `atomic` `increment-and-get` or `compare-and-set` the `linearizable` integer
  - then attach the value you got from the register as a `sequence number` to the message
  - you can then send the message to all nodes, and the recipients will deliver the messages consecutively by `sequence number`
  - Key difference between `total order broadcast` and `timestamp ordering`
    - unlike `Lamport timestamps`, the numbers you get from incrementing the `linearizable register` form a sequence with `no gaps`
    - ref) the `ORDER` is `FIXED` at the time `the messages are delivered` -> this fact makes `total order broadcast` STRONGER than `timestamp ordering`
- `LINEARIZABLE SEQUENCE NUMBER GENERATORS`: you inevitably end up with a `consensus` algorithm
  - `consensus` is equivalent to...
    - `linearizable compare-and-set` / `increment-and-get` register
    - `total order broadcast`

### Distributed Transactions and Consensus
- `consensus`: get several nodes to agree on something
- use cases
  - leader election
  - atomic commit

#### Atomic Commit and Two-Phase Commit (2PC)
- From single-node to distributed atomic commit
  - transaction commit must be `irrevocable`
    - once data has been committed, it becomes visible to other transactions
    - this principle forms the basis of `read committed` isolation
- Introduction to 2-phase commit
  - commit/abort process in 2PC is split into two phases(prepare phase / commit phase)
  - 2PC use a `coordinator`(= `transaction manager`)
- A system of promises
  - globally unique `transaction ID` from the `coordinator`
  - `commit point`
    - when the `coodinator` has received responses to `all` prepare requests, it makes a definitive `decision` on whether to commit or abort the transaction.
    - `coordinator` must write that decision to its `transaction log on disk`
  - two crucial `points of no return`
    - when a participant votes "yes", it promises that it will definitely be able to commit later
    - once the coodinator decides, that decision is irrevocable
- Coordinator failure

#### Distributed transcations in practice