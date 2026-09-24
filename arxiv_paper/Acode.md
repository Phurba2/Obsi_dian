| `furba@fu:~$ git clone https://github.com/pgvector/pgvector.git`                        |
| --------------------------------------------------------------------------------------- |
| `Cloning into 'pgvector'...`                                                            |
| `remote: Enumerating objects: 13558, done.`                                             |
| `remote: Counting objects: 100% (3839/3839), done.`                                     |
| `remote: Compressing objects: 100% (388/388), done.`                                    |
| `remote: Total 13558 (delta 3575), reused 3452 (delta 3451), pack-reused 9719 (from 4)` |
| `Receiving objects: 100% (13558/13558), 2.21 MiB                                        |
| `Resolving deltas: 100% (10112/10112), done.`                                           |
| `furba@fu:~$ cd pgvector`                                                               |
| furba@fu:~/pgvector$ make                                                               |
| error: postgres.h: No such file or directory                                            |

---


```text
fatal error: postgres.h: No such file or directory
```

`postgres.h` **PostgreSQL server development header which files are missing or Postgres 18 isn't installed correclty**

```text
/usr/include/postgresql/18/server
```


```bash
psql --version
```


```bash
pg_config --version
```


```bash
pg_config --includedir-server
```

You should get something like:

```text
PostgreSQL 18.x
/usr/include/postgresql/18/server
```

---


```bash
sudo apt update
sudo apt install postgresql-server-dev-18
```


```bash
ls /usr/include/postgresql/18/server/postgres.h
```

If everything is correct, you should see:

```text
/usr/include/postgresql/18/server/postgres.h
```

---

```bash
cd ~/pgvector
```

```bash
make clean
make
```

```bash
sudo make install
```

---

```bash
psql
```

```sql
CREATE EXTENSION vector;
```

```sql
SELECT extversion FROM pg_extension WHERE extname = 'vector';
```


```text
 extversion
------------
 0.x.x
```

---

```text
pgvector source code
       │
       │ make
       ▼
    gcc compiler
       │
       │ needs
       ▼
PostgreSQL development files
       │
       └── postgres.h  ← MISSING
