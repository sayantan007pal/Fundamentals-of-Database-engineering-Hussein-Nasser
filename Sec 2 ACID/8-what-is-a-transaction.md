# What Is a Transaction?

> **Course:** Fundamentals of Database Engineering — Sec 2: ACID
> **Lecture:** What is a transaction
> **Goal:** After this note you can watch a $100 transfer fail in two different ways, say exactly what a transaction is, and point at which ACID letter each camera is filming.

Later notes go deep on one letter: [Atomicity](9-Atomicity.md) · [Isolation](10-Isolation.md) · [Consistency](11.Consistency.md) · [Durability](12.Durability.md). This note is the picture they all share.

---

## 0. After this note you can...

- Draw one box that is **committed truth**, another that is **T1's private view**, and a third that is **data files on disk** — and keep them different.
- Predict what happens if Alice is debited **without** `BEGIN`, then the credit to Bob errors.
- Predict the same two statements **inside** `BEGIN` … `ROLLBACK`.
- Name the four ACID cameras on that same transfer in one sentence each.
- Say the sentence interviewers wait for: **COMMIT means the log is durable, not that table files were rewritten.**

---

## 1. The one picture

First National Bank has two rows. That is the whole universe for lectures 8, 9, 10, and 12.

```text
accounts
┌────────┬───────┬─────────┐
│ id     │ name  │ balance │
├────────┼───────┼─────────┤
│ 1      │ Alice │ 100    │
│ 2      │ Bob   │  50    │
└────────┴───────┴─────────┘
total money in the system = 150
```

A **transfer** is not one SQL statement. It is two:

```sql
UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
```

Those two statements are one **logical unit of work**. Either both become the new truth, or neither does. That unit is a **transaction**.

```text
                    the transfer (one transaction)
                    ┌─────────────────────────────────────┐
                    │  debit Alice          credit Bob     │
                    │  100 → 0             50 → 150      │
                    └─────────────────────────────────────┘
                              all of this, or none of this
```

Without wrapping them, each statement is its own tiny transaction (autocommit). That is how money vanishes.

---

## 2. Simulation — watch it happen

Shared starting snapshot. Every later tick updates this same box.

```text
START
committed truth     Alice 100   Bob 50
T1 private view     (same — no open transaction yet)
WAL                 (empty of this transfer)
data files          Alice 100   Bob 50
```

### Simulation A — no BEGIN (autocommit). Credit fails.

PostgreSQL, MySQL, and MongoDB all **commit each successful statement immediately** unless you open a transaction.

**Pause and predict:** After the first `UPDATE` succeeds and the second errors, what are Alice and Bob?

```sql
-- T1, autocommit on (the default)
UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
-- success. That statement COMMITTED.

UPDATE accounts SET balance = balance + 100 WHERE name = 'Nobody';
-- 0 rows (or an error). There is nothing left to roll back.
```

**Reveal:**

```text
Tick A1 — debit committed as its own transaction
committed truth     Alice   0   Bob 50     ← already the new truth
T1 private view     Alice   0   Bob 50
WAL (fsynced)       COMMIT of "Alice 100 → 0"
data files          may still say 100 until checkpoint; WAL says 0

Tick A2 — credit fails. There is no open transaction.
committed truth     Alice   0   Bob 50     ← $100 vanished
T1 private view     Alice   0   Bob 50
WAL                 no Bob credit was ever committed
data files          Alice 0 (after recovery / checkpoint)
```

Alice is poorer. Bob is unchanged. Total money is **50**. The database did exactly what you asked, statement by statement. It did not know those two lines were one business action.

That is why we have transactions.

### Simulation B — BEGIN, then the same error, then ROLLBACK.

**Pause and predict:** Same two statements, but wrapped. After `ROLLBACK`, what are Alice and Bob?

```sql
BEGIN;   -- or START TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
-- success, but NOT committed. Pencil, not ink.

UPDATE accounts SET balance = balance + 100 WHERE name = 'Nobody';
-- fails

ROLLBACK;
```

**Reveal:**

