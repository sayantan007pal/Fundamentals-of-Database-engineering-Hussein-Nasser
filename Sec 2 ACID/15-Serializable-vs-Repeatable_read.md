# Serializable vs Repeatable Read

> **Course:** Fundamentals of Database Engineering — Sec 2: ACID
> **Prerequisite:** [10-Isolation.md](10-Isolation.md) (write skew in one paragraph). **This note:** Hussein’s aa/bb swap — when a frozen photocopy is still not serial.
> **Lecture slides:** [read.pdf](read.pdf)
> **Typed in PostgreSQL, InnoDB, and MongoDB.**

---

## 0. After this note you can...

- Draw the six cells before and after under Repeatable Read (swap) vs the two legal serial endings (all aa / all bb).
- Say why Repeatable Read is **silent** (write sets disjoint) and why PostgreSQL Serializable cancels **at COMMIT** (rw-edges, pivot).
- Type the same swap in InnoDB (blocks) and MongoDB (both commit — no Serializable).

---

## 1. The one picture

```text
start (Hussein)
id    1     2     3     4     5     6
t     aa    bb    bb    bb    aa    aa
      three aa, three bb
```

T1: `UPDATE ... SET t = 'bb' WHERE t = 'aa';`  (touches ids 1, 5, 6)  
T2: `UPDATE ... SET t = 'aa' WHERE t = 'bb';`  (touches ids 2, 3, 4)

They never write the **same** row. Repeatable Read thinks there is no fight.

If they had run **one after the other**:

```text
T1 then T2:  all aa became bb → all bb → T2 turns every bb to aa → ALL aa

T2 then T1:  all bb became aa → all aa → T1 turns every aa to bb → ALL bb
```

There is **no** serial order that ends mixed. If the committed table is still mixed, isolation was weaker than Serializable.

---

## 2. Simulation — watch the cells

### Under Repeatable Read (both commit)

```text
Tick 0  start
        aa  bb  bb  bb  aa  aa

Tick 1  both take the same photocopy: 3 aa, 3 bb

Tick 2  T1 rewrites ITS aa → bb          T2 rewrites ITS bb → aa
        (ids 1,5,6)                      (ids 2,3,4)
        no overlapping ids

Tick 3  both COMMIT
        bb  aa  aa  aa  bb  bb
        still 3 aa, 3 bb — SWAPPED
```

**Pause and predict:** Would PostgreSQL raise `could not serialize access due to concurrent update`?

**Reveal:** **No.** That error is first-committer-wins on the **same** row. These are different rows. RR is silent. That is write skew / serialization anomaly.

T1's private view after its UPDATE looks like all bb. T2's looks like all aa. Committed truth is the swap.

### Under Serializable (PostgreSQL SSI)

Same UPDATEs. They **succeed**. Detection waits for `COMMIT`.

```text
T1 read predicate t='aa';  T2 later wrote those rows  →  rw-edge
T2 read predicate t='bb';  T1 later wrote those rows  →  rw-edge

Tin --rw--> Tpivot --rw--> Tout
```

Two consecutive rw-edges = dangerous structure. The middle transaction is the **pivot**. Theorem: in a real cycle, **Tout commits first**. PostgreSQL lets that COMMIT through, then **cancels the other COMMIT**:

```text
ERROR:  could not serialize access due to read/write dependencies among transactions
SQLSTATE 40001
```

Retry the loser. It now sees the winner's table. The `WHERE` matches a serial world → **all aa** or **all bb**.

```text
SIREAD flags (not locks that wait)
        T1 photographed "aa"          T2 photographed "bb"
        T1 wrote aa→bb                 T2 wrote bb→aa
        flags form a loop
        one stamp shredded at COMMIT
```

InnoDB Serializable does **not** abort at COMMIT. It **blocks** (shared / next-key locks on the read set). MongoDB has **no** this knob: both commit, same swap as RR.

---

## 3. Why the engine does that

