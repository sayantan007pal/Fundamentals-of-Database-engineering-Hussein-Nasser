# MySQL — Connect to Server + How to Replicate It

This is the MySQL counterpart of `posgres-podman-command.md`.

On this Mac, MySQL is **not** a Podman container. It is **Homebrew `mysql` 26.7.0** running as a launchd service, listening on **`127.0.0.1:3306`**. That is what the Database Client “Connect to server” form is talking to.

> Course notes in `mysql-commands.txt` used a **Docker** dummy (`mysql_dummy`, password `root`, host `192.168.7.122`). That is a different setup. Do **not** start a second MySQL on port `3306` while Homebrew MySQL is already bound there.

---

## 0. What is actually running (this machine)

| Item | Value |
|------|--------|
| Engine | Homebrew **mysql 26.7.0** (`/opt/homebrew/opt/mysql/bin/mysqld`) |
| Service | `brew services` → `mysql` **started** |
| Data dir | `/opt/homebrew/var/mysql` |
| Host | `127.0.0.1` (also `localhost`) |
| Port | `3306` |
| Username | `root` |
| Password | **empty** (Homebrew install with no root password) |
| Database | `mysql` (system catalog DB; always exists) |
| Socket | `/tmp/mysql.sock` |
| Client | `/opt/homebrew/bin/mysql` (same formula) |
| MySQL Shell | `/usr/local/bin/mysqlsh` (optional GUI/shell from Oracle DMG) |

**Verified:**

```text
mysql -h 127.0.0.1 -P 3306 -u root --protocol=TCP mysql
→ root@localhost | mysql | 26.7.0
```

`root` / `Password` and `root` / `root` both fail (`ERROR 1045`). The word **Password** on the Connect form is the **field placeholder**, not the secret. Leave the password box **blank**.

**URI (if the GUI has “Use Connection String”):**

```text
jdbc:mysql://127.0.0.1:3306/mysql
```

CLI equivalent:

```text
mysql://root@127.0.0.1:3306/mysql
```

---

## 1. Map the “Connect to server” form

Fill it exactly like this for the running Homebrew instance:

| GUI field | What to enter |
|-----------|----------------|
| Name / Connection Name | `local-mysql` (any label) |
| Group / Parent/Sub / Scope | leave default / empty |
| Server Type | **MySQL** |
| Config | Config (not connection-string-only, unless you paste the URI) |
| Host | `127.0.0.1` |
| Port | `3306` |
| Username | `root` |
| Password | **leave empty** |
| Database | `mysql` |
| Socket Path | leave empty for TCP, **or** `/tmp/mysql.sock` if you choose socket |
| Features (Event / Trigger) | optional; leave off |
| Use Connection String | off, unless you paste `jdbc:mysql://127.0.0.1:3306/mysql` |
| SSL | **off / disable** (local, no TLS) |

Then **Test** / **Connect**. Premium Event/Trigger toggles are not required for course SQL.

Same values in DBeaver / TablePlus / MySQL Workbench:

| Field | Value |
|-------|--------|
| Host | `127.0.0.1` |
| Port | `3306` |
| User | `root` |
| Password | *(blank)* |
| Database | `mysql` |
| SSL | disable |

---

## 2. Connect from Terminal (already installed)

Interactive:

```bash
mysql -h 127.0.0.1 -P 3306 -u root --protocol=TCP mysql
```

Or via Unix socket (no host/port):

```bash
mysql -u root
```

You should see `mysql>`. Type `exit` to quit.

One-shot query:

```bash
mysql -h 127.0.0.1 -P 3306 -u root --protocol=TCP mysql \
  -e "SELECT CURRENT_USER() AS who, DATABASE() AS db, VERSION() AS ver;"
```

Expected:

```text
who             db     ver
root@localhost  mysql  26.7.0
```

Useful once inside:

```text
SHOW DATABASES;
SHOW TABLES;
SELECT USER(), DATABASE(), VERSION();
exit
```

---

## 3. Sample queries to test

```sql
SELECT CURRENT_USER() AS who, DATABASE() AS db, VERSION() AS ver;

CREATE TABLE IF NOT EXISTS accounts (
  id      VARCHAR(32) PRIMARY KEY,
  balance DECIMAL(12,2) NOT NULL,
  CHECK (balance >= 0)
);

INSERT IGNORE INTO accounts (id, balance) VALUES
  ('Alice', 500.00),
  ('Bob',     0.00);

SELECT * FROM accounts;
```

