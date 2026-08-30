# ACID by Practical Examples — Lab

> **Course:** Fundamentals of Database Engineering — Sec 2: ACID
> **This file is a lab.** Theory lives in [8](8-what-is-a-transaction.md) · [9](9-Atomicity.md) · [10](10-Isolation.md) · [11](11.Consistency.md) · [12](12.Durability.md).
> **Every letter is typed in PostgreSQL, InnoDB, and MongoDB.**

> **Rule:** Isolation and in-flight atomicity need **two persistent terminals**. Do not use the VS Code SQL extension for Labs A and I (connection pooling + autocommit will lie).

---

## 0. After this note you can...

- Before you type, **predict** Session 2's snapshot in the ASCII box.
- After you type, name **which ACID letter** that snapshot proved.
- If the grid is wrong, know whether you were on the **same session**, **autocommit**, or the **wrong isolation level**.

---

## 1. The one picture

Checkout is one transaction: insert a **sale** and decrement **inventory**.

```text
START (after Lab 0)
committed truth     Product 1 inventory 100    sales 0
Session 1 (cashier)  same
Session 2 (manager)  same
```

Connections: [posgres-podman-command.md](posgres-podman-command.md) · [mysql-connect-command.md](mysql-connect-command.md). Two windows. Label them **Session 1 = cashier**, **Session 2 = manager**. Confirm different backends: `SELECT pg_backend_pid();` / `SELECT CONNECTION_ID();`.

---

## 2. Why the original demo failed

```sql
UPDATE products SET inventory = inventory - 10;   -- no WHERE, no BEGIN
\q                                                  -- psql-only; not ROLLBACK
SELECT * FROM products;                             -- same session / already committed
```

| Bug | What happened |
|-----|----------------|
| No `BEGIN` | Autocommit: each statement is its own transaction |
| VS Code client | Pooling; `\q` ignored |
| Same-session `SELECT` | You always see your own pencil. Isolation is Session 2 |
| No `WHERE` | Deducts 10 from **every** product |
| `\q` as atomicity | Client disconnect, not an engine proof |
| Consistency = locking | Wrong letter. C is valid warehouse state |

---

## 3. Lab 0 — Setup (all three engines)

Reset **before every lab**.

### PostgreSQL

```sql
DROP TABLE IF EXISTS sales;
DROP TABLE IF EXISTS products;

CREATE TABLE products (
    pid        SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    price      NUMERIC(10,2) NOT NULL CHECK (price > 0),
    inventory  INT NOT NULL CHECK (inventory >= 0)
);
CREATE TABLE sales (
    sale_id  SERIAL PRIMARY KEY,
    pid      INT NOT NULL REFERENCES products(pid),
    quantity  INT NOT NULL CHECK (quantity > 0),
    price    NUMERIC(10,2) NOT NULL CHECK (price > 0)
);
INSERT INTO products (name, price, inventory) VALUES
  ('Product 1', 100.99, 100),
  ('Product 2', 200.99, 200),
  ('Product 3', 300.99, 300);
```

Reset: `TRUNCATE sales RESTART IDENTITY;` then set inventories back to 100/200/300.

### InnoDB

```sql
CREATE DATABASE IF NOT EXISTS acid_lab;
USE acid_lab;

CREATE TABLE products (
    pid INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    inventory INT NOT NULL CHECK (inventory >= 0)
) ENGINE=InnoDB;
CREATE TABLE sales (
    sale_id INT AUTO_INCREMENT PRIMARY KEY,
    pid INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    FOREIGN KEY (pid) REFERENCES products(pid)
) ENGINE=InnoDB;
-- same INSERT as PostgreSQL
```

Safer reset: `DELETE FROM sales;` then `UPDATE products SET inventory = CASE pid WHEN 1 THEN 100 WHEN 2 THEN 200 WHEN 3 THEN 300 END;`

### MongoDB

Replica set required for multi-document transactions.

```javascript
use acid_lab
db.products.drop();
db.sales.drop();
db.createCollection("products", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["_id", "name", "price", "inventory"],
      properties: {
        name: { bsonType: "string" },
        price: { bsonType: "double", minimum: 0.01 },
        inventory: { bsonType: "int", minimum: 0 }
      }
    }
  }
});
db.products.insertMany([
  { _id: 1, name: "Product 1", price: 100.99, inventory: NumberInt(100) },
  { _id: 2, name: "Product 2", price: 200.99, inventory: NumberInt(200) },
  { _id: 3, name: "Product 3", price: 300.99, inventory: NumberInt(300) }
]);
```

