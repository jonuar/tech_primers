# PostgreSQL Cheatsheet

## Mental Model

PostgreSQL is a **full-featured relational database** that goes far beyond basic SQL — it has advanced indexing, JSONB for semi-structured data, CTEs, window functions, full-text search, and extensions (pgvector, PostGIS, etc.). The mental model: everything is a relation (table), queries are declarative transformations on those relations, and the query planner decides *how* to execute them. Understanding `EXPLAIN ANALYZE` is the key skill for performance work.

---

## Install & Minimal Setup

```bash
# Ubuntu / Debian
sudo apt install postgresql postgresql-client

# macOS
brew install postgresql@16
brew services start postgresql@16

# Docker (recommended for dev)
docker run -d \
  --name postgres-dev \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  postgres:16

# Connect
psql -U postgres -d mydb
psql "postgresql://user:password@localhost:5432/mydb"

# Python
pip install psycopg2-binary sqlalchemy asyncpg

# Node.js
npm install pg
```

---

## psql Essentials

```sql
-- Meta-commands (no semicolon needed)
\l              -- list databases
\c mydb         -- connect to database
\dt             -- list tables
\dt schema.*    -- list tables in schema
\d users        -- describe table
\di             -- list indexes
\df             -- list functions
\x              -- toggle expanded output (good for wide rows)
\timing         -- show query execution time
\e              -- open query in $EDITOR
\i file.sql     -- execute SQL file
\q              -- quit

-- Useful settings
SET search_path TO myschema, public;
```

---

## Core Concepts

### 1. Data Types

```sql
-- Numeric
INTEGER, INT          -- 4-byte integer
BIGINT                -- 8-byte integer
SERIAL, BIGSERIAL     -- auto-increment integer (legacy — prefer GENERATED)
NUMERIC(10, 2)        -- exact decimal, 10 digits, 2 after decimal
FLOAT8, DOUBLE PRECISION

-- Text
TEXT                  -- unlimited length (preferred)
VARCHAR(n)            -- limited length
CHAR(n)               -- fixed length, space-padded

-- Date / Time
DATE                  -- YYYY-MM-DD
TIME                  -- HH:MM:SS
TIMESTAMP             -- no timezone
TIMESTAMPTZ           -- with timezone (always prefer this)
INTERVAL              -- duration

-- Boolean
BOOLEAN               -- true / false / null

-- JSON
JSON                  -- stored as text, validates on insert
JSONB                 -- binary, indexed, queryable (always prefer this)

-- Arrays
INTEGER[]             -- array of integers
TEXT[]                -- array of text

-- UUID
UUID                  -- universally unique identifier

-- Enum
CREATE TYPE status AS ENUM ('active', 'inactive', 'pending');
```

### 2. Table DDL

```sql
CREATE TABLE users (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        TEXT NOT NULL,
    email       TEXT NOT NULL UNIQUE,
    role        TEXT NOT NULL DEFAULT 'user',
    metadata    JSONB DEFAULT '{}',
    tags        TEXT[],
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Alter table
ALTER TABLE users ADD COLUMN age INTEGER;
ALTER TABLE users DROP COLUMN age;
ALTER TABLE users ALTER COLUMN name SET NOT NULL;
ALTER TABLE users RENAME COLUMN name TO full_name;
ALTER TABLE users ADD CONSTRAINT chk_age CHECK (age > 0);

-- Enums
CREATE TYPE user_role AS ENUM ('admin', 'user', 'guest');
ALTER TABLE users ALTER COLUMN role TYPE user_role USING role::user_role;
```

### 3. Indexes

