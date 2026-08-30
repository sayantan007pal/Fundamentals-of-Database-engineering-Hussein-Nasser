# Eventual Consistency

> **Course:** Fundamentals of Database Engineering — Sec 2: ACID
> **Prerequisite:** [11.Consistency.md](11.Consistency.md) already split data-C vs read-C on **one node**. **This note:** after COMMIT, will **another node** see it?
> **Lecture slides:** [ACID-Updated.pdf](ACID-Updated.pdf) pp. 25–31
> **Typed in PostgreSQL, InnoDB, and MongoDB.**

---

## 0. After this note you can...

- Spot Hussein’s two **data** bugs (orphan like vs lying counter) and refuse to call them replica lag.
- Draw write-node vs read-node: COMMIT returned, replica still old, every row on the replica still valid.
- Draw Alice debited on shard 1, Bob not yet visible on shard 2.
- Name the knob that closes the lag window on each engine.

---

## 1. The one picture

The lecture question:

> If a transaction committed a change, will a new transaction immediately see the change?

| Where you read | Immediate? | Name |
|----------------|------------|------|
| Same node that committed | Yes (isolation still applies) | Strong on that node |
| Another node (standby, replica, secondary) | Not necessarily | **Eventual consistency** |

```text
HQ catalog (write node)     stamps "Jon likes picture 1"
Campus kiosk (read node)    truck of photocopies has not arrived
                            every letter on the kiosk page is valid
                            the page is yesterday
```

Eventual consistency is **not** a replacement for foreign keys. Copies **converge** to the committed state if no newer writes arrive. Until then, a read node can return **stale but internally valid** data.

---

## 2. Simulation — watch it happen

### Simulation 1 — pictures / likes (data bugs, not lag)

Valid start ([ACID-Updated.pdf](ACID-Updated.pdf)):

```text
pictures                    picture_likes
id  blob  likes              user     picture_id
1   xx    2                 Jon      1
2   xx    1                 Edmond   1
                            Jon      2
```

Rules: every `picture_id` exists (FK). `pictures.likes` equals the count of like-rows (denormalized **user** invariant).

Broken slide:

```text
pictures                    picture_likes
1   xx    5                 Edmond   4     ← no picture 4
2   xx    1                 ...
```

**Pause and predict:** Which bug is eventual consistency? Neither.

| Bug | What is wrong | Who should have stopped it |
|-----|----------------|----------------------------|
| **A. Broken FK** | Edmond likes picture 4 | `FOREIGN KEY` on the **write node** |
| **B. Broken counter** | likes=5 but two rows | App + one transaction (or a trigger). Not an FK. |

Replicas will **faithfully copy** both lies if the primary accepted them. Eventual consistency is a **third** crime: a kiosk that still shows yesterday’s **valid** catalog.

### Simulation 2 — write node vs read node (the actual "eventual")

```text
Tick E1 — write node: BEGIN; insert like; bump likes; COMMIT;
          client already got success
write node          picture 1 likes = 3
read node           picture 1 likes = 2     ← WAL/binlog/oplog not applied yet
                    every row is constraint-valid
                    the view is old

Tick E2 — truck arrives (replay)
read node           likes = 3
                    converged. No new writes → every copy matches HQ.
```

**Pause and predict:** Did CHECK fail on the replica? **No.** Stale ≠ invalid.

User-facing names: **read-your-writes** fail (refresh hits the kiosk, comment missing). **Monotonic reads** fail (request 1 saw the like, request 2 hits an older replica, time goes backward).

### Simulation 3 — two shards (horizontal data scale)

```text
Tick S1 — transfer: debit Alice on shard 1, credit Bob on shard 2
          no distributed snapshot

shard 1             Alice 0
shard 2             Bob 50     (credit not visible yet)
SUM(balance)        50          ← money "vanished" in the report
                    this is not replica lag of one copy
                    it is two commit horizons in one query
```

Replicas: stale **complete** picture. Shards: can mix **two** pictures unless you take a snapshot transaction across shards. Cross-shard FK does not exist as `CREATE TABLE`.

MongoDB chunk moves can temporarily show extra docs until `numOrphanedDocs → 0`.

---

## 3. Why the engine does that

Default copy path (all three):

