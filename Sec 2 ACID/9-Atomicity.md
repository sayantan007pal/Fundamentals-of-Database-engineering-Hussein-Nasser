# Atomicity

> **Course:** Fundamentals of Database Engineering — Sec 2: ACID
> **Prerequisite:** [8-what-is-a-transaction.md](8-what-is-a-transaction.md) — the shared Alice/Bob universe and the RAM / WAL / data-file box.
> **This note:** only the **A**. All of the transfer, or the database looks as if it never started.

---

## 0. After this note you can...

- Point at the same transfer and say whether the process is **still alive** or **dead** — those are two different recovery paths.
- Walk the RAM / WAL / data-file box through: debit Alice → credit fails → `ROLLBACK`.
- Walk the same box through: crash **before** COMMIT, and crash **2 seconds after** COMMIT.
- Say how PostgreSQL hides the debit (aborted XID), how InnoDB undoes it (before-image), and how MongoDB discards it (free the RAM buffer).
- Refuse the trap: atomicity does **not** mean "total money is conserved." That is consistency. Atomicity only forbids a **half-applied** transaction.

---

## 1. The one picture

Same bank as lecture 8.

```text
START
committed truth     Alice 100   Bob 50
T1 private view     Alice 100   Bob 50
WAL                 (no transfer yet)
data files          Alice 100   Bob 50
total money         150
```

Atomicity's promise: you will **never** observe this as committed truth:

```text
Alice   0   Bob 50     ← $100 vanished. Forbidden.
```

You may temporarily **write** Alice to 0 in RAM. Other sessions must not see it. If the second statement fails, or the lights go out before COMMIT, committed truth is still `Alice 100, Bob 50`.

```text
without atomicity                         with atomicity
Alice 0, Bob 50   money gone               Alice 100, Bob 50   transfer never happened
```

---

## 2. Simulation — watch it happen

Two failure modes. Interviewers blur them. Keep them in separate boxes.

```text
process still alive? ──yes──► live rollback (statement error or ROLLBACK)
                    └──no───► crash recovery (REDO the log, then hide losers)
```

### Simulation 1 — statement fails, process alive

T1 transfers $100. Alice's debit succeeds. Bob's credit fails (no such account). The server is still running.

**Pause and predict:** After `ROLLBACK`, what does T2's `SELECT` return? What is on disk?

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE name = 'Nobody';  -- fails
ROLLBACK;
```

**Reveal:**

```text
Tick L1 — BEGIN, XID 42
committed truth     Alice 100   Bob 50
T1 private view     Alice 100   Bob 50
WAL                 XID 42 started
data files          Alice 100   Bob 50

Tick L2 — debit Alice in RAM
committed truth     Alice 100   Bob 50     ← T2 still sees this
T1 private view     Alice   0   Bob 50
WAL (not fsynced)   XID 42: Alice 100 → 0
data files          Alice 100   Bob 50
                    PostgreSQL also has a new heap tuple (balance=0, xmin=42)
                    that no other snapshot can read yet

Tick L3 — credit fails. Transaction is aborted.
committed truth     Alice 100   Bob 50
T1 private view     dead — further SQL rejected until ROLLBACK (PostgreSQL)
WAL                 still no COMMIT record for XID 42
data files          Alice 100   Bob 50

Tick L4 — ROLLBACK
committed truth     Alice 100   Bob 50
T1 private view     Alice 100   Bob 50
WAL                 XID 42 aborted (or: no commit record — same effect after crash)
data files          Alice 100   Bob 50
                    InnoDB: undo log restored Alice's before-image in the buffer pool
                    PostgreSQL: xmin=42 is ABORTED; old tuple (100) is still the visible one
                    MongoDB: in-memory buffer freed; journal never saw this txn