```sql
-- B-tree (default) — equality and range queries
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_created ON users (created_at DESC);

-- Unique index
CREATE UNIQUE INDEX idx_users_email_unique ON users (email);

-- Partial index — only index rows matching condition (smaller, faster)
CREATE INDEX idx_active_users ON users (email) WHERE active = true;

-- Composite index — order matters (leftmost prefix rule)
CREATE INDEX idx_users_role_created ON users (role, created_at DESC);

-- GIN index — for JSONB, arrays, full-text search
CREATE INDEX idx_users_metadata ON users USING gin (metadata);
CREATE INDEX idx_users_tags ON users USING gin (tags);

-- GiST index — for full-text search tsvectors
CREATE INDEX idx_docs_search ON documents USING gist (to_tsvector('english', content));

-- BRIN index — for large, naturally ordered tables (time-series, logs)
CREATE INDEX idx_events_created ON events USING brin (created_at);

-- Inspect index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### 4. Queries

```sql
-- Basic SELECT
SELECT id, name, email FROM users WHERE active = true ORDER BY created_at DESC LIMIT 10;

-- DISTINCT
SELECT DISTINCT role FROM users;

-- LIKE / ILIKE (case-insensitive)
SELECT * FROM users WHERE name ILIKE '%joshua%';

-- Array operators
SELECT * FROM users WHERE 'admin' = ANY(tags);
SELECT * FROM users WHERE tags @> ARRAY['admin', 'staff'];  -- contains

-- JSONB operators
SELECT metadata->>'key' FROM users;                   -- get as text
SELECT metadata->'nested'->>'key' FROM users;        -- nested
SELECT * FROM users WHERE metadata @> '{"plan": "pro"}';  -- contains
SELECT * FROM users WHERE metadata ? 'api_key';       -- key exists
SELECT * FROM users WHERE metadata->>'score' > '90'; -- compare as text

-- NULL handling
SELECT * FROM users WHERE deleted_at IS NULL;
SELECT COALESCE(nickname, name, 'Unknown') FROM users;
SELECT NULLIF(value, '') FROM table;   -- returns NULL if value = ''
```

### 5. Joins

```sql
-- INNER JOIN
SELECT u.name, o.total
FROM users u
JOIN orders o ON o.user_id = u.id;

-- LEFT JOIN (keep all users even without orders)
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;

-- Multiple joins
SELECT u.name, p.name AS product, oi.quantity
FROM users u
JOIN orders o ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id;

-- Self join
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;
```

### 6. Aggregations & Window Functions

```sql
-- Aggregations
SELECT role, COUNT(*), AVG(age), MAX(created_at)
FROM users
GROUP BY role
HAVING COUNT(*) > 5
ORDER BY COUNT(*) DESC;

-- Window functions — aggregate without collapsing rows
SELECT
    name,
    salary,
    department,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    ROW_NUMBER() OVER (ORDER BY created_at) AS row_num,
    LAG(salary, 1) OVER (ORDER BY created_at) AS prev_salary,
    SUM(amount) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM employees;
```

### 7. CTEs (Common Table Expressions)

```sql
-- Basic CTE — readability
WITH active_users AS (
    SELECT * FROM users WHERE active = true AND deleted_at IS NULL
),
user_stats AS (
    SELECT user_id, COUNT(*) AS order_count, SUM(total) AS lifetime_value
    FROM orders
    GROUP BY user_id
)
SELECT u.name, s.order_count, s.lifetime_value
FROM active_users u
LEFT JOIN user_stats s ON s.user_id = u.id
ORDER BY s.lifetime_value DESC NULLS LAST;

-- Recursive CTE — for hierarchies and graphs
WITH RECURSIVE org_tree AS (
    -- Base case
    SELECT id, name, manager_id, 0 AS depth
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case
    SELECT e.id, e.name, e.manager_id, ot.depth + 1
    FROM employees e
    JOIN org_tree ot ON ot.id = e.manager_id
)
SELECT * FROM org_tree ORDER BY depth, name;
```

### 8. Full-Text Search

```sql
-- to_tsvector converts text to a lexeme list
-- to_tsquery creates a query from search terms
SELECT title, content
FROM documents
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'machine & learning')
ORDER BY ts_rank(to_tsvector('english', content), to_tsquery('english', 'machine & learning')) DESC;

-- Stored tsvector column (much faster with GIN index)
ALTER TABLE documents ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(content,''))) STORED;

CREATE INDEX idx_docs_fts ON documents USING gin (search_vector);

