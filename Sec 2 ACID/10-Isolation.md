# Isolation

> **Course:** Fundamentals of Database Engineering — Sec 2: ACID
> **Prerequisite:** [8-what-is-a-transaction.md](8-what-is-a-transaction.md) (the box), [9-Atomicity.md](9-Atomicity.md) (T1's own pencil is never visible to others after rollback).
> **This note:** only the **I**. What T2 is allowed to see while T1 is still running.

The aa/bb swap that Repeatable Read still allows is [15-Serializable-vs-Repeatable_read.md](15-Serializable-vs-Repeatable_read.md). Type the labs in [13-ACID-by-practical-examples.md](13-ACID-by-practical-examples.md).

---

## 0. After this note you can...

- Draw, for any tick, **three numbers**: committed truth, T1's view, T2's view.
- Name dirty read vs non-repeatable read vs phantom vs lost update vs write skew without mixing them.
- Predict T2's `SELECT` **before** looking at the reveal, then pick the isolation level that would have made that tick illegal.
- Draw Alice's row as a **version list** (`v1 xmin=10 balance=100` → `v2 xmin=11 balance=0`) and say which version a snapshot sees.
- Pick a level for "don't show pencil," "this report must not change mid-transaction," and "this invariant spans two rows."

---

## 1. The one picture

Isolation is **not** "run one transaction at a time." The bank runs many cashiers. Isolation is the rule about **whose photocopy** each cashier is allowed to use.

```text
                    committed truth (ink)
                    Alice 100

T1 (cashier A)                    T2 (cashier B)
scratch pad: Alice 0              looking at ... ?
(uncommitted debit)
```

T1 always sees its own pencil. Isolation is T2's answer.

| If T2 sees | Name | Typical level that allows it |
|------------|------|------------------------------|
| 0, while T1 has not committed | **Dirty read** | Read Uncommitted |
| 100, then later 0 after T1 committed | **Non-repeatable read** | Read Committed |
| A new row that did not exist at T2's first `SELECT` | **Phantom** | Read Committed (and standard RR) |
| Both cashiers add $10 and the row ends at 110 | **Lost update** | Any level, if the app does read-then-write |
| Two doctors both go off call, zero remain | **Write skew** | Snapshot / Repeatable Read |

PostgreSQL never does dirty reads (`READ UNCOMMITTED` is implemented as Read Committed). MongoDB never shows another session's uncommitted multi-doc writes. InnoDB at `READ UNCOMMITTED` can.

---

## 2. Simulation — watch it happen

Shared start for simulations 1–4. Open **two terminals**. Isolation is what the **other** terminal sees.

```text
START
committed truth     Alice 100
T1 view             Alice 100
T2 view             Alice 100
```

### Simulation 1 — Dirty read (pencil)

T1 writes 500 and has **not** committed. T2 reads.

**Pause and predict:** At Read Uncommitted, what does T2 see? At Read Committed?

```text
Tick D1 — T1: BEGIN; UPDATE accounts SET balance = 500 WHERE name = 'Alice';
committed truth     Alice 100
T1 view             Alice 500
T2 view             ???
WAL                 XID 42: Alice 100 → 500   (no COMMIT)

Tick D2 — T2: SELECT balance WHERE name = 'Alice';
```

**Reveal:**

```text
Read Uncommitted (InnoDB / SQL Server NOLOCK)
T2 view             Alice 500     ← pencil. Dirty read.

Tick D3 — T1: ROLLBACK
committed truth     Alice 100
T2 already used 500 to make a decision. That 500 never became truth.
```

```text
Read Committed and above (PostgreSQL always)
T2 view             Alice 100     ← latest COMMITTED version
Tick D3 — T1 ROLLBACK. T2 was never lied to.
```

**Do not call Simulation 2 a dirty read.** Dirty = **uncommitted**.

### Simulation 2 — Non-repeatable read (ink changed mid-checkout)

Both values T2 reads are **committed**. The row just changed between T2's two statements.

**Pause and predict:** At Read Committed, second `SELECT` = ? At Repeatable Read?

```text
Tick N1 — T2: BEGIN; SELECT Alice → 100
committed truth     Alice 100
T2 snapshot         Alice 100     (RC: this snapshot is for THIS statement only)

Tick N2 — T1: UPDATE Alice = 200; COMMIT;
committed truth     Alice 200     ← ink

Tick N3 — T2: SELECT Alice again
```

**Reveal:**

```text
Read Committed — new snapshot per statement
T2 first SELECT     100
T2 second SELECT     200
Same transaction, two committed truths. Non-repeatable read.

Repeatable Read / Snapshot — one photocopy for the whole transaction
T2 first SELECT     100
T2 second SELECT     100     ← still looking at the photocopy from Tick N1
committed truth     Alice 200     (T1 did commit; T2 just cannot see it yet)
```

A **report** that reads Alice then Bob at Read Committed can stitch **two commit points** into one result. Every row is valid ink. The **set** is incoherent. That is the bridge to [11.Consistency.md](11.Consistency.md) (read consistency).

### Simulation 3 — Phantom (new row in a range)

Not the same row changing. The **set of rows** matching a `WHERE` grows.

```sql
-- T2
BEGIN;
SELECT COUNT(*) FROM orders WHERE status = 'open';   -- 3

-- T1
INSERT INTO orders (status) VALUES ('open'); COMMIT;

-- T2
SELECT COUNT(*) FROM orders WHERE status = 'open';   -- ?
```

**Pause and predict:** second count at RC? at PostgreSQL Repeatable Read?

**Reveal:**

```text
Tick P1 — T2 counted 3 open orders (snapshot of the range)
committed truth     3 open

Tick P2 — T1 inserts a 4th open order, COMMITs
committed truth     4 open

Tick P3 — T2 counts again
Read Committed                  4     phantom
PostgreSQL Repeatable Read      3     snapshot isolation: insert is after T2's photocopy
SQL standard Repeatable Read   4 possible (phantoms allowed by the standard)
InnoDB RR, plain SELECT          3     consistent nonlocking read
InnoDB RR, SELECT FOR UPDATE      T1's INSERT waits on the gap lock
```

PostgreSQL Repeatable Read has **no phantoms** because it is Snapshot Isolation, **not** because it uses gap locks. That is the interview sentence.

### Simulation 4 — Lost update (two +10s, one vanishes)

Isolation levels do **not** automatically save you if the **application** reads a number, adds 10 in memory, and writes the sum.

```text
Tick U1 — T1 SELECT Alice → 100. T2 SELECT Alice → 100.
committed truth     Alice 100

Tick U2 — T1 UPDATE Alice = 110; COMMIT;
committed truth     Alice 110

Tick U3 — T2 UPDATE Alice = 110; COMMIT;     -- T2 also computed 100+10
committed truth     Alice 110     ← one $10 vanished. Should be 120.
```

**Pause and predict:** Does raising the isolation level to Repeatable Read fix this if the app still writes `110`?

**Reveal:** No. T2 wrote a **constant** it computed from a stale read. Fixes:

```sql
-- 1. Let the engine add (best default)
UPDATE accounts SET balance = balance + 10 WHERE name = 'Alice';

-- 2. Lock the row, then compute
BEGIN;
SELECT balance FROM accounts WHERE name = 'Alice' FOR UPDATE;
UPDATE accounts SET balance = 120 WHERE name = 'Alice';
COMMIT;

-- 3. Optimistic version
UPDATE accounts SET balance = 110, version = 4
 WHERE name = 'Alice' AND version = 3;
-- 0 rows → retry
```

PostgreSQL Repeatable Read **will** abort if T2's `UPDATE` hits a row T1 already changed after T2's snapshot (`could not serialize access due to concurrent update`). That is first-committer-wins on the **same row**. It is not write skew.

```javascript
db.accounts.updateOne({ name: "Alice" }, { $inc: { balance: 10 } });
```

### Simulation 5 — Write skew (two rows, both "valid")

Two doctors. Invariant: **at least one on call**.

```text
START
Alice on_call = true
Bob   on_call = true     -- two on call. Legal.
```

Each doctor reads "the other one is on call," then clocks out.

```text
Tick W1 — T1 and T2 both take a snapshot: count(on_call) = 2
T1 view             Alice true, Bob true
T2 view             Alice true, Bob true

Tick W2 — T1: UPDATE Alice SET on_call = false; COMMIT;
committed truth     Alice false, Bob true     -- still one on call. Legal.

Tick W3 — T2: UPDATE Bob SET on_call = false; COMMIT;
committed truth     Alice false, Bob false    -- ZERO on call. Invariant dead.
```

**Pause and predict:** Did they write the **same** row? Does Repeatable Read abort?

**Reveal:** Different rows. No lost update. Each photocopy was honest. Repeatable Read / Snapshot Isolation **allows** this. Serializable does not (PostgreSQL SSI aborts one `COMMIT` with `40001`). MongoDB snapshot transactions **do not** detect write skew.

The full visual of a swap that no serial order can produce is lecture 15's aa/bb table. This doctors example is the same shape: disjoint writes, broken invariant.

---

## 3. Why the engine does that

### MVCC — Alice as a version list

A write does **not** overwrite Alice in place. It hangs a new version.

```text
Alice's heap (after Simulation 2, T1 committed 200, T2 still in RR)

v1  balance=100   xmin=10 COMMITTED   xmax=11
v2  balance=200   xmin=11 COMMITTED   xmax=0

T2's snapshot was taken when only XID 10 was committed
  → v2's xmin=11 was in-flight (or committed after the snapshot) → skip
  → T2 reads v1 → 100

A new T3 that BEGINs after T1 COMMIT
  → v2 is visible → 200
```

Visibility checklist (PostgreSQL — say this out loud):

1. xmin aborted? → invisible (atomicity of the writer).
2. xmin still in-flight in **my** snapshot's xip list? → invisible (isolation).
3. xmin committed before my snapshot? → candidate.
4. xmax set and that deleter committed before my snapshot? → invisible.

**Side effect:** a long-running T2 holds an old snapshot. `VACUUM` cannot remove v1. Heap bloat. Isolation has a production bill.

InnoDB stores old versions in the **undo log** and builds a **read view** (list of active trx ids). Same idea: pick the newest version visible to that view.

WiredTiger: new document version in cache, invisible outside the session until `commitTransaction`.

### When the snapshot is taken

```text
Read Committed     new photocopy at EVERY statement     Simulation 2 second SELECT sees 200
Repeatable Read    one photocopy at first statement      Simulation 2 second SELECT still 100
Serializable       RR photocopy + extra conflict check  Simulation 5 one COMMIT cancelled
```

### Isolation level × which tick is illegal

SQL standard **minimum** (what the level must prevent). Engines are often stricter.

| Level | Dirty (Sim 1) | Non-repeatable (Sim 2) | Phantom (Sim 3) | Lost update (Sim 4) | Write skew (Sim 5) |
|-------|---------------|------------------------|-----------------|----------------------|---------------------|
| Read Uncommitted | allowed | possible | possible | possible | possible |
| Read Committed | prevented | possible | possible | possible | possible |
| Repeatable Read (standard) | prevented | prevented | possible | possible | possible |
| Snapshot Isolation (PG RR) | prevented | prevented | prevented | same-row: first-committer-wins | **possible** |
| Serializable | prevented | prevented | prevented | prevented | prevented |

Lost update in the app's `SELECT` then `SET balance = 110` is **not** cured by Read Committed. Use atomic `UPDATE col = col + n`.

Dirty **writes** (T2 overwrites T1's uncommitted row) are blocked everywhere with an exclusive lock — even at Read Uncommitted.

### Why locks still exist

MVCC frees **readers** from blocking **writers**. Locks remain for:

| Purpose | Tool |
|--------|------|
| Nobody overwrites in-flight pencil | Exclusive row lock |
| I will increment this row; wait for me | `SELECT ... FOR UPDATE` |
| Nobody inserts into this range (InnoDB RR locking read) | Next-key / gap lock |
| SSI remembered what I read | PostgreSQL `SIReadLock` (does **not** block; aborts at COMMIT) |

Locks cost memory, waits, deadlocks, escalation, and `idle_in_transaction` (you went to lunch holding the row). Long snapshots also freeze old versions — same bloat as above.

PostgreSQL Serializable is **optimistic** (SSI): everyone runs on snapshots, dangerous rw-cycles abort one COMMIT, the app retries. InnoDB / SQL Server Serializable is often **pessimistic**: hold range locks until commit. Lecture 15 is the picture of a pivot.

---

## 4. Same idea in three engines

### Setup

```sql
CREATE TABLE accounts (
  id INT PRIMARY KEY, name TEXT, balance NUMERIC(10,2)
);
INSERT INTO accounts VALUES (1, 'Alice', 100.00);
```

### PostgreSQL — Simulation 1 cannot happen; Simulation 2 can (default RC)

```sql
-- Session 2 (RC default)                          -- Session 1
BEGIN;                                              BEGIN;
SELECT balance FROM accounts WHERE id = 1;       -- 100
                                                    UPDATE accounts SET balance = 200 WHERE id = 1;
                                                    COMMIT;
SELECT balance FROM accounts WHERE id = 1;       -- 200  (Sim 2)
COMMIT;

-- Freeze the photocopy
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;       -- 100
-- Session 1 commits 200
SELECT balance FROM accounts WHERE id = 1;       -- still 100
COMMIT;
```

`READ UNCOMMITTED` in PostgreSQL is **the same as** Read Committed. There is no dirty-read mode.

### InnoDB

Default is **Repeatable Read**. Plain `SELECT` uses a consistent nonlocking read (snapshot from first read). `SELECT FOR UPDATE` / `UPDATE` read the **latest committed** row — MySQL warns that mixing them in one RR transaction gives **two views**. Lock everything you will read, or use Serializable.

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
-- Session 1 has uncommitted 500
SELECT balance FROM accounts WHERE id = 1;  -- may be 500  (Sim 1)
```

Gap / next-key locks apply to **locking** reads at RR, which is how InnoDB stops phantoms on those reads.

### MongoDB

No ISO isolation knob. Uncommitted multi-doc writes are never visible outside the session. Snapshot transactions ≈ PostgreSQL Repeatable Read, **not** Serializable.

```javascript
const s2 = db.getMongo().startSession();
s2.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
});
const accounts = s2.getDatabase("bank").accounts;
accounts.findOne({ _id: 1 });  // 100
// other session commits 200
accounts.findOne({ _id: 1 });  // still 100
session.commitTransaction();
```

| readConcern | SQL-ish |
|-------------|---------|
| `local` | whatever this node has (may roll back on failover) |
| `majority` | majority-committed |
| `snapshot` | one point in time (in a transaction); needs `w: majority` to mean what you think |

Replica lag is [16-Eventual-Consistancy.md](16-Eventual-Consistancy.md), not this lecture.

### Engine matrix

| | PostgreSQL | InnoDB | SQL Server | MongoDB |
|--|------------|--------|------------|---------|
| Default | RC | RR | RC (Azure: RCSI) | single-doc atomic; txn uses readConcern |
| Dirty reads | never | RU only | `NOLOCK` / RU | never outside session |
| RC snapshot | per statement | per statement | per statement (RCSI) | non-txn `find` is not a txn snapshot |
| RR | = Snapshot Isolation, no phantoms | snapshot for plain SELECT; next-key on locking reads | range locks | `readConcern: snapshot` in txn |
| Serializable | SSI, abort `40001` | locking reads become `LOCK IN SHARE MODE` | key-range locks | **not offered** |
| Write skew | SSI detects | possible at RR | possible at SNAPSHOT | snapshot txn does **not** detect |

SQL Server extras: **RCSI** = Read Committed with per-statement snapshots. **SNAPSHOT** isolation = per-transaction snapshot (`ALLOW_SNAPSHOT_ISOLATION`).

---

## 5. Traps + 60-second interview version

### Which level when

| You need | Pick |
|----------|------|
| Don't show pencil | Read Committed (or any default except InnoDB RU / `NOLOCK`) |
| This checkout's numbers must not change mid-transaction | Repeatable Read / Snapshot / MongoDB snapshot txn |
| Range must not grow | PostgreSQL RR already; elsewhere Serializable or gap locks |
| Two writers increment one counter | `UPDATE col = col + n` / `$inc` — not a higher isolation slogan |
| Invariant across **different** rows | PostgreSQL `SERIALIZABLE`, or lock a sentinel row |
| Hot row, you would rather wait than retry | `SELECT FOR UPDATE` |

### Traps

| Trap | Reality |
|------|---------|
| "My session saw the debit, isolation is broken" | You always see your own pencil. Open a **second** terminal. |
| "Case 2 is a dirty read" | Dirty = uncommitted. Case 2 is committed ink that changed. |
| "PostgreSQL RR uses gap locks so it has no phantoms" | It is Snapshot Isolation. Inserts after the photocopy are invisible. |
| "Raising isolation fixes lost updates" | Not if the app writes a constant from a stale `SELECT`. |
| "Snapshot = Serializable" | Snapshot still allows write skew. Lecture 15. |
| "MongoDB is Serializable if I use a transaction" | It is Snapshot Isolation. No pivot detection. |

### 60 seconds

> *"Isolation is what concurrent transactions are allowed to see. It is tunable. Read Committed hides uncommitted pencil — PostgreSQL and MongoDB never dirty-read. Repeatable Read and Snapshot freeze one photocopy so the same row does not change mid-transaction; PostgreSQL Repeatable Read is Snapshot Isolation, which is why it has no phantoms without gap locks. Lost updates are an application read-modify-write bug — fix them with atomic UPDATE or FOR UPDATE. Write skew is two transactions writing different rows that together break an invariant; that needs Serializable or an explicit lock. PostgreSQL Serializable is SSI: it aborts one COMMIT with 40001 and you retry. MongoDB snapshot transactions do not detect write skew."*

### Cheat sheet

1. Isolation = T2's view, not "no concurrency."
2. Dirty = pencil. Non-repeatable = two inks. Phantom = new row in a range.
3. MVCC = version list + snapshot. Readers do not block writers.
4. RC = new snapshot per statement. RR = one snapshot per transaction.
5. PG RR = SI ≠ SQL-standard RR.
6. Lost update ≠ isolation slogan. Lost update = how you wrote the `UPDATE`.
7. Write skew = disjoint writes, broken invariant. Next lecture.

Next: [11.Consistency.md](11.Consistency.md) — valid state vs a report that does not add up.

---

## 6. Sources

- [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [InnoDB: Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- [InnoDB: Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)
- [InnoDB: Next-Key Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html)
- [SQL Server: SET TRANSACTION ISOLATION LEVEL](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-transaction-isolation-level-transact-sql)
- [MongoDB: Read Concern snapshot](https://www.mongodb.com/docs/manual/reference/read-concern-snapshot/)
- [MongoDB: Transactions](https://www.mongodb.com/docs/manual/core/transactions/)
- Berenson et al. (1995) — *A Critique of ANSI SQL Isolation Levels*
- Ports & Grittner (2012) — Serializable Snapshot Isolation in PostgreSQL
