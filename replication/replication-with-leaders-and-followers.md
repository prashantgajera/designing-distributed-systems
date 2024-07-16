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

### Different Ways to Implement Replication Logs

- Statement based
    - Simplest case where leader sends all the executed statements(Inserts, update or delete) in correct order to it’s followers. But this kind of replication has it’s drawbacks and to overcome that need to take care of below scenarios
        - Non deterministic statements such as date and now and rand may generate different results if send as it is(can be attached values from leader in place)
        - If auto-incremented columns depended on other data then must be executed in the same order in replicas
        - Un-deterministic statements such as procedures, triggers can have side effects in data on replicas
- WAL (write-ahead-logs)
    - In this type every write is appended to a log file.
    - If sstable or lsm trees based engine used then its a primary storage and segments are garbage collected.
    - If B-tree based engine used then write ahead log will have modification first written then reflected to storage blocks.
    
    Problem with this is, it is tightly coupled with the storage engine as raw byte data stored. Upgrading or changing things in followers will be tricky as we may need downtime for that.
    
- Logical (raw) replication
    - This used different kind of log format one which is not tightly coupled with the storage engine which allows data to be independent of version change effect of storage engine or database upgrade.
        - Insert log will contain all the column values logs needs to be inserted.
        - For delete statement log contains information to uniquely identify and delete the rows.
        - In case of update it contains enough info to update rows. if multiple updates then log entries will contains all the row logs needs to be updated.
- Trigger based
    - Usually replication logic is written and implemented on database side. but sometimes there is a need of some custom logic to replicate data. such as subset of data or perform any after effect of data.
    - One can use some external tools or use internal features like triggers or procedures as provided in relational databases. it lets you register custom application logic for database operations which can directly read internal logs.
    

### Problems With Replication Lag

- Master-slave replication is only suitable with async replication because when sync replication is used then as number of followers increases chances of failures and lagging increases.
- With async replication as well it introduces state of inconsistency with master very often as it is async. If writes are stopped for some period then eventually followers will catch up in terms of replication this effect is called eventual consistency.
- In normal conditions eventual consistency can have ms or seconds lag but if network slowness or other problem arises then it can affect applications in large effect.

### Read your own writes (read-after-write ) consistency

When application lets user read data right after they submit, in master slave setup usually it goes to master for write but when reading, follower will be read the data.

In Asynchronous replication, problem is when user tries to see data just right after write data may not have reached the replica and user will think data submitted lost but thats not the case. We can improve this by following some of the strategies as below.

- If we want to read something that user might have modified, read it from the leader all the other reads should be from the follower. For this we will need some way to distinguish this kind of reads and write at application level.
- If most of the things are getting read for user from the master then above solution will not work. We could track the last update time and for next x amount of time read from the master after the data got modified.
- Another approach can be track of last update timestamp of given replica and while reading you are reading most recent data, else read from other replica or wait for x amount of time to get it reflected.