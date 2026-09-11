# SQL & PostgreSQL Fundamentals

## Table of Contents

1. [Why PostgreSQL?](#1-why-postgresql)
2. [PostgreSQL Setup](#2-postgresql-setup)
3. [Relational Database Fundamentals](#3-relational-database-fundamentals)
4. [SQL Basics: DDL and DML](#4-sql-basics-ddl-and-dml)
5. [Data Types](#5-data-types)
6. [Primary Keys](#6-primary-keys)
7. [Constraints](#7-constraints)
8. [Foreign Keys and Relationships](#8-foreign-keys-and-relationships)
9. [CRUD Operations in Depth](#9-crud-operations-in-depth)
10. [Filtering and Sorting](#10-filtering-and-sorting)
11. [JOINs](#11-joins)
12. [Aggregation](#12-aggregation)
13. [Transactions](#13-transactions)
14. [Isolation Levels](#14-isolation-levels)
15. [Pagination](#15-pagination)
16. [Indexes](#16-indexes)
17. [SQL Injection](#17-sql-injection)
18. [Connection Pooling](#18-connection-pooling)
19. [Connecting Go to PostgreSQL](#19-connecting-go-to-postgresql)
20. [Modern Practices](#20-modern-practices)
21. [Common Mistakes](#21-common-mistakes)
22. [Exercises](#22-exercises)
23. [Key Takeaways](#23-key-takeaways)

---

## 1. Why PostgreSQL?

PostgreSQL (often called Postgres) is an open-source relational database that has
been in active development since 1986. It is the most capable open-source RDBMS
available today and the default choice for Go backend applications.

Key strengths:

- **ACID compliance**: Atomicity, Consistency, Isolation, Durability are
  guaranteed by default.
- **Extensibility**: Custom types, functions, operators, and extensions (PostGIS
  for geospatial, pg_trgm for fuzzy text search, pg_cron for scheduled jobs).
- **Advanced data types**: Native JSONB, arrays, hstore (key-value), range types,
  geometric types, and network address types.
- **MVCC (Multi-Version Concurrency Control)**: Readers do not block writers and
  writers do not block readers. High concurrency without locking contention.
- **Standards compliance**: Follows the SQL standard more closely than MySQL,
  Oracle, or SQL Server.
- **Mature ecosystem**: Battle-tested replication, backup, partitioning, and
  monitoring tools. Every major cloud provider offers managed PostgreSQL.

---

## 2. PostgreSQL Setup

### Installation

**macOS** (Homebrew):

```bash
brew install postgresql@16
brew services start postgresql@16
```

**Ubuntu/Debian**:

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**Docker** (fastest for development):

```bash
docker run -d \
  --name postgres-dev \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  postgres:16-alpine
```

### psql basics

```bash
psql -U admin -d myapp -h localhost
```

| Command          | Purpose                              |
|------------------|--------------------------------------|
| `\l`             | List all databases                   |
| `\c dbname`      | Connect to a database                |
| `\dt`            | List tables in the current schema    |
| `\d tablename`   | Describe a table's columns and indexes|
| `\du`            | List database roles/users            |
| `\di`            | List indexes                         |
| `\dn`            | List schemas                         |
| `\q`             | Quit psql                            |
| `\x`             | Toggle expanded output               |
| `\timing`        | Toggle query timing                  |

### Creating a database

```sql
CREATE DATABASE shop;
\c shop
```

---

## 3. Relational Database Fundamentals

A relational database stores data in **tables** (also called relations). Each
table has **columns** (attributes) and **rows** (records).

```
+----+-----------+-------+---------------------+
| id | name      | price | created_at          |
+----+-----------+-------+---------------------+
|  1 | Keyboard  | 79.99 | 2026-01-15 10:30:00 |
|  2 | Mouse     | 29.99 | 2026-01-15 11:00:00 |
|  3 | Monitor   | 349.99| 2026-01-16 09:15:00 |
+----+-----------+-------+---------------------+
```

### Relationships

Tables relate to each other through keys:

- **One-to-one**: A user has one profile. The `profiles` table has a `user_id`
  column with a UNIQUE constraint.
- **One-to-many**: A customer has many orders. The `orders` table has a
  `customer_id` column referencing the `customers` table.
- **Many-to-many**: Students enroll in many courses; courses have many students.
  This requires a **junction table** (`enrollments`) with foreign keys to both
  tables.

> 🧠 **Memory aid:** A junction table turns one messy many-to-many into two clean one-to-many relationships. It is the SQL version of "introduce a middleman".

```mermaid
erDiagram
    CUSTOMERS ||--o{ ORDERS : places
    ORDERS ||--|{ ORDER_ITEMS : contains
    PRODUCTS ||--o{ ORDER_ITEMS : included_in
    STUDENTS }o--o{ COURSES : enrolls
```

### Schema

The structure of your tables — column names, data types, constraints, indexes —
is the **schema**. In PostgreSQL, the default schema is `public`. You modify the
schema with DDL statements like `CREATE TABLE`, `ALTER TABLE`, and `DROP TABLE`.

---

## 4. SQL Basics: DDL and DML

- **DDL (Data Definition Language)**: `CREATE TABLE`, `ALTER TABLE`,
  `DROP TABLE`, `CREATE INDEX`.
- **DML (Data Manipulation Language)**: `SELECT`, `INSERT`, `UPDATE`, `DELETE`.
- **DCL (Data Control Language)**: `GRANT`, `REVOKE`.
- **TCL (Transaction Control Language)**: `BEGIN`, `COMMIT`, `ROLLBACK`.

> 🔑 **Key idea:** Remember the split by asking: "am I changing the *shape* or the *contents*?" DDL edits the schema (tables/indexes); DML edits the data (rows); TCL controls when DML becomes permanent.

### CREATE TABLE

```sql
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    email      VARCHAR(255) NOT NULL UNIQUE,
    name       VARCHAR(100) NOT NULL,
    is_active  BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT now()
);
```

- `SERIAL PRIMARY KEY`: Auto-incrementing integer uniquely identifying each row.
- `VARCHAR(255) NOT NULL UNIQUE`: String up to 255 characters, required, unique.
- `BOOLEAN DEFAULT true`: Boolean that defaults to `true`.
- `TIMESTAMP DEFAULT now()`: Defaults to the current time.

### INSERT

```sql
-- Insert a single row
INSERT INTO users (email, name) VALUES ('alice@example.com', 'Alice');

-- Insert multiple rows
INSERT INTO users (email, name) VALUES
    ('bob@example.com', 'Bob'),
    ('carol@example.com', 'Carol');

-- Insert and return the generated id
INSERT INTO users (email, name) VALUES ('dave@example.com', 'Dave')
    RETURNING id;
```

### SELECT

```sql
SELECT * FROM users;
SELECT id, name, email FROM users;
SELECT * FROM users WHERE is_active = true;
SELECT DISTINCT name FROM users;
```

### UPDATE

```sql
UPDATE users SET name = 'Alice Smith' WHERE id = 1;
UPDATE users SET name = 'Bob Jones', is_active = false WHERE id = 2;
UPDATE users SET is_active = true;  -- all rows (use with caution)
```

### DELETE

```sql
DELETE FROM users WHERE id = 3;
DELETE FROM users WHERE is_active = false;
DELETE FROM users;
TRUNCATE users;  -- faster for deleting all rows
```

---

## 5. Data Types

| Type               | Storage     | Use Case                              |
|--------------------|-------------|---------------------------------------|
| `SMALLINT`         | 2 bytes     | Small numbers (-32768 to 32767)       |
| `INTEGER`          | 4 bytes     | Standard integers (-2B to 2B)         |
| `BIGINT`           | 8 bytes     | Large integers, counters              |
| `SERIAL`           | 4 bytes     | Auto-increment INTEGER                |
| `BIGSERIAL`        | 8 bytes     | Auto-increment BIGINT                 |
| `NUMERIC(p,s)`     | variable    | Exact decimal (money, measurements)   |
| `BOOLEAN`          | 1 byte      | True/false values                     |
| `VARCHAR(n)`       | 1-255+ bytes| Variable string with max length       |
| `TEXT`             | variable    | Unlimited length string               |
| `TIMESTAMPTZ`      | 8 bytes     | Date and time with timezone           |
| `UUID`             | 16 bytes    | Universally unique identifiers        |
| `JSONB`            | variable    | Binary JSON, indexable, queryable     |

### VARCHAR vs TEXT

In PostgreSQL there is no performance difference. Use `VARCHAR(n)` when you have
a meaningful maximum length (email, usernames). Use `TEXT` when there is no
natural limit.

### UUID

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE articles (
    id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title      TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

### JSONB

```sql
CREATE TABLE events (
    id         SERIAL PRIMARY KEY,
    payload    JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

INSERT INTO events (payload) VALUES ('{"type": "click", "page": "/home"}');

SELECT * FROM events WHERE payload @> '{"type": "click"}';
SELECT payload->>'page' AS page FROM events;
SELECT payload->'user'->>'name' AS user_name FROM events;
```

| Operator | Purpose                              |
|----------|--------------------------------------|
| `->`     | Get JSON object field (returns JSON) |
| `->>`    | Get JSON object field (returns text) |
| `#>`     | Get path (returns JSON)              |
| `#>>`    | Get path (returns text)              |
| `@>`     | Contains                             |
| `<@`     | Is contained by                      |
| `?`      | Key exists                           |
| `?|`     | Any key exists                       |

> 💡 **Pro tip:** Unlike plain `JSON`, `JSONB` is stored in a binary format and can be indexed with a GIN index — so you can run fast `@>` "contains" queries on stored documents.

---

## 6. Primary Keys

### SERIAL and BIGSERIAL

```sql
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    total       NUMERIC(10, 2) NOT NULL
);
```

Internally, PostgreSQL creates a sequence named `orders_id_seq` and sets the
column default to `nextval('orders_id_seq')`.

Standard identity columns (PostgreSQL 10+):

```sql
CREATE TABLE orders (
    id          INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    total       NUMERIC(10, 2) NOT NULL
);
```

### UUID Primary Keys

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE products (
    id    UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name  TEXT NOT NULL
);
```

### Composite Primary Keys

```sql
CREATE TABLE enrollments (
    student_id  INTEGER NOT NULL,
    course_id   INTEGER NOT NULL,
    enrolled_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (student_id, course_id)
);
```

Use composite primary keys only when the combination is truly the natural key.

> ⚠️ **Watch out:** Every FK pointing at a composite key must repeat all its columns — that is verbose and error-prone. Prefer a single-column surrogate key unless the composite really is the natural key.

---

## 7. Constraints

Constraints enforce data integrity at the database level. Never rely solely on
application code to enforce rules.

### NOT NULL

```sql
CREATE TABLE users (
    id    SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name  VARCHAR(100) NOT NULL
);
```

### UNIQUE

```sql
ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);
```

A UNIQUE constraint implicitly creates an index.

> 💡 **Note:** Because of that implicit index, a `UNIQUE` constraint both guarantees integrity and accelerates lookups by that column — no separate `CREATE INDEX` needed.

### DEFAULT

```sql
CREATE TABLE posts (
    id         SERIAL PRIMARY KEY,
    title      TEXT NOT NULL,
    status     VARCHAR(20) DEFAULT 'draft',
    created_at TIMESTAMPTZ DEFAULT now()
);
```

### CHECK

```sql
CREATE TABLE products (
    id    SERIAL PRIMARY KEY,
    name  TEXT NOT NULL,
    price NUMERIC(10, 2) NOT NULL CHECK (price > 0),
    stock INTEGER NOT NULL CHECK (stock >= 0)
);
```

### Composite Unique Constraints

```sql
CREATE TABLE enrollments (
    student_id INTEGER NOT NULL,
    course_id  INTEGER NOT NULL,
    enrolled_at TIMESTAMPTZ DEFAULT now(),
    UNIQUE (student_id, course_id)
);
```

---

## 8. Foreign Keys and Relationships

A foreign key references the primary key of another table and enforces
referential integrity.

### One-to-Many

```sql
CREATE TABLE customers (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(100) NOT NULL
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    total       NUMERIC(10, 2) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT now()
);
```

### One-to-One

```sql
CREATE TABLE profiles (
    id         SERIAL PRIMARY KEY,
    user_id    INTEGER NOT NULL UNIQUE REFERENCES users(id),
    bio        TEXT,
    avatar_url TEXT
);
```

### Many-to-Many

```sql
CREATE TABLE students (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE courses (
    id    SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL
);

CREATE TABLE enrollments (
    student_id  INTEGER NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    course_id   INTEGER NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    enrolled_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (student_id, course_id)
);
```

### ON DELETE and ON UPDATE Actions

| Action       | Behavior                                             |
|--------------|------------------------------------------------------|
| `CASCADE`    | Delete/update the referencing rows automatically     |
| `SET NULL`   | Set the foreign key column to NULL                   |
| `SET DEFAULT`| Set the foreign key column to its default value      |
| `RESTRICT`   | Reject the delete/update immediately                 |
| `NO ACTION`  | Same as RESTRICT but checked at end of transaction   |

> ⚠️ **Gotcha:** `ON DELETE CASCADE` deletes children silently. It is convenient but can wipe out data you never expected to lose — use it deliberately, never as a default reflex.

---

## 9. CRUD Operations in Depth

### SELECT Execution Order

```sql
SELECT column1, column2, expression AS alias
FROM table_name
WHERE condition
GROUP BY column1
HAVING group_condition
ORDER BY column1 [ASC|DESC]
LIMIT n OFFSET m;
```

Logical execution order:

1. FROM (and JOINs)
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT
6. ORDER BY
7. LIMIT/OFFSET

> 🧠 **Memory aid:** SQL reads like the sentence is written — it figures out WHERE's after WHERE's: think "**F**rom **W**here **G**rouping, **H**aving **S**elect **O**rdered, **L**imited". Write your queries in that conceptual order and you'll rarely misplace a clause.

### INSERT ... RETURNING

```sql
INSERT INTO users (email, name) VALUES ('eve@example.com', 'Eve')
    RETURNING id, email, created_at;
```

### UPDATE ... RETURNING

```sql
UPDATE users SET name = 'Eve Smith' WHERE email = 'eve@example.com'
    RETURNING id, name;
```

### INSERT ... ON CONFLICT (Upsert)

```sql
INSERT INTO users (email, name) VALUES ('alice@example.com', 'Alice')
    ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;
```

`EXCLUDED` refers to the row that was proposed for insertion.

### DELETE ... RETURNING

```sql
DELETE FROM orders WHERE created_at < '2025-01-01'
    RETURNING id, total;
```

### CTEs (Common Table Expressions)

```sql
WITH active_customers AS (
    SELECT id, name
    FROM customers
    WHERE is_active = true
)
SELECT ac.name, COUNT(o.id) AS order_count
FROM active_customers ac
JOIN orders o ON o.customer_id = ac.id
GROUP BY ac.name
ORDER BY order_count DESC;
```

### Subqueries

```sql
-- Subquery in WHERE
SELECT * FROM orders
WHERE customer_id IN (
    SELECT id FROM customers WHERE is_active = true
);

-- Correlated subquery
SELECT c.name,
    (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS order_count
FROM customers c;
```

---

## 10. Filtering and Sorting

### WHERE Operators

```sql
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE name != 'Alice';
SELECT * FROM users WHERE deleted_at IS NULL;
SELECT * FROM users WHERE deleted_at IS NOT NULL;
SELECT * FROM products WHERE price > 10.00;
SELECT * FROM products WHERE price <= 50.00;
SELECT * FROM users WHERE id IN (1, 3, 5);
SELECT * FROM orders WHERE created_at BETWEEN '2026-01-01' AND '2026-01-31';
SELECT * FROM users WHERE name LIKE 'A%';
SELECT * FROM users WHERE name ILIKE '%alice%';
```

> ⚠️ **Gotcha:** `NULL = anything` is never true (and so is `NULL <> x`) — even `NULL = NULL` fails. Always use `IS NULL` / `IS NOT NULL` to test for missing values.

### Logical Operators

```sql
SELECT * FROM users WHERE is_active = true AND name ILIKE '%alice%';
SELECT * FROM users WHERE role = 'admin' OR role = 'moderator';
SELECT * FROM users WHERE NOT is_active = true;
```

### ORDER BY

```sql
SELECT * FROM users ORDER BY name ASC;
SELECT * FROM users ORDER BY created_at DESC;
SELECT * FROM users ORDER BY is_active DESC, name ASC;
```

---

## 11. JOINs

JOINs combine rows from two or more tables based on a related column.

### Sample Tables

```sql
CREATE TABLE customers (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(100) NOT NULL
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    total       NUMERIC(10, 2) NOT NULL
);

CREATE TABLE order_items (
    id       SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product  TEXT NOT NULL,
    quantity INTEGER NOT NULL
);
```

### INNER JOIN

Returns only rows that have matching values in both tables.

```sql
SELECT c.name, o.id AS order_id, o.total
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id;
```

### LEFT JOIN

Returns all rows from the left table, and matching rows from the right. If
there is no match, the right-side columns are NULL.

```sql
SELECT c.name, o.id AS order_id, o.total
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;
```

### RIGHT JOIN

Returns all rows from the right table, and matching rows from the left. Rarely
used — rewrite as a LEFT JOIN by swapping table order.

```sql
SELECT c.name, o.id AS order_id, o.total
FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.id;
```

### FULL JOIN

Returns all rows from both tables. Unmatched rows have NULL on the side without
a match.

```sql
SELECT c.name, o.id AS order_id, o.total
FROM customers c
FULL JOIN orders o ON o.customer_id = c.id;
```

### JOIN Types Summary

```mermaid
flowchart LR
    subgraph INNER["INNER JOIN"]
        A[Matched Only]
    end
    subgraph LEFTJ["LEFT JOIN"]
        B[All Left] --> C[Matched Right]
        B --> D[NULL Right]
    end
    subgraph FULLJ["FULL JOIN"]
        E[All Left + NULL] --- F[All Right + NULL]
    end
```

| Join Type | Left rows | Right rows | No match             |
|-----------|-----------|------------|----------------------|
| INNER     | matched   | matched    | excluded             |
| LEFT      | all       | matched    | NULL on right        |
| RIGHT     | matched   | all        | NULL on left         |
| FULL      | all       | all        | NULL on either side  |

> 🧠 **Memory aid:** Ask "what survives if there is no match?" — INNER keeps matched rows only, LEFT keeps everything on the left with NULLs on the right, FULL keeps both sides with NULLs where either is missing.

### Multi-table JOIN

```sql
SELECT c.name, o.id AS order_id, oi.product, oi.quantity
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE c.name = 'Alice';
```

### Self JOIN

```sql
SELECT a.name, b.name, a.total
FROM orders a
JOIN orders b ON a.total = b.total AND a.id != b.id;
```

---

## 12. Aggregation

### Aggregate Functions

```sql
SELECT COUNT(*) AS total_orders FROM orders;
SELECT SUM(total) AS revenue FROM orders WHERE created_at >= '2026-01-01';
SELECT AVG(total) AS avg_order_value FROM orders;
SELECT MIN(total) AS smallest_order FROM orders;
SELECT MAX(total) AS largest_order FROM orders;
```

### GROUP BY

```sql
SELECT customer_id, COUNT(*) AS order_count, SUM(total) AS total_spent
FROM orders
GROUP BY customer_id;
```

Every non-aggregated column in SELECT must appear in GROUP BY.

> ⚠️ **Gotcha:** This rule trips up beginners daily: select `c.name` but group by `c.id` only, and PostgreSQL rejects the query. Group by the same columns you select (or wrap them in an aggregate).

### HAVING

Filters groups after aggregation (unlike WHERE which filters rows before
grouping):

```sql
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 5;
```

### GROUP BY with JOINs

```sql
SELECT c.name, COUNT(o.id) AS order_count, SUM(o.total) AS total_spent
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;
```

### Aggregate with CASE

```sql
SELECT
    customer_id,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN total >= 100 THEN 1 ELSE 0 END) AS large_orders,
    SUM(CASE WHEN total < 100 THEN 1 ELSE 0 END) AS small_orders
FROM orders
GROUP BY customer_id;
```

---

## 13. Transactions

A transaction is a logical unit of work that either completes entirely or not
at all.

### Why Transactions?

Without transactions, a failure halfway through a multi-step operation leaves
the database in an inconsistent state:

```sql
-- Without transactions: if the second INSERT fails, money is lost
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
```

With transactions:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

```mermaid
stateDiagram-v2
    [*] --> Active: BEGIN
    Active --> Committed: COMMIT
    Active --> RolledBack: ROLLBACK
    Committed --> [*]
    RolledBack --> [*]
    Active --> Savepoint: SAVEPOINT sp1
    Savepoint --> Active: ROLLBACK TO SAVEPOINT
```

### Savepoints

For partial rollback within a transaction:

```sql
BEGIN;
INSERT INTO users (email, name) VALUES ('a@test.com', 'A');
SAVEPOINT sp1;
INSERT INTO users (email, name) VALUES ('b@test.com', 'B');
ROLLBACK TO SAVEPOINT sp1;
-- 'a' is still inserted, 'b' was rolled back
COMMIT;
```

### Error Handling in Go

```go
tx, err := db.Begin(ctx)
if err != nil {
    return fmt.Errorf("begin tx: %w", err)
}
defer tx.Rollback(ctx) // no-op if already committed

_, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, fromID)
if err != nil {
    return fmt.Errorf("debit: %w", err)
}

_, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, toID)
if err != nil {
    return fmt.Errorf("credit: %w", err)
}

if err := tx.Commit(); err != nil {
    return fmt.Errorf("commit: %w", err)
}
```

The deferred `Rollback` is safe — calling it after `Commit` is a no-op.

> 💡 **Pro tip:** The `defer tx.Rollback(ctx)` pattern is idiomatic Go: you never have to trace every error path to remember to clean up. Commit first, and the deferred rollback simply does nothing.

---

## 14. Isolation Levels

Isolation levels define how transactions interact when they run concurrently.

### READ COMMITTED (PostgreSQL default)

Each statement sees only data committed before that statement began. A
transaction may see different rows across multiple SELECTs (non-repeatable
reads). Good for most workloads.

### REPEATABLE READ

All SELECTs within a transaction see a snapshot as of the start of the
transaction. Non-repeatable reads are prevented. PostgreSQL also prevents
phantom reads at this level.

### SERIALIZABLE

Transactions behave as if they were executed sequentially. The strongest
isolation level. PostgreSQL implements this with SSI (Serializable Snapshot
Isolation).

### Setting Isolation Level

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
```

In Go:

```go
tx, err := db.BeginTx(ctx, &sql.TxOptions{
    Isolation: sql.LevelSerializable,
})
```

Start with READ COMMITTED. Use SERIALIZABLE sparingly — it has the highest
abort rate.

> 🔑 **Key idea:** Isolation is a trade-off: the stronger the guarantee, the more aborts and contention. Pick the weakest level that your correctness requires — most apps never need more than READ COMMITTED.

---

## 15. Pagination

### OFFSET/LIMIT Pagination

```sql
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 0;   -- Page 1
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 10;  -- Page 2
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 20;  -- Page 3
```

**The problem**: OFFSET 1000000 LIMIT 10 reads 1,000,010 rows and discards
1,000,000. Performance degrades linearly.

### Cursor-Based (Keyset) Pagination

```sql
-- First page
SELECT * FROM products ORDER BY id LIMIT 10;
-- Next page: use the last id from previous page
SELECT * FROM products WHERE id > 10 ORDER BY id LIMIT 10;
-- Next page
SELECT * FROM products WHERE id > 20 ORDER BY id LIMIT 10;
```

Performance is constant regardless of page depth.

**When to use which**:

- OFFSET/LIMIT: Small datasets, page numbers in UI, admin dashboards.
- Cursor-based: Infinite scroll feeds, large datasets, API pagination.

> 💡 **Note:** For APIs, prefer cursor pagination — it stays fast at any page depth and avoids the "page 3 changed while reading" inconsistency that OFFSET suffers from.

### Cursor Pagination in Go

```go
type PageRequest struct {
    Cursor string `form:"cursor"`
    Limit  int    `form:"limit" binding:"max=100"`
}

type PageResponse[T any] struct {
    Items      []T    `json:"items"`
    NextCursor string `json:"next_cursor"`
    HasMore    bool   `json:"has_more"`
}
```

---

## 16. Indexes

An index is a data structure (typically a B-tree) that allows the database to
find rows matching a condition without scanning the entire table.

### Without an Index

```sql
SELECT * FROM orders WHERE customer_id = 42;
```

PostgreSQL performs a **sequential scan** — reads every row. For 1 million rows,
this reads 1 million rows.

### With an Index

```sql
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

PostgreSQL uses the B-tree index to jump directly to the matching rows. For
1 million rows where customer 42 has 10 orders, this reads ~3-4 index pages
plus 10 data pages.

### Creating Indexes

```sql
-- Basic index
CREATE INDEX idx_users_email ON users (email);

-- Unique index (also enforces uniqueness)
CREATE UNIQUE INDEX idx_users_email_unique ON users (email);

-- Composite index
CREATE INDEX idx_orders_customer_date ON orders (customer_id, created_at DESC);

-- Partial index
CREATE INDEX idx_active_users ON users (name) WHERE is_active = true;

-- Expression index
CREATE INDEX idx_users_lower_email ON users (LOWER(email));

-- GIN index for JSONB
CREATE INDEX idx_events_payload ON events USING GIN (payload);

-- GIN index for full-text search
CREATE INDEX idx_products_name_search ON products USING GIN (to_tsvector('english', name));
```

### When to Create Indexes

- A column is frequently used in WHERE clauses.
- A column is used in JOIN conditions (foreign keys).
- A column is used in ORDER BY.
- A column is used in GROUP BY.

### When NOT to Create Indexes

- Columns with very few distinct values (booleans, 3-option status columns).
- Small tables (under ~10,000 rows).
- Columns that are rarely queried.

### Index Maintenance

Indexes speed up reads but slow down writes. A table with 20 indexes will have
slower writes than one with 3.

Monitor unused indexes:

```sql
SELECT
    indexrelname AS index_name,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(i.indexrelid)) AS size
FROM pg_stat_user_indexes ui
JOIN pg_index i ON ui.indexrelid = i.indexrelid
WHERE idx_scan = 0
ORDER BY pg_relation_size(i.indexrelid) DESC;
```

### EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

If you see "Seq Scan" on a large table, you likely need an index.

---

## 17. SQL Injection

SQL injection is an attack where malicious SQL is inserted via user input.

### The Vulnerable Code

```go
// NEVER DO THIS
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", userInput)
rows, err := db.Query(query)
```

If `userInput` is `' OR '1'='1' --`, the query becomes:

```sql
SELECT * FROM users WHERE email = '' OR '1'='1' --'
```

A more destructive attacker might use `'; DROP TABLE users; --`.

### Parameterized Queries

```go
// SAFE: parameterized query
rows, err := db.Query("SELECT * FROM users WHERE email = $1", userInput)
```

PostgreSQL uses `$1`, `$2`, etc. as placeholders. The driver handles escaping.

> ⚠️ **Watch out:** String concatenation (`fmt.Sprintf("WHERE email = '%s'", input)`) is the #1 SQL injection vector. A parameterized query not only fixes it — it also removes quoting bugs.

### Prepared Statements

```go
stmt, err := db.Prepare("INSERT INTO users (email, name) VALUES ($1, $2)")
if err != nil {
    return err
}
defer stmt.Close()

_, err = stmt.Exec("alice@example.com", "Alice")
```

### Rules

1. Never concatenate user input into SQL strings.
2. Always use parameterized queries.
3. Never use `fmt.Sprintf` to build SQL queries.
4. Use an ORM or query builder if you prefer — they handle parameterization.

---

## 18. Connection Pooling

A PostgreSQL connection is a process with dedicated memory (~10MB). Creating a
new connection for every HTTP request is expensive.

### database/sql Pool

Go's `database/sql` package has a built-in connection pool:

```go
db, err := sql.Open("pgx", "postgres://localhost:5432/myapp")
if err != nil {
    log.Fatal(err)
}

// Maximum number of open connections (default: 0 = unlimited)
db.SetMaxOpenConns(25)

// Maximum number of idle connections (default: 2)
db.SetMaxIdleConns(10)

// Maximum lifetime of a connection (0 = no limit)
db.SetConnMaxLifetime(5 * time.Minute)

// Maximum time a connection can sit idle (0 = no limit)
db.SetConnMaxIdleTime(1 * time.Minute)
```

### Why These Settings Matter

- **MaxOpenConns**: Limits total connections. PostgreSQL default is 100. If every
  instance opens 100, 2 instances will exceed the limit.
- **MaxIdleConns**: Idle connections are warm — they skip authentication.
- **ConnMaxLifetime**: Prevents using connections PostgreSQL may have closed.
- **ConnMaxIdleTime**: Closes idle connections, freeing resources.

```mermaid
flowchart LR
    A[Go App] --> B[Connection Pool]
    B --> C1[Conn 1]
    B --> C2[Conn 2]
    B --> C3[Conn N]
    C1 --> D[PostgreSQL]
    C2 --> D
    C3 --> D
```

### External Connection Poolers

- **PgBouncer**: Lightweight, supports transaction, session, and statement
  pooling modes.
- **Pgpool-II**: More features (load balancing, replication).

For most single-instance Go apps, `database/sql`'s built-in pool is sufficient.

> 🔑 **Remember:** Watch PostgreSQL's `max_connections` (default 100) — if you run several app instances, spread the pool budget (`MaxOpenConns`) so `instances * budget` stays under the limit, or connections start failing.

---

## 19. Connecting Go to PostgreSQL

### Install Driver

```bash
go get github.com/jackc/pgx/v5/stdlib
```

### Basic Connection

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"

    _ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
    dsn := "postgres://admin:secret@localhost:5432/myapp?sslmode=disable"
    db, err := sql.Open("pgx", dsn)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := db.PingContext(ctx); err != nil {
        log.Fatal("database unreachable:", err)
    }

    fmt.Println("connected to database")
}
```

### Querying

```go
// Query a single row
var name string
err = db.QueryRowContext(ctx, "SELECT name FROM users WHERE id = $1", 1).Scan(&name)
if err != nil {
    if err == sql.ErrNoRows {
        // handle not found
    }
    return err
}

// Query multiple rows
rows, err := db.QueryContext(ctx, "SELECT id, name, email FROM users WHERE is_active = $1", true)
if err != nil {
    return err
}
defer rows.Close()

for rows.Next() {
    var id int
    var name, email string
    if err := rows.Scan(&id, &name, &email); err != nil {
        return err
    }
    fmt.Printf("id=%d name=%s email=%s\n", id, name, email)
}
if err := rows.Err(); err != nil {
    return err
}

// Execute (INSERT, UPDATE, DELETE)
result, err := db.ExecContext(ctx,
    "INSERT INTO users (email, name) VALUES ($1, $2)",
    "bob@example.com", "Bob",
)
if err != nil {
    return err
}
rowsAffected, _ := result.RowsAffected()
```

### Connection String Options

| Parameter          | Purpose                           | Example             |
|--------------------|-----------------------------------|----------------------|
| `sslmode`          | TLS mode                          | `disable`, `require` |
| `connect_timeout`  | Connection timeout in seconds     | `5`                  |
| `application_name` | Application name in pg_stat       | `myapp`              |
| `search_path`      | Schema search path                | `public, custom`     |

---

## 20. Modern Practices

### Always Use TIMESTAMPTZ

Use `TIMESTAMPTZ` instead of `TIMESTAMP`. PostgreSQL stores TIMESTAMPTZ in UTC
and converts to the session timezone on retrieval. This avoids timezone bugs
across servers in different regions.

### Prefer TEXT Over VARCHAR(n) Unless You Need a Limit

In PostgreSQL there is no performance difference. `TEXT` is simpler and avoids
arbitrary limits.

### Use Generated Columns for Computed Data

PostgreSQL 12+ supports generated columns that are automatically computed from
other columns:

```sql
CREATE TABLE products (
    price    NUMERIC(10,2),
    tax_rate NUMERIC(4,2),
    total    NUMERIC(10,2) GENERATED ALWAYS AS (price * (1 + tax_rate)) STORED
);
```

### Use pg_stat_statements for Query Analysis

Enable the `pg_stat_statements` extension to track query performance:

```sql
CREATE EXTENSION pg_stat_statements;
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;
```

### Environment Variables for Connection Strings

Never hardcode credentials. Use environment variables or a secrets manager.

### Use EXPLAIN ANALYZE Before Deploying

Always run `EXPLAIN ANALYZE` on slow queries before deploying. A query that
works fast with 100 rows may take seconds with 1 million rows.

---

## 21. Common Mistakes

### Using SELECT *

Specify the columns you need:

```sql
-- Bad
SELECT * FROM users WHERE id = $1;

-- Good
SELECT id, name, email FROM users WHERE id = $1;
```

### Forgetting to Check rows.Err()

```go
for rows.Next() { ... }
if err := rows.Err(); err != nil {
    return err
}
```

Rows may encounter an error mid-iteration. Without checking, you silently lose
data.

### Not Using Transactions for Multi-Step Operations

Any operation that must be atomic should use a transaction. Without one, partial
failures leave the database inconsistent.

### Ignoring NULL Handling

Scanning a NULL value into a non-pointer type causes an error:

```go
var bio sql.NullString
err := row.Scan(&id, &name, &bio)
if bio.Valid {
    fmt.Println(bio.String)
}
```

### Creating Too Many Indexes

Every index slows down writes. Monitor with `pg_stat_user_indexes` and drop
unused indexes.

### Not Using EXPLAIN

Always run `EXPLAIN ANALYZE` on slow queries before deploying.

### Using VARCHAR(255) Everywhere

Not every text column needs a length limit. Use `TEXT` for unlimited content.

### Not Setting Timezone Awareness

Always use `TIMESTAMPTZ` instead of `TIMESTAMP`.

---

## 22. Exercises

### Exercise 1: Schema Design

Create a schema for a blog application:

- `authors`: id, name, email, bio, created_at
- `posts`: id, author_id (FK), title, body, status (draft/published/archived),
  created_at, published_at
- `tags`: id, name (unique)
- `post_tags`: post_id (FK), tag_id (FK) -- junction table

Write the complete DDL including all constraints, primary keys, foreign keys,
and appropriate indexes.

### Exercise 2: CRUD Queries

Using the blog schema from Exercise 1:

1. Insert 3 authors and 5 posts across them.
2. Select all published posts with their author names, ordered by
   `published_at` DESC.
3. Update a post's status from 'draft' to 'published' and set `published_at`
   to the current time. Return the updated row.
4. Delete a post and verify the junction table entries are also removed (use
   ON DELETE CASCADE).

### Exercise 3: JOINs and Aggregation

1. Write a query that returns every author and their post count (including
   authors with zero posts).
2. Write a query that returns each tag and the number of posts using it,
   sorted by post count DESC. Only show tags used by more than 2 posts.
3. Write a query that returns the author with the highest total word count
   across all their posts.

### Exercise 4: Pagination

Implement cursor-based pagination for a posts listing API:

1. Write a SQL query that fetches 10 published posts after a given post ID,
   ordered by `published_at DESC, id DESC`.
2. Explain why `ORDER BY published_at DESC, id DESC` is necessary for
   cursor-based pagination (hint: what if two posts have the same timestamp?).

### Exercise 5: Transactions and Error Handling

Write a Go function that transfers a "credit balance" between two users:

```go
func TransferCredits(ctx context.Context, db *sql.DB, fromID, toID, amount int) error
```

Requirements:
1. Use a transaction.
2. Check that the source user has sufficient balance.
3. Debit the source and credit the destination.
4. Return an error if any step fails.
5. Use parameterized queries throughout.

### Exercise 6: Indexing and Performance

Given this table with 10 million rows:

```sql
CREATE TABLE analytics (
    id          SERIAL PRIMARY KEY,
    event_type  VARCHAR(50) NOT NULL,
    user_id     INTEGER NOT NULL,
    payload     JSONB,
    created_at  TIMESTAMPTZ DEFAULT now()
);
```

1. Write the most common queries you would run against this table.
2. Create appropriate indexes for those queries.
3. Write an EXPLAIN ANALYZE for each query and verify the indexes are used.

### Exercise 7: SQL Injection

Given this vulnerable Go code:

```go
func GetUser(db *sql.DB, email string) (*User, error) {
    query := "SELECT id, name, email FROM users WHERE email = '" + email + "'"
    row := db.QueryRow(query)
    var u User
    err := row.Scan(&u.ID, &u.Name, &u.Email)
    return &u, err
}
```

1. Rewrite this function to use parameterized queries.
2. Explain what an attacker could do with the original code.
3. Write a test case that demonstrates the injection attempt would fail with
   the fixed version.

---

## 23. Key Takeaways

- PostgreSQL is the default choice for Go backends: ACID, MVCC, JSONB, extensions.
- Tables, columns, constraints, and indexes form the schema — enforce integrity
  at the database level, not just in application code.
- JOINs (INNER, LEFT, FULL) combine data across tables; aggregation computes
  summaries with GROUP BY and HAVING.
- Transactions ensure atomicity — use `BEGIN`, `COMMIT`, `ROLLBACK` with
  `defer tx.Rollback()` in Go.
- Indexes accelerate reads but slow down writes — create them for WHERE, JOIN,
  and ORDER BY columns; monitor with `pg_stat_user_indexes`.
- Always use parameterized queries ($1, $2) — never `fmt.Sprintf` for SQL.
- `database/sql` has a built-in connection pool — configure MaxOpenConns,
  MaxIdleConns, ConnMaxLifetime.
- Use `TIMESTAMPTZ`, `TEXT` over `VARCHAR(255)`, and `EXPLAIN ANALYZE` for
  production queries.

---

**Next**: [database/sql in Go — connecting, querying, transactions, repository pattern, migrations, testing](02-database-sql-go.md)