```text
Tick B1 — BEGIN. Engine assigns a transaction id (XID).
committed truth     Alice 100   Bob 50
T1 private view     Alice 100   Bob 50     T1 sees its own writes from here on
WAL                 XID 42 started
data files          Alice 100   Bob 50

Tick B2 — debit Alice. Other cashiers still see 100.
committed truth     Alice 100   Bob 50     ← Isolation: other sessions do not see pencil
T1 private view     Alice   0   Bob 50
WAL (not fsynced)   XID 42: Alice 100 → 0
data files          Alice 100   Bob 50

Tick B3 — credit fails. Transaction is doomed.
committed truth     Alice 100   Bob 50
T1 private view     (aborted — further SQL is rejected until ROLLBACK)
WAL                 no COMMIT record for XID 42
data files          Alice 100   Bob 50

Tick B4 — ROLLBACK
committed truth     Alice 100   Bob 50     ← as if the transfer never started
T1 private view     Alice 100   Bob 50
WAL                 XID 42 aborted (or simply: no commit record)
data files          Alice 100   Bob 50
```

Same two SQL statements. Opposite bank. The only difference is **the unit of work**.

### Simulation C — both statements succeed, then COMMIT.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
COMMIT;
```

```text
Tick C1 — both writes in RAM, not committed
committed truth     Alice 100   Bob  50
T1 private view     Alice   0   Bob 150
WAL (not fsynced)   XID 42: Alice 100→0 ; Bob 50→150
data files          Alice 100   Bob  50

Tick C2 — COMMIT. Engine fsyncs the WAL, then tells the client "ok".
committed truth     Alice   0   Bob 150     ← now everyone sees this
T1 private view     Alice   0   Bob 150
WAL (fsynced)       COMMIT record for XID 42
data files          still Alice 100 Bob 50 until checkpoint
```

**The sentence to keep:** after `COMMIT`, other sessions see the new balances. The **table files** may still hold the old bytes. Recovery will replay the WAL. That is lecture 12. You only need the picture now:

```text
scratch pad (RAM)     →   receipt book (WAL)     →   cash drawer (data files)
  dirty pages               fsync on COMMIT            checkpoint, later
```

---

## 3. Why the engine does that

### The unit of work

```
Step 1 → BEGIN / START TRANSACTION   (or autocommit: the engine begins for you, per statement)
Step 2 → Engine assigns a transaction id
Step 3 → Reads come from pages already in RAM, or from disk into the buffer pool
Step 4 → Writes change RAM (dirty pages) and append a description to the WAL/journal buffer
Step 5 → Other sessions do not see those writes yet (isolation — lecture 10)
Step 6 → COMMIT: flush WAL to disk, then return success to the client
         ROLLBACK: hide or undo the RAM changes; no commit record
Step 7 → Later: checkpoint writes dirty data pages to table files
```

A session **always sees its own uncommitted writes**. Isolation is about **other** sessions. If you `SELECT` Alice after debiting her in the same `BEGIN`, you will see `0`. That does not prove isolation. Isolation is what the cashier at the next terminal does **not** see.

### Four cameras on the same transfer

Once you can see Simulation B and C, ACID is four cameras pointed at that box.

| Camera | What it films on the Alice → Bob transfer |
|--------|------------------------------------------|
| **A Atomicity** | You never observe "Alice 0, Bob 50" as committed truth. All of the transfer, or none. [9-Atomicity.md](9-Atomicity.md) |
| **C Consistency** | After COMMIT or ROLLBACK, the rules still hold: PK/FK/CHECK, and "total money = 150" if your transaction was written to preserve it. [11.Consistency.md](11.Consistency.md) |
| **I Isolation** | While T1's transfer is pencil, T2's `SELECT` still sees Alice 100 (at Read Committed and above). [10-Isolation.md](10-Isolation.md) |
| **D Durability** | After COMMIT returns, a power cut must not restore Alice to 100. The stamped receipt (WAL) is on disk. [12.Durability.md](12.Durability.md) |

They are not four separate features you turn on. They are four promises about **one unit of work**.

```text
T1: BEGIN
    debit Alice          ─┐
    credit Bob            ├─ Atomicity: both become truth together, or neither
    COMMIT               ─┘
         │
         ├─ Isolation: T2 cannot read the debit until COMMIT
         ├─ Consistency: after COMMIT, constraints + your invariants hold
         └─ Durability: after COMMIT returns, crash recovery will restore the transfer