SELECT title FROM documents
WHERE search_vector @@ websearch_to_tsquery('english', 'machine learning RAG')
ORDER BY ts_rank(search_vector, websearch_to_tsquery('english', 'machine learning RAG')) DESC;
```

### 9. Transactions & Locking

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;   -- or ROLLBACK;

-- Savepoints
BEGIN;
UPDATE ...;
SAVEPOINT my_savepoint;
UPDATE ...;
ROLLBACK TO my_savepoint;  -- undo only back to savepoint
COMMIT;

-- Explicit locking
SELECT * FROM orders WHERE id = 1 FOR UPDATE;          -- exclusive row lock
SELECT * FROM orders WHERE id = 1 FOR SHARE;           -- shared row lock
SELECT * FROM orders WHERE id = 1 FOR UPDATE SKIP LOCKED; -- skip locked rows (job queues)
```

### 10. Performance — EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;

-- Key things to look for:
-- Seq Scan → missing index
-- Nested Loop on large tables → missing index on join key
-- High "actual rows" vs "estimated rows" → stale statistics → run ANALYZE
-- High cost nodes → optimization targets

-- Update statistics
ANALYZE users;
ANALYZE;    -- all tables

-- Identify slow queries
SELECT query, mean_exec_time, calls, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

---

## Most-Used Patterns

### Upsert (INSERT ON CONFLICT)

```sql
INSERT INTO users (email, name, updated_at)
VALUES ('j@example.com', 'Joshua', NOW())
ON CONFLICT (email) DO UPDATE
    SET name = EXCLUDED.name,
        updated_at = NOW();

-- Ignore if exists
INSERT INTO tags (name) VALUES ('ai') ON CONFLICT DO NOTHING;
```

### Pagination

```sql
-- Offset-based (simple, slow on large offsets)
SELECT * FROM users ORDER BY created_at DESC LIMIT 20 OFFSET 40;

-- Cursor-based (fast at any depth — preferred for large tables)
SELECT * FROM users
WHERE created_at < '2024-01-01 00:00:00'   -- cursor from last row
ORDER BY created_at DESC
LIMIT 20;
```

### Update with Returning

```sql
UPDATE users SET active = false WHERE id = 1
RETURNING id, name, updated_at;
```

### Bulk Insert

```sql
INSERT INTO products (name, price, category)
VALUES
    ('Item A', 9.99, 'tools'),
    ('Item B', 19.99, 'tools'),
    ('Item C', 4.99, 'supplies');
```

---

## Gotchas

- **Always use `TIMESTAMPTZ`** — `TIMESTAMP` stores no timezone info; you'll get silent bugs when servers or clients are in different timezones.
- **`SERIAL` is legacy** — use `GENERATED ALWAYS AS IDENTITY` instead; `SERIAL` creates a sequence with ownership issues.
- **`COUNT(*)` vs `COUNT(column)`** — `COUNT(*)` counts all rows; `COUNT(column)` ignores NULLs. They're often different.
- **`NOT IN` with NULLs is a trap** — `WHERE id NOT IN (1, 2, NULL)` returns zero rows because `NULL` comparisons return `UNKNOWN`. Use `NOT EXISTS` instead.
- **Index on high-cardinality, not low-cardinality** — indexing a boolean column is usually useless; indexing email or UUID is very useful.
- **`LIKE '%term%'` can't use B-tree indexes** — leading wildcard forces a sequential scan. Use full-text search (`tsvector`) or `pg_trgm` for substring search.
- **`VACUUM` and `AUTOVACUUM`** — PostgreSQL uses MVCC; deleted rows aren't removed immediately. Autovacuum runs in the background, but heavy-write tables may need manual `VACUUM ANALYZE`.

---

## Quick Links

- [PostgreSQL Docs](https://www.postgresql.org/docs/current/)
- [pgAdmin](https://www.pgadmin.org) — GUI client
- [explain.dalibo.com](https://explain.dalibo.com) — visual EXPLAIN ANALYZE
- [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html) — slow query tracking
- [Supabase](https://supabase.com/docs) — managed Postgres with good extension support
- [Flyway](https://flywaydb.org) / [Liquibase](https://www.liquibase.org) — schema migrations