```

T2, the whole time, saw Alice 100. There was never a committed half-transfer. That is atomicity for a **live** failure.

PostgreSQL gotcha after Tick L3: you cannot `COMMIT` the debit "anyway." The block is dead.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
SELECT 1/0;
-- ERROR: current transaction is aborted, commands ignored until end of transaction block
COMMIT;     -- still aborted
ROLLBACK;   -- the only way out
```

### Simulation 2 — crash, process dead

The engine is gone. Your `catch` block is gone. On restart the engine runs **crash recovery**, not your application.

Two freeze-frames. Same transfer. Different WAL.

#### 2a. Crash before COMMIT

```text
Tick C-before — debit in RAM, no COMMIT yet, power cut
committed truth     (server is down)
T1 private view     gone — process died
WAL on disk         maybe the Alice change is in the log file, maybe only in RAM
                    there is NO durable COMMIT record for XID 42
data files          Alice 100   Bob 50  (or a stolen dirty page — see STEAL below)
```

**Pause and predict:** After restart, Alice = ?

**Reveal:** Alice 100, Bob 50. Recovery rule: **no durable COMMIT record → loser.** Pretend the transaction never started.

```text
Tick C-before-recovery
1. Find last checkpoint
2. REDO WAL forward from that checkpoint (repeat history — even loser bytes may be replayed)
3. XID 42 has no COMMIT record → loser
   InnoDB: undo Alice using the before-image
   PostgreSQL: treat XID 42 as ABORTED; MVCC hides the new tuple
   MongoDB: uncommitted work was RAM-only (or never journaled as a commit) → gone
committed truth     Alice 100   Bob 50
```

#### 2b. Crash 2 seconds after COMMIT

```sql
BEGIN;
UPDATE ... Alice - 100;
UPDATE ... Bob   + 100;
COMMIT;     -- client already received "success"
-- 2 seconds later: power cut. Data files still say Alice 100, Bob 50.
```

```text
Tick C-after — COMMIT returned, data files stale, power cut
committed truth     (server is down — but the receipt book has a stamp)
WAL on disk         COMMIT record for XID 42   ← this is what "COMMIT" meant
data files          Alice 100   Bob 50          ← checkpoint had not run yet
```

**Pause and predict:** After restart, Alice = ?

**Reveal:** Alice 0, Bob 150. Recovery **REDOs** the WAL. The transfer survives even though the table file was stale. That is atomicity sharing the log with [durability](12.Durability.md).

```text
Tick C-after-recovery
1. Find last checkpoint
2. REDO: replay "Alice 100→0, Bob 50→150" onto the data pages
3. COMMIT record exists → winner. No undo.
committed truth     Alice 0   Bob 150
```

| Crash moment | COMMIT record on disk? | After recovery |
|--------------|-------------------------|----------------|
| Mid-statement, before COMMIT | No | Transfer never happened |
| After COMMIT returned | Yes | Transfer stands |
| After COMMIT returned, `synchronous_commit=off`, WAL not flushed yet | No | Transfer **lost** (not corrupt) — durability knob, lecture 12 |

---

## 3. Why the engine does that

### Live rollback vs crash recovery

| | Live rollback | Crash recovery |
|--|---------------|----------------|
| Process | Alive | Dead, then restart |
| Who runs it | Your `ROLLBACK`, or the engine after a statement error | Startup code |
| Goal | Hide or reverse XID 42 **now** | Reconstruct pages, then decide winners vs losers |
| PostgreSQL | Mark XID `ABORTED` in clog. Heap tuples stay. `VACUUM` later. | REDO WAL. Missing commit = abort. No row-by-row undo pass. |
| InnoDB | Apply **undo log** (before-images) in reverse | ARIES: Analysis → Redo **all** → Undo **losers** (CLRs so undo itself is crash-safe) |
| MongoDB | Free WiredTiger's in-memory buffer. No journal record. | Replay journal from last checkpoint. Only committed records exist in the journal. |

Crash recovery **replays forward first** (REDO), then undoes losers. You do not "delete partial work" off the disk. You reconstruct the moment of the crash, then decide.