Repeatable Read asks: *"did my photocopy stay stable?"*  
Serializable asks: *"can these photocopies be ordered into a single story?"*

| Pattern | Same rows written? | PG Repeatable Read | PostgreSQL Serializable |
|---------|---------------------|--------------------|------------------------|
| Lost update (Sim 4 in lecture 10) | Yes | Abort on second UPDATE | Abort |
| Write skew / aa/bb | No | **Both commit** | One COMMIT `40001` |
| Phantom | Insert into a range | Invisible on snapshot | SSI also covers predicate |

Hard rule: every transaction in the invariant must run `SERIALIZABLE`. A Repeatable Read partner is **invisible** to the SSI graph.

SIREAD locks are **flags** in `pg_locks` (`mode = 'SIReadLock'`). They do not block readers or writers. Failure mode is abort + retry, not deadlock.

False positives possible (dangerous structure that is not a real cycle). Never a false negative for SI anomalies.

Pick Serializable when the invariant is a **predicate** (count, sum, "all rows of this color", "at least one doctor") and writes land on **different** rows. Skip it when one `UPDATE col = col + n` or one document `$inc` is enough.

---

## 4. Same idea in three engines

### Setup (PostgreSQL / InnoDB)

```sql
DROP TABLE IF EXISTS items;
CREATE TABLE items (id INT PRIMARY KEY, t CHAR(2) NOT NULL);
INSERT INTO items (id, t) VALUES
  (1, 'aa'), (2, 'bb'), (3, 'bb'),
  (4, 'bb'), (5, 'aa'), (6, 'aa');
```

Reset: put those six values back.

### PostgreSQL — Lab A (RR allows swap) then Lab B (SSI cancels)

Two `psql` windows.

```sql
-- Session 1                                      -- Session 2
BEGIN ISOLATION LEVEL REPEATABLE READ;           BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT t FROM items ORDER BY id;                 -- same photocopy
UPDATE items SET t = 'bb' WHERE t = 'aa';      UPDATE items SET t = 'aa' WHERE t = 'bb';
COMMIT;                                           COMMIT;
SELECT t FROM items ORDER BY id;
-- SWAPPED MIX
```

```sql
-- reset, then:
BEGIN ISOLATION LEVEL SERIALIZABLE;              BEGIN ISOLATION LEVEL SERIALIZABLE;
UPDATE items SET t = 'bb' WHERE t = 'aa';      UPDATE items SET t = 'aa' WHERE t = 'bb';
COMMIT;                                           COMMIT;
-- one succeeds; the other: SQLSTATE 40001
-- retry loser → all aa or all bb
```

Optional third session while both are open:

```sql
SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks WHERE mode = 'SIReadLock';
```

PostgreSQL docs `mytab` (same anomaly, sums):

```sql
CREATE TABLE mytab (class INT, value INT);
INSERT INTO mytab VALUES (1, 10), (1, 20), (2, 100), (2, 200);
-- Session A SERIALIZABLE: SUM class=1 (=30), INSERT class=2 value=30
-- Session B SERIALIZABLE: SUM class=2 (=300), INSERT class=1 value=300
-- RR: both commit (30 and 300 — impossible serially)
-- SERIALIZABLE: one COMMIT cancelled
```

### InnoDB — RR swaps; SERIALIZABLE waits

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
UPDATE items SET t = 'bb' WHERE t = 'aa';
COMMIT;
-- other session: UPDATE bb→aa; COMMIT;  → swapped mix
```

```sql
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
SELECT * FROM items WHERE t = 'aa';   -- implicit FOR SHARE + next-key
UPDATE items SET t = 'bb' WHERE t = 'aa';
COMMIT;
-- Session 2 waits, then sees a serial world (all aa or all bb)
-- failure mode: blocking / deadlock, not 40001
```

Or at RR, lock the range yourself: `SELECT * FROM items WHERE t IN ('aa','bb') FOR UPDATE;`

### MongoDB — snapshot ≈ RR; write skew allowed

Replica set required.

```javascript
use acid_lab
db.items.drop();
db.items.insertMany([
  { _id: 1, t: "aa" }, { _id: 2, t: "bb" }, { _id: 3, t: "bb" },
  { _id: 4, t: "bb" }, { _id: 5, t: "aa" }, { _id: 6, t: "aa" }
]);