Reset: `db.sales.deleteMany({});` then set inventories back.

**SEE after setup:** Product 1 = 100, sales count = 0.

---

## 4. Lab A — Atomicity

**Predict before typing:**

```text
After Session 1 inserts a sale and decrements stock, BEFORE COMMIT
committed / Session 2     inventory 100    sales 0
Session 1                 inventory  90    sales 1

After ROLLBACK (or abort)
committed / Session 2     inventory 100    sales 0
```

If Session 2 already shows 90, you autocommitted or used one session. Stop.

### PostgreSQL

Session 1:

```sql
BEGIN;
INSERT INTO sales (pid, quantity, price) VALUES (1, 10, 100.99);
UPDATE products SET inventory = inventory - 10 WHERE pid = 1 AND inventory >= 10;
SELECT inventory FROM products WHERE pid = 1;  -- 90 (own pencil)
```

Session 2 (Session 1 still in `BEGIN`):

```sql
SELECT inventory FROM products WHERE pid = 1;  -- must be 100
SELECT count(*) FROM sales;                     -- 0
```

Session 1: `SELECT 1/0;` then `ROLLBACK;`  
Session 2 again: **100** and **0**.

Happy path: same two writes, `COMMIT;` Session 2 sees **90** and **one** sale.

### InnoDB

```sql
START TRANSACTION;
INSERT INTO sales (pid, quantity, price) VALUES (1, 10, 100.99);
UPDATE products SET inventory = inventory - 10 WHERE pid = 1 AND inventory >= 10;
-- Session 2 still 100 / 0
INSERT INTO sales (pid, quantity, price) VALUES (999, 1, 1.00);  -- FK error
ROLLBACK;
-- Session 2: 100 / 0   (undo log reversed both writes)
```

### MongoDB

```javascript
// Session 1
const s1 = db.getMongo().startSession();
s1.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
});
const shop = s1.getDatabase("acid_lab");
shop.sales.insertOne({ pid: 1, quantity: 10, price: 100.99 });
shop.products.updateOne({ _id: 1, inventory: { $gte: 10 } }, { $inc: { inventory: -10 } });
shop.products.findOne({ _id: 1 });  // 90

// Session 2 — no s1
db.products.findOne({ _id: 1 });     // 100
db.sales.countDocuments();            // 0

s1.abortTransaction();
s1.endSession();
// Session 2 still 100 / 0
```

**If you saw X / you proved Y:**

| You saw | Meaning |
|---------|---------|
| Session 2: 100 / 0 then after ROLLBACK still 100 / 0 | **Atomicity** (and isolation of pencil) |
| Session 2: 90 before COMMIT | Same session or autocommit — not a proof |
| Sale row, inventory 100 after COMMIT | You committed only the insert — A broken |

---

## 5. Lab C — Consistency

**Predict:** illegal sale / negative stock is **rejected**. Committed warehouse stays legal.

### PostgreSQL

```sql
INSERT INTO sales (pid, quantity, price) VALUES (999, 1, 1.00);
-- ERROR: foreign key  (ghost SKU)

UPDATE products SET inventory = inventory - 1000 WHERE pid = 1;
-- ERROR: check constraint inventory >= 0
SELECT inventory FROM products WHERE pid = 1;  -- still 100
```

After a correct checkout (Lab A happy path): `sold 10 + remaining 90 = 100`.

### InnoDB

Same FK error (`1452`) and CHECK on `inventory >= 0`. `ENGINE=InnoDB` required for FK.

Two cashiers overselling: CHECK does **not** serialize them. Use:

```sql
UPDATE products SET inventory = inventory - 10 WHERE pid = 1 AND inventory >= 10;
-- 0 rows → refuse the sale
```

### MongoDB

```javascript
db.sales.insertOne({ pid: 999, quantity: 1, price: 1.00 });
// succeeds — no FK. Data-C is your job. Prefer embed:

db.products.updateOne(
  { _id: 1, inventory: { $gte: 10 } },
  { $inc: { inventory: -10 }, $push: { sales: { qty: 10, price: 100.99 } } }
);
// remaining + sold stays inside one document
```