```


```bash
sudo apt install postgresql-server-dev-18
```

provide files needed to compile PostgreSQL extensions such as `pgvector`.

---

| sudo make install                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/bin/mkdir -p '/usr/lib/postgresql/18/lib'`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `/bin/mkdir -p '/usr/share/postgresql/18/extension'`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `/bin/mkdir -p '/usr/share/postgresql/18/extension'`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `/usr/bin/install -c -m 755  vector.so '/usr/lib/postgresql/18/lib/vector.so'`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `/usr/bin/install -c -m 644 .//vector.control '/usr/share/postgresql/18/extension/'`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `/usr/bin/install -c -m 644 .//sql/vector--0.1.0--0.1.1.sql .//sql/vector--0.1.1--0.1.3.sql .//sql/vector--0.1.3--0.1.4.sql .//sql/vector--0.1.4--0.1.5.sql .//sql/vector--0.1.5--0.1.6.sql .//sql/vector--0.1.6--0.1.7.sql .//sql/vector--0.1.7--0.1.8.sql .//sql/vector--0.1.8--0.2.0.sql .//sql/vector--0.2.0--0.2.1.sql .//sql/vector--0.2.1--0.2.2.sql .//sql/vector--0.2.2--0.2.3.sql .//sql/vector--0.2.3--0.2.4.sql .//sql/vector--0.2.4--0.2.5.sql .//sql/vector--0.2.5--0.2.6.sql .//sql/vector--0.2.6--0.2.7.sql .//sql/vector--0.2.7--0.3.0.sql .//sql/vector--0.3.0--0.3.1.sql .//sql/vector--0.3.1--0.3.2.sql .//sql/vector--0.3.2--0.4.0.sql .//sql/vector--0.4.0--0.4.1.sql .//sql/vector--0.4.1--0.4.2.sql .//sql/vector--0.4.2--0.4.3.sql .//sql/vector--0.4.3--0.4.4.sql .//sql/vector--0.4.4--0.5.0.sql .//sql/vector--0.5.0--0.5.1.sql .//sql/vector--0.5.1--0.6.0.sql .//sql/vector--0.6.0--0.6.1.sql .//sql/vector--0.6.1--0.6.2.sql .//sql/vector--0.6.2--0.7.0.sql .//sql/vector--0.7.0--0.7.1.sql .//sql/vector--0.7.1--0.7.2.sql .//sql/vector--0.7.2--0.7.3.sql .//sql/vector--0.7.3--0.7.4.sql .//sql/vector--0.7.4--0.8.0.sql .//sql/vector--0.8.0--0.8.1.sql .//sql/vector--0.8.1--0.8.2.sql .//sql/vector--0.8.2--0.8.3.sql .//sql/vector--0.8.3--0.8.4.sql .//sql/vector--0.8.4--0.8.5.sql .//sql/vector--0.8.5--0.8.6.sql .//sql/vector--0.8.6--0.8.7.sql sql/vector--0.8.6.sql '/usr/share/postgresql/18/extension/'` |
| `/bin/mkdir -p '/usr/include/postgresql/18/server/extension/vector/'`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `/usr/bin/install -c -m 644   .//src/halfvec.h .//src/sparsevec.h .//src/vector.h '/usr/include/postgresql/18/server/extension/vector/'`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `/bin/mkdir -p '/usr/lib/postgresql/18/lib/bitcode/vector'`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `/bin/mkdir -p '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `/usr/bin/install -c -m 644 src/bitutils.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `/usr/bin/install -c -m 644 src/bitvec.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `/usr/bin/install -c -m 644 src/halfutils.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `/usr/bin/install -c -m 644 src/halfvec.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `/usr/bin/install -c -m 644 src/hnsw.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `/usr/bin/install -c -m 644 src/hnswbuild.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `/usr/bin/install -c -m 644 src/hnswinsert.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `/usr/bin/install -c -m 644 src/hnswscan.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `/usr/bin/install -c -m 644 src/hnswutils.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `/usr/bin/install -c -m 644 src/hnswvacuum.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `/usr/bin/install -c -m 644 src/ivfbuild.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `/usr/bin/install -c -m 644 src/ivfflat.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `/usr/bin/install -c -m 644 src/ivfinsert.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `/usr/bin/install -c -m 644 src/ivfkmeans.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `/usr/bin/install -c -m 644 src/ivfscan.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `/usr/bin/install -c -m 644 src/ivfutils.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `/usr/bin/install -c -m 644 src/ivfvacuum.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `/usr/bin/install -c -m 644 src/sparsevec.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `/usr/bin/install -c -m 644 src/vector.bc '/usr/lib/postgresql/18/lib/bitcode'/vector/src/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `cd '/usr/lib/postgresql/18/lib/bitcode' && /usr/lib/llvm-19/bin/llvm-lto -thinlto -thinlto-action=thinlink -o vector.index.bc vector/src/bitutils.bc vector/src/bitvec.bc vector/src/halfutils.bc vector/src/halfvec.bc vector/src/hnsw.bc vector/src/hnswbuild.bc vector/src/hnswinsert.bc vector/src/hnswscan.bc vector/src/hnswutils.bc vector/src/hnswvacuum.bc vector/src/ivfbuild.bc vector/src/ivfflat.bc vector/src/ivfinsert.bc vector/src/ivfkmeans.bc vector/src/ivfscan.bc vector/src/ivfutils.bc vector/src/ivfvacuum.bc vector/src/sparsevec.bc vector/src/vector.bc`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `furba@fu:~/pgvector$`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

---

# ✅ pgvector installed successfully

That output shows that:

```bash
sudo make install
```

completed **without errors**.

The important lines are:

```text
/usr/bin/install -c -m 755 vector.so \
'/usr/lib/postgresql/18/lib/vector.so'
```

and:

```text
'/usr/share/postgresql/18/extension/vector.control'
```

So PostgreSQL 18 can now load `pgvector` extension.

---

## 1. Enable pgvector in your database

First connect to PostgreSQL:

```bash
psql
```


```text
psql (18.x)
Type "help" for help.

postgres=#
```


```sql
CREATE EXTENSION vector;
```

```text
CREATE EXTENSION
```

---

## 2. Verify


```sql
\dx
```


```text
             List of installed extensions
   Name   | Version |   Schema   |        Description
----------+---------+------------+-------------------------
 plpgsql  | 1.0     | pg_catalog | PL/pgSQL procedural language
 vector   | 0.8.6   | public     | vector data type and ...
```

You can also run:

```sql
SELECT extversion
FROM pg_extension
WHERE extname = 'vector';
```

---

# 3. Test actual vector

Create a small test table:

```sql
CREATE TABLE test_vectors (
    id SERIAL PRIMARY KEY,
    text TEXT,
    embedding vector(3)
);
```

Insert some vectors:

```sql
INSERT INTO test_vectors (text, embedding)
VALUES
    ('hello', '[1,0,0]'),
    ('good morning', '[0.9,0.1,0]'),
    ('cat', '[0,0,1]');
```

Then search for vector closest to:

```text
[1,0,0]
```

using cosine distance:

```sql
SELECT
    text,
    embedding <=> '[1,0,0]' AS distance
FROM test_vectors
ORDER BY embedding <=> '[1,0,0]'
LIMIT 3;
```

