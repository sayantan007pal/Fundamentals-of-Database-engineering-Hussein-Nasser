# PostgreSQL on Podman — Step-by-Step Connect Guide

> **Container:** `pg-dev`  
> **ID:** `832098da827a8fb25f312c65483be40ac5e2a5972cb1a77fdeb77f75526766ea`  
> **Engine:** Podman (`podman.podman-machine-default`)  
> **Image:** `docker.io/library/postgres:16`  
> **State:** running (port `5432:5432`)  
> **Local client:** Homebrew `libpq` → `psql` **18.6** (keg-only; added to `~/.zshrc` PATH)

You can connect **from the Mac** with local `psql`, **or** with `podman exec` (the image already includes `psql`). Both hit the same server.

---

## 0. Connection details (this running instance)

| Item | Value |
|------|--------|
| Container name | `pg-dev` |
| Image | `postgres:16` (PostgreSQL **16.15**) |
| Host port | `5432` → container `5432` |
| Host connect | `127.0.0.1` (also `localhost`) |
| Internal IP (Podman network) | `10.88.0.3` |
| User | `sayantanPosgres` |
| Password | `P@ssw0rd` |
| Database | `Posgres` |
| Data volume | `pgdata` → `/var/lib/postgresql/data` |

**URI (GUI clients / DBeaver / TablePlus / pgAdmin):**

```text
postgresql://sayantanPosgres:P@ssw0rd@127.0.0.1:5432/Posgres
```

**libpq env vars:**

```bash
export PGHOST=127.0.0.1
export PGPORT=5432
export PGUSER=sayantanPosgres
export PGPASSWORD='P@ssw0rd'
export PGDATABASE=Posgres
```

---

## 1. Pull the official image

```bash
podman pull docker.io/library/postgres:16
```

---

## 2. Create a named volume (data survives container recreate)

```bash
podman volume create pgdata
```

---

## 3. Start the container

```bash
podman run -d \
  --name pg-dev \
  -e POSTGRES_USER=sayantanPosgres \
  -e POSTGRES_PASSWORD='P@ssw0rd' \
  -e POSTGRES_DB=Posgres \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
```

If the name already exists:

```bash
podman start pg-dev
```

Check it is up:

```bash
podman ps --filter name=pg-dev
podman port pg-dev
# expected: 5432/tcp -> 0.0.0.0:5432
```

Inspect (same facts as the Podman Desktop "Connect Details" panel):

```bash
podman inspect pg-dev --format 'State={{.State.Status}} Image={{.Config.Image}} IP={{.NetworkSettings.IPAddress}}'
```

---

## 4. Install local `psql` on the Mac (done on this machine)

Homebrew **does not** put `psql` on PATH by default. Install the client only (`libpq`), not a second PostgreSQL server:

```bash
brew install libpq
```

`libpq` is **keg-only** (it conflicts with a full PostgreSQL formula). Add it to PATH in `~/.zshrc`:

```bash
echo 'export PATH="/opt/homebrew/opt/libpq/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Verify:

```bash
which psql
psql --version
# expected: /opt/homebrew/opt/libpq/bin/psql
#           psql (PostgreSQL) 18.6
```

A **18.x client talking to a 16.x server is fine**. You may see a minor version-mismatch notice; ignore it for this course.

If `psql` is still “not found”, the current terminal has not loaded `~/.zshrc` yet — run `source ~/.zshrc` or open a new tab.

---

## 5. Connect — Method A: local `psql` on `127.0.0.1:5432` (recommended now)

```bash
psql -h 127.0.0.1 -p 5432 -U sayantanPosgres -d Posgres
# password when prompted: P@ssw0rd
```

Or skip the prompt with env vars (section 0), then just:

```bash
psql
```

One-shot query (verified on this machine):

```bash
PGPASSWORD='P@ssw0rd' psql -h 127.0.0.1 -p 5432 -U sayantanPosgres -d Posgres \
  -c "SELECT current_user, current_database(), version();"