Negative inventory: `$jsonSchema` `minimum: 0` rejects the update.

**If you saw X:** ghost SKU inserted in MongoDB without a txn → you proved there is **no FK**, not that C is optional.

---

## 6. Lab I — Isolation

**Predict:**

```text
Session 1 inside BEGIN, inventory 90 on its scratch pad
Session 2 SELECT                      inventory 100    sales 0     (RC+)
Session 1 COMMIT
Session 2 SELECT                      inventory  90    sales 1
```

### PostgreSQL (cannot dirty-read)

Session 1: `BEGIN;` insert sale + decrement (as Lab A).  
Session 2: `SELECT` → **100** / **0**.  
Session 1: `COMMIT;` Session 2: **90** / **1**.

Non-repeatable read (default RC):

```sql
-- Session 1
BEGIN;
SELECT inventory FROM products WHERE pid = 1;  -- 100
-- Session 2: UPDATE ... - 10; COMMIT;
SELECT inventory FROM products WHERE pid = 1;  -- 90
COMMIT;

BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT inventory FROM products WHERE pid = 1;  -- 100
-- Session 2 commits 90
SELECT inventory FROM products WHERE pid = 1;  -- still 100
COMMIT;
```

### InnoDB

Same two-session dance with `START TRANSACTION`. Default RR already hides dirty reads.

Optional dirty read (InnoDB only):

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT inventory FROM products WHERE pid = 1;  -- may be 90 while Session 1 uncommitted
ROLLBACK;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### MongoDB

Session 1: open snapshot txn, write, **do not** commit.  
Session 2 without that session: still **100**.  
`commitTransaction` → Session 2 sees **90**.

**If you saw 90 in Session 2 before COMMIT:** you used Session 1, or RU on InnoDB.

---

## 7. Lab D — Durability

You cannot pull the plug in this lab. You **inspect which freeze-frame COMMIT waits for** ([12.Durability.md](12.Durability.md)).

**Predict:** financial checkout uses Frame 3 (log on media), not Frame 2 (OS cache only).

### PostgreSQL

```sql
SHOW fsync;                  -- on
SHOW synchronous_commit;     -- on  → COMMIT waits for WAL fsync
BEGIN;
UPDATE products SET inventory = inventory - 10 WHERE pid = 1 AND inventory >= 10;
INSERT INTO sales (pid, quantity, price) VALUES (1, 10, 100.99);
COMMIT;
```

### InnoDB

```sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';  -- 1 = ACID
START TRANSACTION;
-- same checkout
COMMIT;
```

### MongoDB

```javascript
db.products.updateOne(
  { _id: 1, inventory: { $gte: 10 } },
  { $inc: { inventory: -10 } },
  { writeConcern: { w: "majority", j: true } }
);
db.adminCommand({ getParameter: 1, journalCommitInterval: 1 });  // ~100 ms if j:false
```

**If `synchronous_commit=off` / `flush_log=2` / `j:false`:** you accepted possible **loss** of this checkout on crash, not corruption.

---

## 8. Checkout API (all three)

| Do | PostgreSQL / SQL | MongoDB |
|----|------------------|---------|
| One txn for sale + stock | `BEGIN` … both … `COMMIT` | Embed `$inc`+$push, or snapshot txn |
| Conditional stock | `WHERE inventory >= qty` | `{ inventory: { $gte: qty } }` |
| After error | `ROLLBACK` (PG block is dead) | `abortTransaction`; retry transient |
| Money / stock reads | Primary, sync commit | Primary, `w: majority, j: true` |

Never: autocommit insert then later update inventory. Never: HTTP inside `BEGIN`.

### 60 seconds

> *"I prove ACID with products and sales in two terminals. Isolation is what Session 2 does not see before COMMIT. Atomicity is that pair appearing or vanishing together. Consistency is FK and CHECK rejecting ghost SKUs and negative stock — plus an atomic WHERE inventory >= qty so two cashiers cannot oversell. Durability is COMMIT waiting for WAL, redo, or journal fsync. Same recipe in PostgreSQL, InnoDB, and MongoDB; MongoDB has no FK so I embed or check in the transaction."*

---

## 9. Sources

- Connection notes in this folder
- Theory files 8–12
- [PostgreSQL transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)
- [InnoDB isolation](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- [MongoDB transactions](https://www.mongodb.com/docs/manual/core/transactions/)