```

### Two commit processes

Modern engines use **Process A**. Process B is the naive design that explains why WAL exists.

```text
Process A (modern)                         Process B (naive)
─────────────────                          ──────────────────
1. Change pages in RAM                    1. Write table pages to disk now
2. Append WAL in RAM                       2. fsync those random pages
3. COMMIT = fsync the WAL only             3. Then ack the client
4. Ack the client
5. Checkpoint data pages later

Scratch pad → stamp receipt book            Put cash in the drawer on every sale
then later put cash in the drawer          Slow. Random I/O. Still needs a log
                                           for crash safety anyway.
```

| | Process A | Process B |
|--------|-----------|-----------|
| What COMMIT waits for | Sequential log fsync | Random data-page fsyncs |
| Data files at COMMIT | May be stale | Current |
| Crash after COMMIT | REDO the WAL | Pages already there (torn-page risk remains) |
| Used by | PostgreSQL, InnoDB, MongoDB WiredTiger | Almost never in OLTP |

Weaker cousins of Process A (still not Process B): `synchronous_commit = off`, InnoDB `innodb_flush_log_at_trx_commit = 2`, MongoDB's default ~100 ms journal. They ack **before** freeze-frame "WAL on disk." They may **lose** the last few commits on crash. They do not **corrupt**. Details in [12.Durability.md](12.Durability.md).

---

## 4. Same idea in three engines

Setup once. Then the same transfer.

```sql
CREATE TABLE accounts (
  id      INT PRIMARY KEY,
  name    TEXT NOT NULL,
  balance NUMERIC(10,2) NOT NULL
);
INSERT INTO accounts VALUES (1, 'Alice', 100.00), (2, 'Bob', 50.00);
```

### PostgreSQL

Default isolation is `READ COMMITTED`. Default durability waits for WAL fsync (`synchronous_commit = on`).

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100.00
WHERE name = 'Alice' AND balance >= 100.00;

UPDATE accounts
SET balance = balance + 100.00
WHERE name = 'Bob';

COMMIT;
```

After an error inside the block, PostgreSQL **aborts the whole block**. Further SQL is rejected until `ROLLBACK`:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Nobody';
-- you cannot COMMIT your way out of this
ROLLBACK;
```

Savepoints undo part of a still-alive transaction:

```sql
BEGIN;
INSERT INTO audit_log (event) VALUES ('transfer started');
SAVEPOINT before_transfer;
UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
ROLLBACK TO SAVEPOINT before_transfer;  -- debit gone; audit row remains
COMMIT;
```

On `COMMIT`: shared buffers hold dirty pages → `XLogFlush()` fsyncs WAL → client is told success → checkpointer / bgwriter flush data files later.

### Generic SQL / InnoDB (MySQL)

```sql
START TRANSACTION;

UPDATE accounts
SET balance = balance - 100.00
WHERE name = 'Alice' AND balance >= 100.00;

UPDATE accounts
SET balance = balance + 100.00
WHERE name = 'Bob';

