# MySQL Cheatsheet

## Mental Model

MySQL is a **relational database** optimized for read-heavy OLTP workloads. The two most important things to understand: (1) the **storage engine** matters — InnoDB (default) gives you transactions, foreign keys, and row-level locking; MyISAM does not. (2) **indexes** are the biggest lever on performance — `EXPLAIN` before optimizing anything. MySQL is schema-first: define structure, then insert data.

---

## Install & Minimal Setup

```bash
# macOS
brew install mysql
brew services start mysql
mysql_secure_installation

# Ubuntu
sudo apt install mysql-server
sudo systemctl start mysql
sudo mysql_secure_installation

# Docker (easiest for dev)
docker run -d \
  --name mysql-dev \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=mydb \
  -e MYSQL_USER=myuser \
  -e MYSQL_PASSWORD=mypass \
  -p 3306:3306 \
  mysql:8.0

# Connect
mysql -u root -p
mysql -h 127.0.0.1 -u myuser -pmypass mydb

# Python
pip install mysql-connector-python
# or: pip install PyMySQL sqlalchemy
```

---

## Core Concepts

### 1. Database & Table Management

```sql
-- Databases
SHOW DATABASES;
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE mydb;
DROP DATABASE mydb;

-- Tables
SHOW TABLES;
DESCRIBE users;
SHOW CREATE TABLE users;

CREATE TABLE users (
    id          BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT,
    name        VARCHAR(100)       NOT NULL,
    email       VARCHAR(255)       NOT NULL,
    role        ENUM('admin','user','guest') NOT NULL DEFAULT 'user',
    age         TINYINT UNSIGNED,
    bio         TEXT,
    metadata    JSON,
    created_at  DATETIME(3)        NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_at  DATETIME(3)        NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted_at  DATETIME(3),
    PRIMARY KEY (id),
    UNIQUE KEY uq_users_email (email),
    INDEX idx_users_role (role),
    INDEX idx_users_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Alter table
ALTER TABLE users ADD COLUMN phone VARCHAR(20) AFTER email;
ALTER TABLE users MODIFY COLUMN name VARCHAR(200) NOT NULL;
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users ADD INDEX idx_name (name);
ALTER TABLE users DROP INDEX idx_name;
```

### 2. Data Types

```sql
-- Integer
TINYINT       -- 1 byte: -128 to 127 (UNSIGNED: 0-255)
SMALLINT      -- 2 bytes
MEDIUMINT     -- 3 bytes
INT / INTEGER -- 4 bytes: -2.1B to 2.1B
BIGINT        -- 8 bytes

-- Decimal / Float
DECIMAL(10,2) -- exact: total 10 digits, 2 after decimal (use for money)
FLOAT         -- approximate, 4 bytes
DOUBLE        -- approximate, 8 bytes

-- String
CHAR(n)       -- fixed-length, padded with spaces
VARCHAR(n)    -- variable-length up to n chars (n ≤ 65535)
TINYTEXT      -- up to 255 bytes
TEXT          -- up to 65KB
MEDIUMTEXT    -- up to 16MB
LONGTEXT      -- up to 4GB

-- Binary
BLOB, MEDIUMBLOB, LONGBLOB

-- Date / Time
DATE          -- 'YYYY-MM-DD'
TIME          -- 'HH:MM:SS'
DATETIME(3)   -- 'YYYY-MM-DD HH:MM:SS.mmm' — use for application timestamps
TIMESTAMP     -- stored as UTC, auto-converts to server timezone — avoid
YEAR          -- 4-digit year

-- Other
BOOLEAN       -- alias for TINYINT(1)
JSON          -- validated JSON, queryable
ENUM('a','b') -- one value from a list
SET('a','b')  -- zero or more values from a list
```

### 3. CRUD

