## Chapter 5: Replication

### 멀티 리더 레플리케이션(Multi-Leader Replication)
- 싱글 리더 레플리케이션의 단점: 모든 쓰기(write) 작업은 반드시 "단일" 리더를 거쳐야 하므로, 싱글 리더가 SPOF 가 될 수 있음
- 이러한 제약에 대한 대안으로서, 멀티 리더 레플리케이션 전략이 쓰일 수 있음
- multi-leader configuration = "master-master" replication = "active/active" replication
- 각 리더는, 리더인 동시에 다른 리더들에 대한 팔로어 역할도 함

#### 멀티-리더 레플리케이션의 유즈케이스
- Multi-datacenter operation
  - 데이터센터를 여러 곳에 분산으로 둘 경우, 기존의 싱글 리더 전략을 취할 시에 여러 데이터 센터 중에 한 곳에 리더를 두어야 해서, 모든 write 작업이 해당 데이터센터로 집중됨, 따라서 각 데이터 센터마다 별도의 리더를 두는 전략을 취할 수 있음
  - 각 데이터 센터 간의 싱크 이슈가 더해져, 복잡한 컨플릭트 해결(Conflict resolution)이 필요해지는 단점이 있음
  - 여러 이슈(autoincrementing keys, triggers, integrity constraints)가 발생 가능하여 멀티-리더 레플리케이션은 가급적 지양되는 편
- Clients with offline operation
  - 캘린더 앱이 예시
    - 캘린더 앱을 이용하는 각각의 디바이스는 온/오프라인에 상관없이 일정을 읽고/쓸수 있어야 함
    - 오프라인 상태에서의 업데이트 내역도, 해당 디바이스가 온라인이 되면 싱크되어야 함
  - 이 경우, 각각의 디바이스는 쓰기(write)가 가능한 리더 역할을 수행하는 로컬 데이스베이스를 가져야 함
  - 캘린더 앱이 설치된 모든 디바이스 사이에 비동기(asynchronous) 멀티 리더 레플리케이션 프로세스(싱크)가 필요함
  - 이때 replication lag은 인터넷 접근 주기에 따라 수시간에서 수일이 될 수도 있음
- Collaborative editing
  - Google Docs와 같은 Real-time collaborative editing application이 예시
  - lock과 transaction에 기반한 싱글 리더 전략으로 구현시 반응성이 좋지 않음
  - 각 사용자의 브라우저 또는 클라이언트 앱이 리더로 동작하되, 각 변화의 단위를 작게(single keystroke) 하고 lock을 사용하지 않으면서, conflict resolution을 수행하는 방식으로 구현됨
    
#### 쓰기 충돌의 해결
Although multi-leader replication has advantages, it also has a big downside: the same data may be concurrently modified in two different datacenters, and those write conflicts mut be resolved.

The biggest problem with multi-leader replication is that `write conflicts` can occur, which means that `conflict resolution` is required.

- Synchronous versus asynchronous conflict detection
- Conflict avoidance! 최대한 피하자!
- Converging toward a consistent state (Eventually Consistency)
  - Give each write a unique ID / LWW(last write wins) : 데드락을 회피하기 위해 락 사이의 global order 를 두는 것과 유사
  - Give each replica a unique ID
  - merge/concatenate values together
  - record conflicts and handle it on application code level (Custom conflict resolution logic)
- What is a conflict?

### 리더-리스 레플리케이션(Leaderless Replication)
- 말 그대로 리더라는 컨셉 없이, 모든 레플리카가 클라이언트로부터 직접 쓰기(write) 요청을 받아 처리 가능한 시스템
- RDB가 주름잡던 시대에는 빛을 보지 못하다가, 아마존이 in-house Dynamo 시스템에 도입하면서, Dynamo 스타일이라는 이름과 함께 빛을 봄
- leaderless replication architecture 예시: Cassandra, Riak, Voldemort
- 구현 형태
  - 클라이언트가 여러 레플리카에 직접 쓰기 요청을 하는 형태
  - Coordinator 노드가 존재하여, 이 Coordinator가 클라이언트를 대리하여 쓰기 작업을 요청하는 형태
    - (이 Coordinator는 리더 기반 아키텍처와 다르게, 쓰기 요청의 특정 순서를 강제하지 않음)