You should get `hello` and `good morning`.

---

| `furba@fu:~$ psql`          |
| --------------------------- |
| role "furba" does not exist |

---


```text
PostgreSQL server
      │
      ├── running ✅
      │
      └── PostgreSQL user "furba" ❌ doesn't exist
```

---

```bash
sudo -u postgres psql
```


```text
psql (18.x)
Type "help" for help.

postgres=#
```

---

```sql
\du
```


```text
                             List of roles
 Role name |                         Attributes
-----------+------------------------------------------------
 postgres  | Superuser, Create role, Create DB, Replication
```

---

```sql
CREATE ROLE furba WITH LOGIN CREATEDB PASSWORD 'your_password';
```


```sql
\du
```


```text
postgres
furba
```


```sql
\q
```

---

```bash
psql
```

```text
furba
```

---

```sql
CREATE DATABASE ragdb;
```


```sql
\q
```

```bash
psql -d ragdb
```

```text
ragdb=#
```

---

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

```text
CREATE EXTENSION
```

```sql
SELECT *
FROM pg_extension
WHERE extname = 'vector';
```

```text
extname | vector
```

---

```text
Ubuntu
  │
  │ sudo -u postgres psql
  ▼
PostgreSQL administrator
  │
  ├── CREATE ROLE furba
  │
  └── CREATE DATABASE ragdb
                │
                ▼
          psql -d ragdb
                │
                ▼
       CREATE EXTENSION vector
                │
                ▼
          PostgreSQL + pgvector
```

---

```bash
sudo make install
```

does **not automatically enable it in every database**.

You enable it separately in each database:

```sql
CREATE EXTENSION vector;
```

---

| `furba@fu:~$ psql`                                                      |
| ----------------------------------------------------------------------- |
| database "furba" does not exist                                         |
| furba@fu:~$ sudo -u postgres psql                                       |
| psql (18.6 (Ubuntu 18.6-1.pgdg24.04+2))                                 |
| postgres=# \du                                                          |
| List of roles                                                           |
| Role name \| Attributes   \|                                            |
| furba           \| Create DB                                            |
| postgres  \| Superuser, Create role, Create DB, Replication, Bypass RLS |
| postgres=#                                                              |

---

```bash
psql
```

PostgreSQL tries to connect :

```text
user     = furba
database = furba
```

You have **user**, but not a **database named `furba`**.

---

```sql
CREATE DATABASE furba OWNER furba;
```


```text
CREATE DATABASE
```


```sql
\q
```


```bash
psql
```


```text
psql (18.6 ...)
Type "help" for help.

furba=#
```

---

From `furba=#`:

```sql
CREATE DATABASE ragdb;
```

connect :

```sql
\c ragdb
```

```text
You are now connected to database "ragdb" as user "furba".
```
enable pgvector:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

```text
CREATE EXTENSION
```


```sql
SELECT * FROM pg_extension WHERE extname = 'vector';
```

---

```text
PostgreSQL 18
│
├─ Role
│    └── furba
│─ Database
    ├── furba
    └── ragdb
          │
          └── pgvector ✅
```

---

| furba=> CREATE DATABASE ragdb;                             |
| ---------------------------------------------------------- |
| CREATE DATABASE                                            |
| furba=> \c ragdb                                           |
| You are now connected to database "ragdb" as user "furba". |
| ragdb=> CREATE EXTENSION IF NOT EXISTS vector;             |
| ERROR:  permission denied to create extension "vector"     |
| HINT:  Must be superuser to create this extension.         |
| ragdb=>                                                    |

---

```bash
sudo -u postgres psql -d ragdb
```



```text
ragdb=#
```

`#` instead of `>` — you're connected as  superuser.

---

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

```text
CREATE EXTENSION
```

---

```sql
SELECT * FROM pg_extension WHERE extname = 'vector';
```


```text
 oid  | extname | extowner | extnamespace | extrelocatable | extversion
------+---------+----------+--------------+----------------+-----------
 ...  | vector  | ...      | ...          | t              | 0.8.6
```

---

```sql
\q
```


```bash
psql -d ragdb
```


```text
ragdb=>
```


```sql
SELECT '[1,2,3]'::vector;
```


```text
 vector
---------
 [1,2,3]
```

---

```text
                  PostgreSQL
                      │
          ┌───────────┴───────────┐
          │                       │
       INSTALL                 ENABLE
       pgvector                pgvector
          │                       │
    sudo make install       CREATE EXTENSION
          │                       │
          ▼                       ▼
   Server has vector.so      ragdb has vector
```

You already completed **installation**:

```text
/usr/lib/postgresql/18/lib/vector.so
```

**enable it in `ragdb`**

```sql
CREATE EXTENSION vector;
```

---

