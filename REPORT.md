# Lab Report: CAP Theorem & Hazelcast

## 1. Answers to scenario questions (3p)

### Exercise 1: Single node isolation (CP - AtomicLong)
In this scenario, the cluster operated in CP mode (prioritizing consistency). The system used the Raft algorithm, which requires a quorum (a majority of votes, here 2 out of 3 nodes) to execute any operation.
* **Majority side (Nodes 1 and 2):** We were able to read and increment the `AtomicLong` variable without any issues because these nodes could communicate with each other and maintained the quorum necessary to commit operations in the Raft log.
* **Minority side (isolated Node 3):** Reading and writing were impossible (returned a `timeout - no quorum` error). This node lost contact with the leader and could not win a new election on its own. In a CP system, when facing a network partition (P), the node chooses to sacrifice availability (A) by refusing to provide any value. This prevents breaking strong consistency (C) guarantees and returning stale data. After the network partition was healed, Node 3 synchronized its logs with the leader and safely updated its value.
<img width="1810" height="2064" alt="image" src="https://github.com/user-attachments/assets/9f9bc5aa-68bf-4212-a53c-07917283ab12" />



### Exercise 2: Complete node isolation (CP - AtomicLong)
After all three nodes were isolated from each other, none of them could form the required majority (2 out of 3).
* As a result, we were **unable to read or increment** the counter value on **any of the nodes**. Every attempt resulted in a `timeout - no quorum` error.
* The system completely halted operations for clients, thereby preventing a "split-brain" scenario (where each node would diverge with its own independent value). Once the network was restored, the nodes first had to go through a new leader election and log synchronization process, which took some additional time before the cluster became fully responsive again.
<img width="3456" height="2026" alt="image" src="https://github.com/user-attachments/assets/08538cb6-814d-4b12-bc52-f19f903a7658" />


### Exercise 3: Single node isolation (AP - PNCounter)
In this scenario, we used a `PNCounter` data structure based on CRDT, which follows AP guarantees, providing only Eventual Consistency.
* **During the network partition:** We were able to read and increment the counter value **on all nodes**, regardless of whether they were on the majority side or the isolated Node 3. The system prioritized Availability (A) over Consistency (C).
* **Why this happened:** CRDT (Conflict-free Replicated Data Type) structures are specifically designed to tolerate conflicts in distributed systems. Instead of blocking writes, they allow states to diverge. After communication was restored (`heal`), the built-in mathematical properties of the structure allowed it to seamlessly merge the local increment operations without conflicts. As a result, all nodes eventually reached an agreement and returned the correct, aggregated total value (3).
<img width="3456" height="1792" alt="image" src="https://github.com/user-attachments/assets/02370664-fb34-4da5-9326-3266e1060e6e" />

---

## 2. The role of the Raft algorithm (2p)

**Which data structures require Raft?**
The Raft algorithm is required by **CP** (Consistency & Partition tolerance) data structures, such as the `IAtomicLong` used within the `CPSubsystem` instance in Hazelcast. AP structures do not rely on Raft.

**Why is this algorithm needed?**
Raft is a consensus algorithm for distributed systems. It is essential to ensure that the cluster behaves as a single, strongly consistent state machine (linearizability). Raft is responsible for leader election, replicating operation logs to follower nodes, and ensuring that a system state change can only occur if it is committed by a majority of nodes (quorum). This effectively prevents data corruption and inconsistencies during network failures.

**The role of Raft after a partition heal:**
When the network is restored, Raft handles the automatic reconciliation of the cluster. Previously isolated nodes recognize the current leader (or new elections are triggered based on the latest logs and higher election terms). The leader then sends the missing operation logs to the followers, overwriting any uncommitted or conflicting changes and bringing the reconnected nodes back to strict consistency with the rest of the system.

---

## 3. Session Guarantees: RYW and Monotonic Reads (2p)

According to concepts presented in distributed systems like Bayou, enforcing full, strong consistency introduces a massive performance overhead. Conversely, weak (eventual) consistency often leads to user confusion. Session Guarantees offer a practical middle ground.

* **Read-Your-Writes (RYW):** This guarantees that if a client performs a write operation (e.g., incrementing the `PNCounter`) during their session, any subsequent read attempt by *that same* client will reflect that write. The user will never see the system in a state prior to their own modification.
* **Monotonic Reads:** This guarantees that if a client reads a variable at a specific state `X` during their session, no subsequent read by that client will return a state older than `X`. Time will never "move backward" from the client's perspective, even if a load balancer routes their request to an older, lagging replica.

**Why are they called "session guarantees"?**
Because these guarantees are enforced and maintained strictly from the perspective of a single client's process (session), rather than globally across the entire system. The system as a whole might be inconsistent and return different values to different users connected to different nodes (Eventual Consistency). However, for an individual user—within the lifespan of their specific session—the history of operations remains logical, monotonic, and predictable. This approach allows the system to achieve high scalability and availability (AP) without degrading the User Experience for a specific client.
