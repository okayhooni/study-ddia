## Chapter 8: The Trouble with Distributed Systems
- "Pessimism to the maximum": Anything that can go wrong will go wrong.

### Faults apnd Partial Failures
- Single computer
  - `deterministic` behavior, `always-correct` computation (deliberate choice in design)
  - individual computer with good software is usually either fully functional or entirely broken("blue screen of death"), but not something in between.
  - a problem that is often fixed by a reboot
- Distributed system
  - `partial failures` are `nondeterministic` 
  - you may not even know whether something succeeded or not, as the time it takes for a message to travel across a network is also nondeterministic!

#### Cloud Computing ans SUpercomputing
- Supercomputing
  - `HPC` (high-performance computing)
  - job typically `checkpoints` the state of its computation to durable storage from time to time.
  - supercomputer is more like a `single-node` computer
  - `offline` (batch) jobs like weather simulations
  - specialized hardware, where each node is quite reliable, and nodes communitcate through shared memory and remote direct momory access(RDMA).
- Cloud computing
  - multi-tenant datacenters, commodity computers connected with an IP network(often Ethernet), elatic/on-demand resource allocation.
  - systems for implementing `internet services` (`online`)
  - it is reasonable to asume that `something is always broken`
  - `rolling upgrade`
  - `building a reliable system from unreliable components`
- Traditional enterprise datacenters: lie somewhere between these extremes.

### Unreliable Networks
- `shared-nothing` systems with `scale-out`, a buch of machines connected by a network.
- the internet and most internal networks in datacenters(often Ethernet) are `asynchronous packet networks`
- If you send a request and don't get a reponse it's not possible to distinguish whether
  - the request was lost
  - the remote node is down
  - the response is lost
- usual way os handling issus on `asynchronous network` is a `timeout`

#### Network faults in practice

#### Detecting Faults

#### Timeouts and Unbounded Delays
- fictitious system with a network that guaranteed a maximum delay for packets: `2d + r` would be a reasonable timeout to use
- `asynchronous networks` have `unbounded delays`

##### Network congestion and queueing
- network congestion: network switch must queue them up, if there is so much incoming data that the switch queue fills up, the packet is dropped, so it needs to be resent.
- CPU core scheduling: if all CPU cores are currently busy, the incoming request from the network is queued by the operating system.
- Hypervisor in virtualized enviroment: incoming data is queued(buffered) by the virtual machine monitor
- TCP flow control(=`congestion avoidance` = `backpressure`): additional queueing at the sender
- `TCP` vs `UDP`
- trade-off between failure detection delay and risk of premature timeouts.
- (rathre than using configured constant timeouts) systems can continually measure response times and their variability(jitter), and automatically adjust timeouts according to the `observed response time distribution`
  - Phi Accrual failure detector is used for example in `Akka` and `Cassandra`
  - TCP retransmission timeouts also work similarly

#### Synchronous versus Asnychronous Networks
- traditional fixed-line telephone network: `synchronous` & `bounded delay`
- Latency and Resource Utilization trade-off

### Unreliable CLocks
- durations & points in time
- synchronized clocks to some degree with `Network Time Protocol(NTP)`
#### Monotonic versus Time-of-Day Clocks
- time-of-day clock
  - `clock_gettime(CLOCK_REALTIME)` on Linux
  - wall-clock time
  - time-of-day clocks are usually synchronized with NTP
  - if the local clock is too far ahed of the NTP server, it may be forcibly reset and appear to jump back to a previous point in time.
  - thsese jumps, as well as the fact that they often ignore leap seconds, make time-of-day clocks unsuitable for measuring elpased time
- monotonic clock
  - `clock_gettime(CLOCK_MONOTONIC)` on Linux
  - guaranteed to always move forward!
  - NTP may adjust the frequency at which the monotonic clock moves forward(`slewing the clock`)
#### Clock synchronization and Accuracy
#### Relying on Synchronized Clocks
#### Process Pauses

### Knowledge, Truth, and Lies
#### The Truth is defined by the majority
#### Byzantine Faults
#### System Model and Reality