```sql
-- INSERT
INSERT INTO users (name, email, role) VALUES ('Joshua', 'j@example.com', 'admin');

-- Insert multiple rows (much faster than individual inserts)
INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob',   'bob@example.com'),
    ('Carol',  'carol@example.com');

-- Upsert — insert or update on duplicate key
INSERT INTO users (id, name, email) VALUES (1, 'Joshua Updated', 'j@example.com')
ON DUPLICATE KEY UPDATE
    name = VALUES(name),
    updated_at = CURRENT_TIMESTAMP(3);

-- INSERT IGNORE — skip on duplicate key (no error, no update)
INSERT IGNORE INTO users (name, email) VALUES ('Joshua', 'j@example.com');

-- SELECT
SELECT * FROM users;
SELECT id, name, email FROM users WHERE role = 'admin' ORDER BY name LIMIT 10 OFFSET 20;
SELECT COUNT(*), role FROM users GROUP BY role HAVING COUNT(*) > 5;

-- UPDATE
UPDATE users SET name = 'Joshua N', updated_at = NOW(3) WHERE id = 1;
UPDATE users SET role = 'user' WHERE created_at < '2024-01-01' AND role = 'guest';

-- DELETE (soft delete pattern)
UPDATE users SET deleted_at = NOW(3) WHERE id = 1;

-- Hard delete
DELETE FROM users WHERE id = 1;
DELETE FROM users WHERE deleted_at < DATE_SUB(NOW(), INTERVAL 30 DAY);
```

### 4. JOINs

```sql
-- INNER JOIN — only matching rows
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN — all users, even without orders
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- Multiple joins
SELECT u.name, o.id AS order_id, p.name AS product
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE u.id = 1;
```

### 5. Indexes & EXPLAIN

```sql
-- Types of indexes
CREATE INDEX idx_name ON users(name);                  -- single column
CREATE INDEX idx_name_email ON users(name, email);     -- composite
CREATE UNIQUE INDEX uq_email ON users(email);
CREATE FULLTEXT INDEX ft_bio ON users(bio);            -- for MATCH AGAINST

-- EXPLAIN — always check before optimizing
EXPLAIN SELECT * FROM users WHERE email = 'j@example.com';
-- Key columns: type (ALL is bad), key (index used), rows (estimated scan)

EXPLAIN FORMAT=JSON SELECT ...\G  -- detailed JSON output

-- Types from best to worst:
-- system → const → eq_ref → ref → range → index → ALL (full scan)

-- Force index (when optimizer makes wrong choice)
SELECT * FROM users FORCE INDEX (idx_name) WHERE name LIKE 'Josh%';
```

### 6. Transactions

```sql
START TRANSACTION;          -- or: BEGIN;

UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;

-- Check if transfer is valid before committing
SELECT balance FROM accounts WHERE id = 1;

COMMIT;     -- make changes permanent
ROLLBACK;   -- undo all changes since START TRANSACTION

-- Savepoints (partial rollback)
SAVEPOINT sp1;
DELETE FROM order_items WHERE order_id = 5;
ROLLBACK TO sp1;  -- undo only the delete, keep the rest

-- Isolation levels
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- default in most apps
-- READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ (MySQL default) | SERIALIZABLE
```

### 7. Stored Procedures & Functions

```sql
DELIMITER //

CREATE PROCEDURE transfer_funds(
    IN  p_from_id   BIGINT,
    IN  p_to_id     BIGINT,
    IN  p_amount    DECIMAL(10,2),
    OUT p_status    VARCHAR(50)
)
BEGIN
    DECLARE v_balance DECIMAL(10,2);

    -- Error handler
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_status = 'ERROR';
    END;

    START TRANSACTION;

    SELECT balance INTO v_balance FROM accounts WHERE id = p_from_id FOR UPDATE;

    IF v_balance < p_amount THEN
        ROLLBACK;
        SET p_status = 'INSUFFICIENT_FUNDS';
    ELSE
        UPDATE accounts SET balance = balance - p_amount WHERE id = p_from_id;
        UPDATE accounts SET balance = balance + p_amount WHERE id = p_to_id;
        COMMIT;
        SET p_status = 'SUCCESS';
    END IF;
END //

DELIMITER ;

-- Call
CALL transfer_funds(1, 2, 500.00, @status);
SELECT @status;

-- Stored function
DELIMITER //
CREATE FUNCTION calculate_tax(p_amount DECIMAL(10,2)) RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    RETURN p_amount * 0.16;
END //
DELIMITER ;

SELECT calculate_tax(1000);  -- 160.00
```