#### 노드가 다운되었을 때의 데이터베이스 쓰기 요청
- Leader 기반 아키텍처에서 리더 노드가 다운되면, 페일오버(failover)가 필요함
- Leaderless 아키텍처에서는 리더가 없으므로, 페일오버 자체가 필요 없음
  - `quorum write`: 대신 쓰기 요청을 여러 노드에 보내고, 정족수(quorum)에서 쓰기 요청이 성공(OK 응답)하면, 쓰기 성공 <in parallel>
  - `quorum read`: 읽기 역시 여러 노드에 보내고, 정족수(quorum) 읽기를 통해, 가장 많이 읽혀진 값을 가장 최신의(up-to-date) 값으로 확인 <in parallel>
    - 이때 `Version number`를 사용해서 어떤 값이 더 최신의 값인지 결정할 수도 있음
    - 이 과정에서 `read repair`를 적용하여, stale 값을 가진 레플리카의 레코드를 최신 값으로 업데이트 할 수 있음

##### Read repair and anti-antropy
replication scheme 은 모든 레플리카의 레코드 값들이 eventually consistency를 가질 수 있도록 보장해야 함
이를 위해, Dynamo-style(leaderless) 데이터스토어에는 다음 두 메커니즘들이 보통 사용됨

- `read repair`:
  - 클라이언트가 여러 노드에 읽기 요청 후 stale 응답을 감지하면, 클라이언트가 직접 stale 레코드를 가진 레플리카의 레코드 값을 갱신하는 방식
  - 이 방법은 각 레코드 값들이 빈번하게 읽히는 워크로드 케이스에 적합함
- `anti-entropy process`:
  - 레플리카 간의 데이터 차이를 지속적으로 모니터링하고 차이 확인 시 값을 복사하는 백그라운드 프로세스(= anti-entropy process)를 데이터스토어 차원에서 수행
  - 리더 기반 레플리케이션의 replication log 기반 방식과 달리, 쓰기(writes) 간의 특정 순서를 강제하지 않으며, 데이터가 새로운 값으로 복사되는데 긴 지연(delay)이 발생 가능

##### Quorums for reading and writing
Strict quorum = quorum reads and writes
`w + r > n`
- think of `w` and `r` as the minimum number of `votes` required for the write or read to be valid
- at least one of the `r` nodes we're reading from, must be up-to-date
- 일반적인 설정 전략
  - 노드 수 `n`: 홀수개
  - `w = r = (n + 1) / 2` (`w + r = n + 1`)
- 쓰기 요청이 드물고 읽기 요청만 빈번한 경우에 가능한 설정 전략
  - `n = w`
  - `r = 1` (`w + r = n + 1`)
  - 읽기 요청 응답이 빨라지지만, 클러스터를 구성하는 노드 중 단 하나의 fail만으로도 전체 데이터베이스의 쓰기 요청을 fail 시킬 수 있다는 단점 존재
- Normally, reads and writes are always sent to all n replicas in parallel. The parameters w and r determine how many nodes we wait for
- i.e., how many of the n nodes need to report success before we consider the read or write to be successful.
- There may be more than n nodes in the cluster, but any given value is stored only on n nodes. This allows the dataset to be partitioned, supporting datasets that are larger than you can fit on one node.

#### 정족수 일관성의 한계 (Limitations of Quorum Consistency)
- `w + r > n` : make `overlap` on `at least one node` ~ 비둘기 집의 원리
  - Often, `r` and `w` are chosen to be a majority(more than `n/2`) of nodes
    - because that ensure `w + r > n`
    - still tolerating up to n/2 node failures
  - But `quorums` anr not necessarily majorities!
    - it only matters that the set of nodes used by the read and write operations `overlap` in at least one node.
  - even with `w + r > n`, there are likely to be edge cases where stale values are returned.
    - `sloppy quorum` case
    - `concurrent writes` case (cf: `Write conflicts` on `Multi-leader Replication`)
      - safe solution: `merge the concurrent writes`
      - when using last-write-wins, writes can be lost due to clock skew
    - concurrent reads and writes: undetermined read
    - partially succeeded writes: undetermined read
    - node carraying a new value fails, and its data is restored from a replica carrying an old value: breaking the quorum condition.
    - unlucky with the `timing`
- `w + r <= n`
  - `quorum` condition is not satisfied
  - reads and wirtes will still be sent to `n` nodes, but a smaller number of successful responses is required for the operation to succeed.
    - more likely to read stale values
    - allows lower latency and higher availability
- Dynamo-style databases are generally optimized for use cases that can tolerate `eventual consistency`
- Stronger guarentees generally require `transactions` or `consensus`

##### Monitoring staleness
`Eventual consistency`는 애매한 용어이지만, `eventual`에 대한 정량화는 필요함
- 리더 기반 레플리케이션: 리더/팔로어들이 모두 같은 순서로 쓰기(write) 적용이 보장되므로, `replication log` 상의 위치 차이로, `replication lag`의 정량화 용이함
- 리더리스 레플리케이션: 쓰기(write) 적용에 강제되는 고정된 순서가 없음, 만약 anti-entropy 프로세스 없이 read-repair 만 적용시에는, 드물게 읽히는 레코드들의 값 업데이트 지연에는 제한이 없음