```text
Write node:  RAM → append log → COMMIT (fsync log) → client OK
Copy path:   log on the network → replica receives → replica APPLIES
Lag window:  client OK  ……  apply on replica
```

| Engine | Log | When a replica read sees the write |
|--------|-----|-------------------------------------|
| PostgreSQL | WAL | After **replay** of the commit. Docs: hot-standby data is **eventually consistent**. |
| InnoDB / MySQL | Binary log | SQL applier. Semi-sync waits for **relay-log receipt**, not apply — still stale reads. |
| MongoDB | Oplog | Secondary apply. `readConcern: local` + `secondary` = textbook eventual read. |

Eventual consistency **solves** "the copy catches up." It does **not** solve Bug A or Bug B.

Vogels session guarantees (name them): read-your-writes, monotonic reads, monotonic writes, writes-follow-reads. Practical combo: sticky session or causal majority R+W.

Horizontal scale:

- **More replicas** → more kiosks → more lag surfaces. FK still on primary.
- **More shards** → no cross-shard FK; multi-shard read without a snapshot can see half a transfer.

---

## 4. Same idea in three engines

### Data-C (Simulation 1) — write node

**PostgreSQL:**

```sql
CREATE TABLE pictures (
  id INT PRIMARY KEY, blob TEXT NOT NULL,
  likes INT NOT NULL DEFAULT 0 CHECK (likes >= 0)
);
CREATE TABLE picture_likes (
  user_name TEXT NOT NULL, picture_id INT NOT NULL,
  PRIMARY KEY (user_name, picture_id),
  FOREIGN KEY (picture_id) REFERENCES pictures(id) ON DELETE CASCADE
);
BEGIN;
INSERT INTO picture_likes VALUES ('Jon', 1);
UPDATE pictures SET likes = likes + 1 WHERE id = 1;
COMMIT;
INSERT INTO picture_likes VALUES ('Edmond', 4);
-- ERROR: FK   (Bug A blocked)
UPDATE pictures SET likes = 5 WHERE id = 1;  -- Bug B: FK silent; don't do this
```

**InnoDB:**

```sql
CREATE TABLE pictures ( ... ) ENGINE=InnoDB;
CREATE TABLE picture_likes (
  ...
  FOREIGN KEY (picture_id) REFERENCES pictures(id) ON DELETE RESTRICT
) ENGINE=InnoDB;
INSERT INTO picture_likes VALUES ('Edmond', 4);  -- errno 1452
-- MyISAM parses FK and ignores it
```

**MongoDB:**

```javascript
// Pattern 1 — embed (no child to orphan)
db.pictures.insertOne({ _id: 1, blob: "xx", likes: [{ user: "Jon" }, { user: "Edmond" }] });
db.pictures.updateOne(
  { _id: 1, "likes.user": { $ne: "Sara" } },
  { $addToSet: { likes: { user: "Sara" } } }
);

// Pattern 2 — two collections: $jsonSchema + unique index. $lookup finds orphans; it does not reject inserts.

// Pattern 3 — snapshot txn on a replica set
const s = db.getMongo().startSession();
s.startTransaction({ readConcern: { level: "snapshot" }, writeConcern: { w: "majority" } });
const pics = s.getDatabase("test").pictures;
const likes = s.getDatabase("test").picture_likes;
if (!pics.findOne({ _id: 1 })) { s.abortTransaction(); throw new Error("no picture"); }
likes.insertOne({ user: "Jon", picture_id: 1 });
pics.updateOne({ _id: 1 }, { $inc: { likes: 1 } });
s.commitTransaction();
```

### Replica lag (Simulation 2) — measure, then close the window

**PostgreSQL:**

```sql
-- primary
SELECT application_name, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
-- standby
SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();

-- THIS transaction visible on sync standby before COMMIT returns:
-- needs synchronous_standby_names; else remote_* silently degrades to local
BEGIN;
SET LOCAL synchronous_commit = remote_apply;
-- insert like ...
COMMIT;
```

| `synchronous_commit` | Waits for | Stops stale standby reads? |
|----------------------|-----------|------------------------------|
| `on` (with sync standby) | Standby **flushed** WAL | **No** — flushed ≠ replayed |
| **`remote_apply`** | Standby **replayed** | **Yes**, on those standbys |