**Atomic transfer (commit):**

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE id = 'Bob';
SELECT * FROM accounts;
COMMIT;
```

**Atomic transfer (rollback):**

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE id = 'Bob';
ROLLBACK;
SELECT * FROM accounts;
```

Prefer creating a **course database** instead of writing into the system `mysql` DB:

```sql
CREATE DATABASE IF NOT EXISTS acid_lab;
USE acid_lab;
-- then CREATE TABLE accounts ...
```

In the GUI, you can then set **Database** to `acid_lab`.

---

## 4. How to replicate this (Homebrew — matches what is running)

Use this path to get the **same** Connect-to-server dialog working on another Mac, or after a wipe.

### 4.1 Install server + client

```bash
brew install mysql
```

That installs `mysqld` and `mysql` (`mysql  Ver 26.7.0 ...` on this machine). Homebrew starts with **no root password**.

### 4.2 Start the service (listens on 3306)

```bash
brew services start mysql
```

Check:

```bash
brew services list | grep mysql
lsof -nP -iTCP:3306 -sTCP:LISTEN
# expected: mysqld ... TCP *:3306
```

### 4.3 Confirm connect

```bash
mysql -h 127.0.0.1 -P 3306 -u root --protocol=TCP mysql \
  -e "SELECT VERSION();"
```

### 4.4 Fill the GUI

Use the table in **section 1**. Password blank, SSL off, database `mysql`.

### 4.5 Stop / restart / uninstall

```bash
brew services stop mysql     # keep data in /opt/homebrew/var/mysql
brew services start mysql
brew services restart mysql

# remove server (data dir is separate)
brew uninstall mysql
# optional: wipe data
# rm -rf /opt/homebrew/var/mysql
```

---

## 5. Alternate replicate: Podman/Docker dummy (course `mysql-commands.txt`)

Only if you **want** the Udemy dummy container instead of (or in addition to) Homebrew.

**Port clash:** Homebrew MySQL already owns **3306**. Either:

- `brew services stop mysql` first, then publish `3306:3306`, **or**
- keep Homebrew and publish the container on another host port (example: `3307:3306`).

Course original (`docker`; `podman` works the same because of `alias docker=podman`):

```bash
podman run -d \
  --name mysql_dummy \
  -e MYSQL_ROOT_PASSWORD=root \
  -p 3306:3306 \
  mysql:latest
```

Safer on this machine (Homebrew keeps 3306):

```bash
podman volume create mysqldata

podman run -d \
  --name mysql_dummy \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=acid_lab \
  -p 3307:3306 \
  -v mysqldata:/var/lib/mysql \
  mysql:latest
```

Then GUI / CLI:

| Field | Homebrew (current) | Course container (alternate) |
|-------|--------------------|------------------------------|
| Host | `127.0.0.1` | `127.0.0.1` |
| Port | `3306` | `3307` (or `3306` if Homebrew is stopped) |
| Username | `root` | `root` |
| Password | *(empty)* | `root` |
| Database | `mysql` | `acid_lab` or `mysql` |
| SSL | off | off |

Connect inside the container (no local client needed):

```bash
podman exec -it mysql_dummy \
  mysql -u root -proot
```

From the Mac against the published port:

```bash
mysql -h 127.0.0.1 -P 3307 -u root -proot
```

The old notes used a **container-host IP** (`192.168.7.122`) because the dummy ran on another machine. With Podman port publish on this Mac, use **`127.0.0.1`**.

Stop / remove:

```bash
podman stop mysql_dummy
podman start mysql_dummy
podman rm -f mysql_dummy
podman volume rm mysqldata
```

---

## 6. Optional: Oracle MySQL Shell (from course notes)

```text
https://dev.mysql.com/get/Downloads/MySQL-Shell/mysql-shell-26.7.1-macos15-arm64.dmg
```

Already present as `mysqlsh`. Example:

```bash
mysqlsh --host 127.0.0.1 --port 3306 --user root --sql
```

Password: press Enter (empty) for Homebrew; type `root` for the course container.

---

## Quick copy-paste (this machine — Homebrew)

```bash
# 1) confirm service + port
brew services list | grep mysql
lsof -nP -iTCP:3306 -sTCP:LISTEN

# 2) connect (same as Connect to server: 127.0.0.1 / 3306 / root / blank password / mysql)
mysql -h 127.0.0.1 -P 3306 -u root --protocol=TCP mysql
```