```

Expected:

```text
  current_user   | current_database | version
-----------------+------------------+---------
 sayantanPosgres | Posgres          | PostgreSQL 16.15 ...
```

Full path if PATH is not loaded yet:

```bash
/opt/homebrew/opt/libpq/bin/psql \
  -h 127.0.0.1 -p 5432 \
  -U sayantanPosgres -d Posgres
```

---

## 6. Connect — Method B: `podman exec` + `psql` inside the container

No local `psql` required. The `postgres:16` image already has `/usr/bin/psql`.

**One-shot query (verified):**

```bash
podman exec -e PGPASSWORD='P@ssw0rd' pg-dev \
  psql -U sayantanPosgres -d Posgres \
  -c "SELECT current_user, current_database(), version();"
```

**Interactive shell:**

```bash
podman exec -it -e PGPASSWORD='P@ssw0rd' pg-dev \
  psql -U sayantanPosgres -d Posgres
```

You should see:

```text
Posgres=#
```

Useful `psql` commands once inside:

```text
\conninfo          -- who/where you are
\l                 -- list databases
\dt                -- list tables in current schema
\q                 -- quit
```

List databases from the host without entering the prompt:

```bash
podman exec -e PGPASSWORD='P@ssw0rd' pg-dev \
  psql -U sayantanPosgres -d Posgres -c "\l"
```

---

## 7. Connect — Method C: GUI “Connect to server” (Database Client / JDBC)

Map the form fields to this instance. Leave **Socket Path** empty (TCP, not a Unix socket). Leave **SSL** off.

| GUI field | Value |
|-----------|--------|
| Name / Connection Name | `pg-dev` (any label) |
| Server Type | PostgreSQL / Config |
| Host | `127.0.0.1` |
| Port | `5432` |
| Username | `sayantanPosgres` |
| Password | `P@ssw0rd` |
| Database | `Posgres` |
| Socket Path | *(leave empty)* |
| SSL | off / disable |
| Use Connection String | optional; if used, paste the URI in section 0 |

Same values work in DBeaver, TablePlus, pgAdmin, and VS Code / Cursor SQL extensions.

---

## 8. Sample queries to test

Paste at the `Posgres=#` prompt, or run with `psql -c "..."`.

**Connectivity:**

```sql
SELECT current_user, current_database(), version();
```

**ACID practice table:**

```sql
CREATE TABLE IF NOT EXISTS accounts (
  id      TEXT PRIMARY KEY,
  balance NUMERIC(12,2) NOT NULL CHECK (balance >= 0)
);

INSERT INTO accounts (id, balance) VALUES
  ('Alice', 500.00),
  ('Bob',     0.00)
ON CONFLICT (id) DO NOTHING;

SELECT * FROM accounts;
```

**Atomic transfer (commit):**

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE id = 'Bob';
SELECT * FROM accounts;
COMMIT;
```

**Atomic transfer (rollback):**

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE id = 'Bob';
ROLLBACK;
SELECT * FROM accounts;  -- back to previous balances
```

---

## 9. Stop / start / remove (when you need it)

```bash
podman stop pg-dev          # keep data in volume pgdata
podman start pg-dev

# destroy container only (volume keeps data)
podman rm -f pg-dev

# destroy data too
podman volume rm pgdata
```

Stopping the container does **not** uninstall local `psql`. Uninstall the client with `brew uninstall libpq` only if you want that.

---

## Quick copy-paste (verified on this machine)

```bash
# 1) confirm container
podman ps --filter name=pg-dev

# 2a) connect from Mac (after brew install libpq + PATH)
psql -h 127.0.0.1 -p 5432 -U sayantanPosgres -d Posgres

# 2b) or connect inside the container (no local client needed)
podman exec -it -e PGPASSWORD='P@ssw0rd' pg-dev \
  psql -U sayantanPosgres -d Posgres
```