### 8. JSON Column (MySQL 8+)

```sql
-- Insert JSON
INSERT INTO users (name, metadata) VALUES ('Joshua', '{"city": "CDMX", "skills": ["Python", "Go"]}');

-- Query JSON
SELECT name, metadata->>'$.city' AS city FROM users;
SELECT name FROM users WHERE metadata->>'$.city' = 'CDMX';

-- JSON functions
SELECT JSON_EXTRACT(metadata, '$.skills[0]') FROM users;
SELECT JSON_ARRAY_LENGTH(metadata->'$.skills') FROM users;
UPDATE users SET metadata = JSON_SET(metadata, '$.level', 'senior') WHERE id = 1;
UPDATE users SET metadata = JSON_ARRAY_APPEND(metadata, '$.skills', 'Rust') WHERE id = 1;
```

---

## Most-Used Patterns

### Pagination

```sql
-- Offset pagination (simple but slow on large offsets)
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 200;

-- Keyset pagination (fast, use for large tables)
SELECT * FROM users WHERE id > 200 ORDER BY id LIMIT 10;
```

### Python with mysql-connector

```python
import mysql.connector

conn = mysql.connector.connect(
    host="localhost", user="myuser", password="mypass", database="mydb"
)
cursor = conn.cursor(dictionary=True)   # returns dicts instead of tuples

# Parameterized query — always use %s placeholders, never f-strings
cursor.execute("SELECT * FROM users WHERE email = %s", ("j@example.com",))
user = cursor.fetchone()

# Insert
cursor.execute(
    "INSERT INTO users (name, email) VALUES (%s, %s)",
    ("Joshua", "j@example.com")
)
conn.commit()
print("Inserted ID:", cursor.lastrowid)

cursor.close()
conn.close()
```

### SQLAlchemy with MySQL

```python
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+pymysql://user:pass@localhost/mydb",
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True    # reconnects on stale connections
)
```

---

## Gotchas

- **`utf8` vs `utf8mb4`** — MySQL's `utf8` only stores 3-byte characters (no emoji). Always use `utf8mb4`.
- **`DATETIME` vs `TIMESTAMP`** — `TIMESTAMP` converts to UTC and back using the server timezone — dangerous if you ever change the timezone. Use `DATETIME(3)` and store everything in UTC at the application level.
- **`NOT IN` with NULL** — `WHERE id NOT IN (1, 2, NULL)` returns zero rows because `NULL` comparisons are always unknown. Use `NOT EXISTS` or filter nulls from the subquery.
- **No partial indexes** — MySQL doesn't support partial/filtered indexes. Index the entire column or use generated columns.
- **`GROUP BY` strictness** — MySQL 5.7+ enforces `ONLY_FULL_GROUP_BY` by default. Every selected column must be in `GROUP BY` or an aggregate function.
- **`ENUM` is hard to alter** — adding a value to an `ENUM` requires a full table rebuild in older MySQL versions. Consider a lookup table or `VARCHAR` instead.
- **`AUTO_INCREMENT` gaps** — rolled-back inserts still consume the auto-increment counter, creating gaps. Don't rely on sequential IDs.

---

## Quick Links

- [MySQL 8.0 Docs](https://dev.mysql.com/doc/refman/8.0/en/)
- [MySQL EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [Percona Blog](https://www.percona.com/blog/) — deep MySQL performance articles
- [MySQL Workbench](https://www.mysql.com/products/workbench/) — official GUI
- [DBeaver](https://dbeaver.io) — cross-DB GUI (recommended)