### What each engine actually stores at Tick L2

Same debit, three physical stories.

```text
PostgreSQL heap after debit, not committed
  old tuple:  Alice 100   xmin=10 COMMITTED   xmax=42 (being updated)
  new tuple:  Alice   0   xmin=42 IN_PROGRESS
  clog:       XID 42 = in progress
  T2's snapshot: xmin 42 is in-flight → skip new tuple → Alice 100

After ROLLBACK
  clog:       XID 42 = ABORTED
  new tuple still on the page, invisible forever until VACUUM
```

```text
InnoDB after debit, not committed
  buffer pool page: Alice 0
  undo log:          before-image Alice 100
  redo log:          after-image of the change
  T2:                MVCC read view → sees 100 from undo

After ROLLBACK
  undo applied: buffer pool Alice 100 again
```

```text
MongoDB multi-doc after debit, not committed
  WiredTiger cache:  Alice 0 inside T1's txn buffer
  journal:           nothing for this txn yet
  T2:               cannot see T1's session writes

After abortTransaction
  buffer freed. Disk unchanged.
```

### STEAL / NO-FORCE (why RAM work + log commit)

Modern OLTP is **STEAL + NO-FORCE**:

| Policy | Meaning | Why |
|--------|---------|-----|
| **STEAL** | An uncommitted dirty page **may** be written to a data file (buffer pool needed the frame) | So RAM is not a prison |
| **NO-FORCE** | COMMIT does **not** wait for data pages — only the log | Sequential fsync, group commit, fast TPS |

If InnoDB **stole** Alice's page to disk at Tick L2, crash recovery **must** undo it. That is why InnoDB has an undo log. PostgreSQL can also write the new tuple to disk before commit; after crash, clog + MVCC make it invisible. MongoDB checkpoints only **visible** (committed) data.

Process B from lecture 8 — force every data page on every COMMIT — would make rollback a second random rewrite of those pages. Nobody uses it for OLTP.

Atomicity **does not** enforce "total money = 150." It only forbids a visible half-transfer. Two concurrent withdrawals that both pass `CHECK (balance >= 0)` and leave Alice at −60 are a **consistency + isolation** problem. See [11.Consistency.md](11.Consistency.md).

---

## 4. Same idea in three engines

### PostgreSQL — type this

```sql
-- success
BEGIN;
UPDATE accounts SET balance = balance - 100.00
 WHERE name = 'Alice' AND balance >= 100.00;
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Bob';
COMMIT;

-- live failure (Simulation 1)
BEGIN;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Nobody';
ROLLBACK;   -- required. COMMIT will not save you.

-- savepoint: undo the debit, keep the audit row
BEGIN;
INSERT INTO audit_log (event) VALUES ('transfer started');
SAVEPOINT before_transfer;
UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
ROLLBACK TO SAVEPOINT before_transfer;
COMMIT;
```

psql-only: `\set ON_ERROR_ROLLBACK on` plants a savepoint before each statement so `SELECT 1/0` does not kill the whole block. That is a client convenience, not how your app should work.

