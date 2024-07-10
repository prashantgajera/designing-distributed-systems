There are mainly three popular algorithms for the replication.  **Single leader**, **multi leader** and **leaderless.** almost all the distributed systems uses one of the mentioned approach.

## Leaders And Followers (Active/Passive)(Master/Slave)

This is the most common one of all of them.

Copy of a database in another system is called replica.

Contracts to work all together

- All the write must be going to leader and it will update its local copy first
- leader will provide updates to slaves or replicas and they will compare and update its local copy
- When client wants to read data it can do it from either slaves or master

### Synchronous vs Async Replication

Sync replication will hold the next write till the replica confirms the write.

Async will not wait, It will continue sending write request and replica get async requests to write.

There can be two cases 

- Data loss happens where we don’t care if replica is failing or fallen behind. Async write will happen via leader.
- In real world mix of the approach semi async where at least one replica is sync, so at a time two one leader and one replica will have all the data guaranteed.
- However many prefer async over this where we have so many replicas or geo distributed nodes.

### How to setup a new follower

It’s never recommended or followed to stop production write and setup new follower or manually copy files over a node and setup.

What we can do is follow below strategy to do it without hampering production

- At any point of time take snapshot of the leader without locking
- Setup up follower with given snapshot of data
- Now follower connect to leader and request for all the data after snapshot was taken. Data will be from the exact sequence of leader’s replication log. postgres call it replication log sequence while mysql calls it binlog number
- As follower catch up with the leader it starts processing most recent changes

### Handling Node outages

In real world systems any node can go down. How we can handle such scenarios in leader follower based replication without a outage?

Follower failure : recovery

- If follower goes down due to any kind of issue it can always compare seq number or check its last transactions logs and request leftover data from the leader to catch up for recovery.

Leader failure

- If leader fails or break down due to some reason, it is complex than follower to recover or handle. As controller must need to choose new leader as well as let followers know for requesting data from new leader.
    
    
    Following steps can be followed if leader fails
    
    - Identifying leader failure
        - Many system uses timeout to identify
    - Choosing a leader
        - Can be done by prev leader or by voting among followers considering follower having most up to date data.
    - Configuring system to use new leader

Some problems to be solved in case of leader failure

- If async replication user there can be inconsistency in leaders and followers. what if old leader is come back again. new leader having less data than old leader
- Discarding write from old leader in case of above problem can cause problem such as faced by github when new leader was out date and was having old primary keys and redis being used with those keys made system inconsistent.
- Split brain problem where both old and new are receiving writes.
- Not using correct amount of timeout period. Less would cause more failovers. More would make system slow in case of recoveries.