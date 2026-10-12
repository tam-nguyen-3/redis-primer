# Redis 

## Pre-class Readings
- [Redis in 100 Seconds by Fireship](https://www.youtube.com/watch?v=G1rOthIU-uo): a very high-level overview of Redis
- This page! Sections marked *(optional deep dive)* are for the curious; skip them if you're short on time.
- [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) by Martin Kleppmann (see [Further readings](#further-readings) at the bottom)

### What is Redis?
Redis (REmote DIctionary Server) is an "in-memory data structure store", which stores all data in memory (aka RAM, not disks) as key/value pairs. You can think of it as an extremely fast dictionary/ hashmap. 

Some properties of Redis:
- Fast, low latency: again, because everything is in memory, so reads and writes skip the disk
- Stores data structures, not just strings. 
- Single-threaded design: it processes commands sequentially, one at a time.

#### Use cases
Redis has many use cases! The most common one is as a cache. Other use cases are:
- Cache: the real data lives in the database, Redis just keeps a copy with some expiration time.
- As a home for the data: Rate limiters, login sessions, leaderboards, counters, etc. The data (e.g., tokens, counters) live in Redis but not in another database.
 
If you're interested in other use cases, read the examples in [Redis.io](https://redis.io/docs/latest/develop/use-cases/#available-use-cases)!

#### Try it out
1. Using Docker: by default, Redis is on port 6379. Running this will create a container named `redis` and open up an interactive Redis CLI inside it:
```
docker run --name redis -p 6379:6379 -d redis   # creating the container
docker exec -it redis redis-cli                 # opens an interactive Redis CLI
```
2. Installing it: You can do `brew install redis` (on Mac), or use WSL for Windows ([guide](https://redis.io/docs/latest/operate/oss_and_stack/install/archive/install-redis/install-redis-on-windows/#run-redis-on-windows-using-wsl-windows-subsystem-for-linux)). Other ways to install [here](https://redis.io/docs/latest/operate/oss_and_stack/install/archive/install-redis/).

#### Example: Redis in Modern AI Systems

Adapted from [UMD's CMSC 389A](https://www.cs.umd.edu/class/fall2026/cmsc389A/Redis.html)

```
                   User
                     │
                     ▼
                Application 
                     │
      ┌──────────────┼──────────────┐
      ▼              ▼              ▼
Redis Cache     Job Queue      Vector Database
(LLM answers)   (async         (semantic search over
(rate limits)    document       task descriptions,
(sessions)       processing)    for retrieval)
      │              │              │
      └──────────────┼──────────────┘
                     ▼
                    LLM
```

I hope the above helps you understand why we use Redis and some of its use cases better. 

For each use case, we can use one Redis node, but for high availability and later, horizontal scalability, we can use multiple Redis servers.

Now, let's get to the "distributed" part of Redis...

## Transactions (within one node)
A Redis transaction is a group of commands that modify values under a single step (`MULTI` ... `EXEC`). It is **not** a SQL transaction, and doesn't have the ACID properties, so there is no rollback if a command fails.

All commands are **serialized** (I understand it as being sequential) -- running many at the same time gives the same result as running them one by one. 

For example, suppose `page_viewed` starts at 0 and we have 100 clients, each running `INCR page_viewed` at the same moment. Redis guarantees that `page_viewed` ends at exactly 100, because the read, the +1, and the write all happen inside Redis as one command, and nothing can run in between.

**But careful:** this only works if the increment happens *inside* Redis. If each client does `GET page_viewed`, adds 1 in its own code, and then `SET`s the result, two clients can both read 0 and both write 1, so we end up with `page_viewed=1` instead of 2 (a "lost update"). Redis can't protect us here, since the +1 happens outside it. To fix it, use `INCR`, or wrap the read and write in `MULTI`/`EXEC` with `WATCH`. For more details on why, see these [docs on atomic operations](https://redis.io/docs/latest/develop/clients/patterns/twitter-clone/#atomic-operations).

## Persistence: durability of data
Since data lives in memory, a crash wipes it unless Redis also saves to disk. Two options ([docs](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)):
- RDB (Redis Database): point-in-time snapshots of the whole dataset every so often. 
  - Pros: fast, simple. 
  - Cons: if the server crashes, you lose everything since the last snapshot.
- AOF (append-only file): essentially a **write-ahead log** of every write command (we learned this in CS162! -- [WAL](https://www.postgresql.org/docs/current/wal-intro.html)). You choose how often it hits the disk: every write (safe, slow), every second (the default, can lose ~1 second), or let the OS decide.
  - Pros: data is much more durable
  - Cons: heavier files compared to RDB and can be slower

Note: Redis says "OK" to the client before the write is safely on disk (under the default), so acknowledged ≠ durable

## Replication

Time to add Availability to our system! I encourage you to read the [docs](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/) (until Configuration).

Tl;dr: Redis replication has a leader - follower (single-leader) mechanism that we have studied so far. And Redis uses **asynchronous** replication, which means the master acknowledges a write *before* the replicas confirm it. 

This has two consequences:
- Replicas can lag behind, so reading from a replica can return stale data.
- If the master crashes after saying "OK" but before sending the write to any replica, that write is gone.

*(Optional deep dive)* The main mechanism for how replicas stay up-to-date is:
- Each master has a `replication ID` and an `offset` that goes up for every byte it sends to replicas. The pair `(replication ID, offset)` identifies exactly the version of the dataset of a master.
- When replicas connect to master, it uses the command `PSYNC` (partial sync) sending its `(replication ID, offset)` to master.
  - Does the `replication ID` match? If not, the replica gets a full sync (see below). If yes,
  - Is the `offset` too far behind? 
    - If not, then the replica is just missing a few commands, so the master sends only that part. 
    - Else, it means there is not enough *backlog* in the master's buffer. The master will send a full copy of the dataset and overwrite all data currently in the replica.

This works well when the followers crash temporarily or there is a brief disconnection! But what about when the master Redis node crashes?

## Sentinel

Replicas have the data, but nothing tells them to take over. Sentinel is a **program**, a separate process that runs alongside Redis. Usually 3 copies (3 Sentinels) on different machines ([docs](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)).

How it works:
1. Each Sentinel keeps pinging the master to check that it's alive.
2. If one Sentinel doesn't hear back, it asks the other Sentinels (maybe it's just its network that's broken).
3. If enough Sentinels agree (e.g. 2 out of 3), the master is declared dead.
4. One Sentinel leads the failover and promotes the most up-to-date replica to be the new master.

Where it breaks (split-brain problem): if the old master isn't dead but it's just cut off from the network, some clients can keep writing to it while a new master is promoted, resulting in us having two masters. When the network heals, the old master becomes a replica and throws away the writes it accepted, even though clients were told "OK". See [Jepsen: Redis](https://aphyr.com/posts/283-jepsen-redis) (a great resource!).

## Redis Cluster: High availability + horizontal scaling

When one master can't hold all the data, Redis Cluster splits the keys across several masters, each with its own replicas (basically adding sharding). This is helpful since replication doesn't help when we have a lot of data. ([docs](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/))

### Sharding
Redis Cluster partitions all possible keys into 16,384 **hash slots**. To find a key's slot, Redis runs the key name through a hash function `(CRC16(key) mod 16384)` (*optional deep dive:* more on the CRC16 algorithm [here](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/#key-distribution-model)). 

Each master is in charge of a range of slots. With 3 masters, it might look like this:

```
master A          master B          master C
slots 0–5460    slots 5461–10922  slots 10923–16383
  |                 |                 |
replica A         replica B         replica C
```

For example, for a key `user:123`, Redis computes `slot = CRC16(user:123) mod 16384`, and the `slot` determines which master owns the key.

Note: This is not consistent hashing! In consistent hashing, keys and nodes are placed on a ring, and a key goes to the next node clockwise, so a key's node follows automatically from where the nodes sit. In Redis, `hash_function(key)` gives us the `slot`, and a separate table says which node owns each slot. That table can be changed freely, so slots can be moved or allocated to different nodes. (In Kleppmann's book, this is the "fixed number of partitions" approach.)

Therefore, when we add a new master, some slots are handed over to it, and only the keys in those slots have to move. It makes resharding easier.

For example, with the three masters A, B, and C above, after adding a new master D: (this is because key -> slot -> node)
```
A: fewer slots
B: fewer slots
C: fewer slots
D: newly transferred slots
```

### Hash tags -- for multiple keys

Some commands or transactions naturally require us to use several keys, like `user:123:profile` and `user:123:posts`. If we always access a user's profile and posts independently, maybe it's fine if they live in different nodes (e.g. `user:123:profile` → slot 3921 → Node A, `user:123:posts` → slot 9412 → Node C). 

But in many cases, like when we need to render (and thus, cache) the profile page, we need `user:123:profile` and `user:123:posts` together (thru a command like `MGET`). **A command or transaction that touches several keys only works if all of those keys are in the same slot.**

Therefore, Redis uses hash tags -- if part of a key name is in curly braces, only that part is used to pick the slot. So `{user:123}:profile` and `{user:123}:posts` always end up in the same slot.

The downside: if we put too much data under one tag, that one slot (and its node) gets overloaded, creating a "hot spot".

### What happens when a master fails?

(Note: Redis Cluster doesn't use Sentinel to detect failures!)

Tl;dr: All nodes are connected to each other (every node is connected to every other node). Thus, if we have N nodes, every node has N-1 outgoing and N-1 incoming connections. Nodes use a **gossip protocol** (sounds familiar? :D) to make sure all other nodes are working properly (by continuously exchanging heartbeat packets) -- read [here](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/#heartbeat-and-gossip-messages) for full details!

If a master dies, one of its replicas asks the other masters to vote for it. If a majority agrees, that replica becomes the new master. Every change like this gets a number that only goes up (an "epoch"), and if there's ever a conflict, the higher number wins. If you remember Raft terms, it's the same idea. Because the cluster handles this on its own, you don't need Sentinel.

## Consistency

So, where does Redis land? Short answer: Redis chooses speed over strong guarantees.

- Replication is still asynchronous, even in Redis Cluster. So everything from the Replication section still applies: stale reads from replicas, and lost writes if a master crashes before passing them on.
- Split-brain can still happen in Redis Cluster. If a master gets cut off from most of the cluster, it keeps accepting writes for a while, until a timeout (`cluster-node-timeout`) runs out. Meanwhile, the rest of the cluster promotes one of its replicas. When things reconnect, the writes the old master accepted during that window are lost. Same problem as with Sentinel, but the window is limited by that timeout.
- What about the `WAIT` command? It lets a client wait until N replicas have received its write. This makes losing data less likely, but it doesn't guarantee anything: the write was already applied on the master, and a failover could still promote a replica that didn't get it. It's not consensus!
- So Redis is **not linearizable** once replicas are involved. Redis votes on *who* becomes master (like Raft), but it doesn't use consensus for the data itself.

That's why Redis is great for things like caches, sessions, and rate limiters, where losing a little data now and then is okay. But it's rarely used as the main "source of truth" database.

Something to think about for class: why do you think Redis uses voting to pick a new master, but not to store the data?

## Further readings

Sometimes many servers must not do the same thing at once (e.g. edit the same file). People often use Redis as a distributed lock: set a key like `lock:file42` that auto-expires after, say, 10 seconds, so a crashed holder doesn't block everyone forever.

Redlock is antirez's (Redis's creator) sturdier version: try to grab the lock on 5 independent Redis servers, and you only hold it if you got at least 3, quickly enough.

Kleppmann says it isn't safe when correctness matters: a lock holder can freeze (e.g. a GC pause) past the expiry, wake up and keep writing while someone else holds the lock. His fix: fencing tokens.

antirez disagrees: he argues the timing assumptions are reasonable in practice, and the pause problem hits almost any lock with an expiry, not just Redlock.

Readings:
- [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) by Martin Kleppmann (**required**)
- [Is Redlock safe?](http://antirez.com/news/101) by antirez (optional, his full reply)

Come ready to argue your assigned side: who's right, and when does it matter?