// Session 1                                          // Session 2
const s1 = db.getMongo().startSession();              const s2 = db.getMongo().startSession();
s1.startTransaction({                              s2.startTransaction({
  readConcern: { level: "snapshot" },                readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }                    writeConcern: { w: "majority" }
});                                                });
s1.getDatabase("acid_lab").items.updateMany(       s2.getDatabase("acid_lab").items.updateMany(
  { t: "aa" }, { $set: { t: "bb" } });               { t: "bb" }, { $set: { t: "aa" } });
s1.commitTransaction();                           s2.commitTransaction();
db.items.find().sort({ _id: 1 });
// swapped mix — both committed
```

WiredTiger only fail-on-conflicts **the same document**. Workaround: a **sentinel** both transactions `$inc`:

```javascript
db.constraints.updateOne(
  { _id: "color_batch" },
  { $setOnInsert: { epoch: 0 } },
  { upsert: true }
);
// inside each txn, also:
// db.constraints.updateOne({ _id: "color_batch" }, { $inc: { epoch: 1 } }, { session })
// loser: WriteConflict / TransientTransactionError → retry
```

Or embed the six letters in **one** document and `$set` atomically.

Retry (PostgreSQL `40001` and MongoDB transient):

```javascript
async function runWithRetry(fn, maxRetries = 8) {
  for (let i = 0; i < maxRetries; i++) {
    try { return await fn(); }
    catch (e) {
      const retry = e.code === "40001" || e.hasErrorLabel?.("TransientTransactionError");
      if (!retry || i === maxRetries - 1) throw e;
    }
  }
}
```

| Engine | How "serializable" is enforced | aa/bb result |
|--------|--------------------------------|--------------|
| PostgreSQL | SSI, non-blocking flags, abort at COMMIT | One `40001` |
| InnoDB | Pessimistic next-key / FOR SHARE | Second session **waits** |
| MongoDB | Not offered (SI only) | **Both commit**, swap |

---

## 5. Traps + 60-second interview version

| Trap | Reality |
|------|---------|
| "RR aborted so it is serializable" | Same-row first-committer-wins ≠ write skew |
| "PostgreSQL cancelled the UPDATE" | UPDATEs succeed. **COMMIT** is cancelled. |
| "MongoDB snapshot = Serializable" | SI. Disjoint docs both commit. |
| "Mixing RR and SERIALIZABLE is fine" | SSI cannot see the RR partner. Skew returns. |

### 60 seconds

> *"Repeatable Read freezes a snapshot. Serializable additionally requires that committed transactions be equivalent to some one-at-a-time order. Hussein's table starts three aa and three bb. T1 turns aa into bb; T2 turns bb into aa; they write disjoint rows. Repeatable Read lets both commit and the table is still mixed — swapped — which no serial order produces. PostgreSQL Serializable tracks read/write dependencies, identifies a pivot, and cancels one COMMIT with 40001; the app retries. InnoDB Serializable blocks with next-key locks. MongoDB snapshot transactions behave like Repeatable Read; you invent a sentinel document if you need a conflict."*

### Cheat sheet

1. Serial endings: all aa or all bb. Mix = not serial.
2. Disjoint writes → RR silent.
3. SSI: rw-edges, pivot, cancel at COMMIT.
4. InnoDB waits. MongoDB both commit.

---

## 6. Sources

- [read.pdf](read.pdf)
- [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- Ports & Grittner (2012) — Serializable Snapshot Isolation in PostgreSQL
- [InnoDB: Next-Key Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html)
- [MongoDB: Transactions](https://www.mongodb.com/docs/manual/core/transactions/)