### InnoDB

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Nobody';
ROLLBACK;   -- undo log restores Alice
```

| Log | Role |
|-----|------|
| Redo | After-images. Crash REDO of winners (and of history that undo will reverse). |
| Undo | Before-images. Live rollback + crash UNDO of losers + MVCC old versions. |

ARIES on restart: Analysis (who was active, which pages dirty) → Redo everything after the checkpoint → Undo losers, writing compensation log records so a crash **during undo** is also safe.

### MongoDB

**Prefer one document** (already atomic — Simulation 1 cannot leave Alice debited):

```javascript
db.bank.updateOne(
  { _id: "bank_main", "accounts.name": "Alice", "accounts.balance": { $gte: 100 } },
  { $inc: { "accounts.$[alice].balance": -100, "accounts.$[bob].balance": 100 } },
  { arrayFilters: [{ "alice.name": "Alice" }, { "bob.name": "Bob" }] }
);
```

**Multi-document** — abort is free RAM:

```javascript
const session = db.getMongo().startSession();
try {
  session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" }
  });
  const accounts = session.getDatabase("bank").accounts;
  const debit = accounts.updateOne(
    { name: "Alice", balance: { $gte: 100 } },
    { $inc: { balance: -100 } }
  );
  if (debit.modifiedCount !== 1) throw new Error("Alice cannot pay");
  accounts.updateOne({ name: "Nobody" }, { $inc: { balance: 100 } });
  session.commitTransaction();
} catch (e) {
  session.abortTransaction();  // buffer freed, no journal record
} finally {
  session.endSession();
}
```

Auto-abort (retry): write conflict, ~60 s limit, cache pressure, `TransactionTooLargeForCache`, primary failover (`UnknownTransactionCommitResult`). Always retry labeled transient errors.

| | PostgreSQL | InnoDB | MongoDB (multi-doc) |
|--|------------|--------|---------------------|
| Live rollback | Mark XID aborted; tuples stay | Apply undo log | Free RAM buffer |
| Crash losers | No commit in WAL → aborted | ARIES undo pass | Never journaled as commit |
| Undo log | No (MVCC + clog) | Yes | No |
| Cost of abort | Cheap now, VACUUM later | Medium now | Cheapest |
| COMMIT | WAL fsync | Redo fsync | One journal commit record |

---

## 5. Traps + 60-second interview version

### Traps

| Trap | Reality |
|------|---------|
| "Rollback and crash recovery are the same" | Live = process alive. Crash = REDO then decide winners. |
| "PostgreSQL rewrites Alice back to 100" | It marks XID aborted. The new tuple is invisible. |
| "Atomicity keeps total money at 150" | Atomicity prevents a **partial** transaction. The invariant is **consistency**. |
| "COMMIT wrote accounts to disk, so crash is fine" | COMMIT wrote the **log**. Simulation 2b is the proof. |
| "I can continue after an error in PostgreSQL" | Not without a savepoint. The block is dead. |
| "MongoDB abort writes an abort record" | Abort discards RAM. Nothing to journal. |

### 60 seconds

> *"Atomicity means the transaction is indivisible. Either every change becomes committed truth together, or the database looks as if it never started. Two failure modes: if a statement fails while the server is up, that is live rollback — InnoDB applies undo, PostgreSQL marks the XID aborted and MVCC hides the new tuple, MongoDB frees the in-memory buffer. If the process dies, that is crash recovery: find the last checkpoint, REDO the WAL or journal, then any transaction without a durable COMMIT record is a loser. COMMIT means the log is durable, not that data files were updated. STEAL plus NO-FORCE is why that design is fast."*

### Cheat sheet

1. Atomicity = no visible half-transfer.
2. Live failure ≠ crash. Name which one.
3. No COMMIT record → loser.
4. COMMIT record fsynced → winner, even if data files are stale.
5. PostgreSQL abort = clog bit. InnoDB abort = undo. MongoDB abort = free buffer.
6. STEAL + NO-FORCE = modern OLTP.

Next: [10-Isolation.md](10-Isolation.md) — what T2 is allowed to see while T1 is still pencil.

---

## 6. Sources

- [PostgreSQL: Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)
- [PostgreSQL: WAL](https://www.postgresql.org/docs/current/wal-intro.html)
- [PostgreSQL: WAL Internals](https://www.postgresql.org/docs/current/wal-internals.html)
- [How Postgres Makes Transactions Atomic — Brandur](https://brandur.org/postgres-atomicity)
- [MongoDB: Transactions](https://www.mongodb.com/docs/manual/core/transactions/)
- [MongoDB: Journaling](https://www.mongodb.com/docs/manual/core/journaling/)
- Mohan et al., ARIES (Analysis, Redo, Undo)