Money / inventory: **read the primary**.

**InnoDB / MySQL:**

```sql
SHOW REPLICA STATUS\G
-- after a write, before a replica read:
SELECT WAIT_FOR_EXECUTED_GTID_SET('source-gtid-set', 10);
-- Group Replication: group_replication_consistency = AFTER / BEFORE
```

Semi-sync ≠ apply. Route checkout reads to the **source**.

**MongoDB:**

```javascript
rs.printSecondaryReplicationInfo();

// default-ish eventual: secondary + local
db.pictures.findOne({ _id: 1 }, { readPreference: "secondary", readConcern: { level: "local" } });

// close the window
db.pictures.findOne({ _id: 1 }, { readPreference: "primary" });

// causal session + majority R+W (read-your-writes, monotonic)
session = db.getMongo().startSession({ causalConsistency: true });
session.getDatabase("test").pictures.updateOne(
  { _id: 1 }, { $inc: { likes: 1 } },
  { writeConcern: { w: "majority" } }
);
session.getDatabase("test").pictures.findOne({ _id: 1 });  // sees own write

// linearizable: single-document, primary only
db.pictures.findOne({ _id: 1 }, { readConcern: { level: "linearizable" } });
```

### Shards (Simulation 3)

MongoDB: `readConcern: "snapshot"` + `w: majority` **in a transaction** is the cross-shard point-in-time. `local` / `majority` without snapshot are not.

PostgreSQL Citus / app-sharding: no distributed SSI. Keep related rows on one shard or accept eventual SUM.

| Symptom | PostgreSQL | InnoDB | MongoDB |
|---------|------------|--------|---------|
| Stale replica read | Primary, or `remote_apply` | Source, or `WAIT_FOR_EXECUTED_GTID_SET` | Primary, or causal majority |
| Checkout / money | Do not read replica | Do not read replica | Do not use `secondary` + `local` |
| Cross-table invariant | FK + one txn | FK + one txn | Embed or snapshot txn |
| Cross-shard | Avoid / co-locate | App-sharded same | Snapshot txn; shard key so related docs share a shard |

---

## 5. Traps + 60-second interview version

| Trap | Reality |
|------|---------|
| "Eventual consistency means the data can be wrong" | Committed data on the write node is as right as your constraints. The **copy** can be **behind**. |
| "FK on the replica will catch Edmond" | FK ran on the primary. Replica replays what committed. |
| "`synchronous_commit=on` means standby reads are fresh" | Flushed ≠ replayed. Need `remote_apply`. |
| "MongoDB `$lookup` is an FK" | Join at read time. |

### 60 seconds

> *"Eventual consistency is the default once you have a write node and a read node. The primary still enforces PK, FK, CHECK — that is ACID data consistency and it does not go away. Replication streams WAL, binlog, or oplog asynchronously, so a replica can miss a just-committed write. That is a stale read, not a broken constraint. PostgreSQL docs call hot-standby data eventually consistent. MySQL async replication and MongoDB secondary plus readConcern local are the same idea. Close the window by routing reads to the primary, waiting for apply (remote_apply, WAIT_FOR_EXECUTED_GTID_SET, causal majority), or accepting lag for feeds. Sharding adds a second problem: no cross-shard FK, and a multi-shard read without a distributed snapshot can see half a transfer."*

### Cheat sheet

1. Data bug ≠ replica lag ≠ shard split. Three simulations.
2. Eventual = copy converges. It never invents an FK.
3. Name the three logs: WAL, binlog, oplog.
4. Checkout reads the write node.

---

## 6. Sources

- [ACID-Updated.pdf](ACID-Updated.pdf)
- [PostgreSQL: Hot Standby](https://www.postgresql.org/docs/current/hot-standby.html)
- [PostgreSQL: Synchronous Commit](https://www.postgresql.org/docs/current/runtime-config-wal.html)
- [MySQL: Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [MongoDB: Read Concern](https://www.mongodb.com/docs/manual/reference/read-concern/)
- [MongoDB: Causal Consistency](https://www.mongodb.com/docs/manual/core/read-isolation-consistency-recency/)
- Werner Vogels — *Eventually Consistent*