#### Sloppy Quorums and Hinted Handoff
In a large cluster (with significantly more than `n` nodes), it's likely that the client can connect to `some` database nodes during the network interruption, just not to the nodes that it needs to assemble a quorum for a paricular value.
- `Sloppy quorum`: writes and reads still require `w` and `r` successful reponses, but those may include nodes that are not among the designated n `home` nodes for a value
  - If you lock yourself out of your house, you may knock on the neighbors's door and ask whether you may stay on their couch temporarily.
- `Hinted handoff`: once the network interruption is fixed, any writes that one node temporarily accepted on behalf of another node are sent to the appropriate `home` nodes
  - Once you find the keys to your house again, your neighbor politely asks you to get off their couch and go home
- pros: increasing write availability, as long as `any w` nodes are available, the database can accept writes
- cons: even when `w + r > n`, you cannot be sure to read the latest value for a key.

##### Multi-datacenter operation
멀티-리더 레플리케이션 방식처럼 리더리스 레플리케이션 방식 역시 멀티-데이터센터 유즈케이스에 사용 가능
- `Cassandra` & `Voldemort` 사례
  - each write from a client is sent to `all` repliceas, regardless of datacenter
  - client usually only waits for ACK from a quorum of nodes within its local datacenter
  - higher-latency writes to other datacenters are often configured to happen `asynchronously`
- `Riak` 사례
  - `n` describes the number of replicas within one datacenter.
  - cross-datacenter replication between database clusters happens `asynchronously` in the background (similar to multi-leader replication)

#### Detecting Concurrent Writes
- 멀티-리더 레플리케이션 방식처럼 리더리스 레플리케이션 방식 역시 (`strict quorum`을 사용할지라도) `write conflict` 이슈를 고려해야함
- 이 Dynamo-style database 들에선 `write` 시점 뿐만 아니라, `read repair` 과정, `hinted handoff` 과정에서도 컨플릭트가 발생 가능
- 이슈: events may arrive in a different order at different nodes: `no well-defined ordering`

##### Last write wins (discarding concurrent writes)
- one approach for achieving eventual convergence, but at the cost of durability
- even though the writes don't have a natural ordering, we can force an arbitrary order on them.
- caching: lost writes are acceptable
- only safe way of using a database with LWW is to ensure that a key is only written once and thereafter treated as `immutable`, thus avoiding any concurrent updates to the same key
- only supported conflict resolution method in Cassandra
  - recommended way of using Cassandra: use a UUID as the key, hus giving each write operation a unique key

##### "Happens-before" relationship & Concurrency
- "Happens-before" : `causally dependency`
  - A `happens before` B, if B `knows` about A
- "Concurrency": two operations are `concurrent` if neither happens before the other
  - neither `knows` about the other

##### Capturing the happens-before relationship
- server can determine whether two operations are concurrent by looking at the `version numbers`
  - server maintains a `version number` for every key
  - when a client writes a key, it must include the `version number` from the prior read, and it must `merge` together all values.
    - the response from a `write` request can be like a `read`
    - if you make a write request without including a `version number`, it is `concurrent` with all other writes
  - when the server receives a write with a particular `version number`, it can overwrite all values with that `version number` or `below`
    - since it knows that they have been `merged` into the new value
    - but it must keep all values with a `higher version number` (because those values are `concurrent` with the incoming write)

##### Merging concurrently written values
- clients have to clean up afterward by `merging` the concurrently written values(=`siblings`).
- `Merging sibling` = conflict resolution in multi-leader replication
- a reasonable approach to `merging siblings`: just take the `UNION`
  - if you want to allow people to also `remove` things from their cars, ant not just `add` things, then taking the `union` of siblings may not yield the right result
    - removed item will `reappear` in the `union of the siblings`
    - To prevent this problem, system must leave a marker(=`tombstone`) with an appropriate `version number` to indicate that the item has been `removed` when `merging siblings`
- CRDT(Conflict-free replicated datatypes): family of data structures for sets, maps, ordered lists, counters, dtc.
  - that can be concurrently edited by multiple users, and which automatically resolve conflicts in sensible ways.
  - that can automatically merge siblings in sensible ways, including preserving deletions.

#### Version vectors
How does the algorithm change when there are `multiple replicas`, but `no` leader?
- We need to use a version number per replica, as well as per key
- `version vector(=vector clock)`: the collection of version numbers from all the replicas
  - `dotted version vector`: Riak encodes the `version vector` as a string that it calls `causal context`
- `version vector` structure ensures that it is afe to read from one replica and subsequently write back to another replica.