COMMIT;
```

Same idea. InnoDB keeps an **undo log** (before-images) as well as a **redo log**. Live `ROLLBACK` physically restores Alice. Crash recovery is ARIES-style: redo everything after the checkpoint, then undo losers. [9-Atomicity.md](9-Atomicity.md) walks that.

### MongoDB

Two levels. Prefer the first.

**One document is already atomic.** Embed Alice and Bob in one document and you do not need a multi-document transaction:

```javascript
db.bank.updateOne(
  {
    _id: "bank_main",
    "accounts.name": "Alice",
    "accounts.balance": { $gte: 100 }
  },
  {
    $inc: {
      "accounts.$[alice].balance": -100,
      "accounts.$[bob].balance": 100
    }
  },
  {
    arrayFilters: [
      { "alice.name": "Alice" },
      { "bob.name": "Bob" }
    ]
  }
);
```

**Multi-document** (separate `accounts` documents) needs a session. Always retry transient errors.

```javascript
const session = db.getMongo().startSession();
try {
  session.startTransaction({
    readConcern:  { level: "snapshot" },
    writeConcern: { w: "majority" }
  });
  const accounts = session.getDatabase("bank").accounts;

  const debit = accounts.updateOne(
    { name: "Alice", balance: { $gte: 100 } },
    { $inc: { balance: -100 } }
  );
  if (debit.modifiedCount !== 1) throw new Error("Alice cannot pay");

  accounts.updateOne({ name: "Bob" }, { $inc: { balance: 100 } });
  session.commitTransaction();
} catch (e) {
  session.abortTransaction();
} finally {
  session.endSession();
}
```

WiredTiger: writes live in cache → one journal commit record on success → abort frees the in-memory buffer (nothing hits the journal) → checkpoint (~60 s) writes `.wt` files.

| Concept | PostgreSQL | InnoDB | MongoDB |
|---------|------------|--------|---------|
| Open a unit of work | `BEGIN` | `START TRANSACTION` | `startTransaction` (or skip, if one document) |
| Memory | shared buffers | buffer pool | WiredTiger cache |
| Log | WAL `pg_wal/` | redo log | journal `WiredTigerLog.*` |
| COMMIT waits for | WAL fsync | redo fsync (`innodb_flush_log_at_trx_commit=1`) | journal sync when `j: true` / majority |
| Data files | checkpoint / bgwriter | checkpoint | checkpoint ~60 s |
| Default isolation | Read Committed | Repeatable Read | snapshot (inside a txn) |
| Single-statement atomicity | autocommit per statement | same | **per document**, always |

---

## 5. Traps + 60-second interview version

### Traps

| Trap | Reality |
|------|---------|
| "`SELECT` in the same session saw the debit, so isolation is broken" | A session always sees its own pencil. Isolation is the **other** terminal. |
| "`COMMIT` wrote the table to disk" | COMMIT wrote the **log**. Table files catch up at checkpoint. |
| "Autocommit means there are no transactions" | Every statement **is** a transaction. You just cannot group two of them. |
| "MongoDB is not transactional" | One document always is. Multi-document needs an explicit session + retry. |
| "I can `COMMIT` after an error in PostgreSQL" | No. The block is dead until `ROLLBACK` (or `ROLLBACK TO SAVEPOINT`). |

### 60 seconds

> *"A transaction is one logical unit of work — here, debit Alice and credit Bob. Either both become committed truth, or the database looks as if neither ran. Autocommit makes each statement its own transaction, which is how a failed second UPDATE can leave Alice poorer and Bob unchanged. BEGIN groups them. ACID is four cameras on that unit: Atomicity is all-or-nothing, Consistency is valid state to valid state, Isolation is what concurrent sessions are allowed to see, Durability is that a COMMIT the client already heard about survives a crash. COMMIT does not mean data files were rewritten. It means the WAL or journal is on disk. Data pages flush later at checkpoint."*

### If they go deeper

| Question | Point them at |
|---------|----------------|
| Half-applied transfer, process still alive vs crash | [9-Atomicity.md](9-Atomicity.md) |
| What T2 sees while T1 has not committed | [10-Isolation.md](10-Isolation.md) |
| Constraints vs "total money = 150" | [11.Consistency.md](11.Consistency.md) |
| `write()` vs `fsync`, freeze-frames of a crash | [12.Durability.md](12.Durability.md) |
| Type it in two terminals | [13-ACID-by-practical-examples.md](13-ACID-by-practical-examples.md) |

### Cheat sheet

1. Transaction = one logical unit of work.
2. Autocommit = one statement, one transaction, committed on success.
3. `BEGIN` … `COMMIT` / `ROLLBACK` groups statements.
4. Committed truth ≠ T1's private view ≠ data files.
5. COMMIT = log durable. Checkpoint = data files catch up.
6. ACID = four cameras on that unit, not four unrelated features.

---

## 6. Sources

- [PostgreSQL: Write-Ahead Logging](https://www.postgresql.org/docs/current/wal-intro.html)
- [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL: Asynchronous Commit](https://www.postgresql.org/docs/current/wal-async-commit.html)
- [MongoDB: Journaling](https://www.mongodb.com/docs/manual/core/journaling/)
- [MongoDB: Transactions](https://www.mongodb.com/docs/manual/core/transactions/)
- [MongoDB: WiredTiger](https://www.mongodb.com/docs/manual/core/wiredtiger/)
- ISO/IEC 9075 — transaction isolation framework
