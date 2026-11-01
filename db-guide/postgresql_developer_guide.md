# PostgreSQL Developer Guide (Linux)

> **Purpose:** This guide helps developers working on the project get productive with PostgreSQL quickly. Focus areas: connecting to the database, writing queries, filtering data, and avoiding common mistakes.

---

## Table of Contents

1. [Installation & Setup](#1-installation--setup)
2. [Connecting to the Database](#2-connecting-to-the-database)
3. [Database & Schema Basics](#3-database--schema-basics)
4. [Querying Data](#4-querying-data)
5. [Filtering Data](#5-filtering-data)
6. [Inserting, Updating & Deleting](#6-inserting-updating--deleting)
7. [Transactions](#7-transactions)
8. [Indexes](#8-indexes)
9. [Useful `psql` Commands](#9-useful-psql-commands)
10. [Common Pitfalls](#10-common-pitfalls)

---

## 1. Installation & Setup

```bash
# Install PostgreSQL (Ubuntu/Debian)
sudo apt update
sudo apt install postgresql postgresql-contrib

# Check service status
sudo systemctl status postgresql

# Start / stop / restart
sudo systemctl start postgresql
sudo systemctl stop postgresql
sudo systemctl restart postgresql
```

> **Note:** The default PostgreSQL service runs on port **5432**.

---

## 2. Connecting to the Database

### Using `psql` (CLI client)

```bash
# Connect as the system postgres superuser
sudo -u postgres psql

# Connect to a specific database
sudo -u postgres psql -d mydb

# Connect with a specific user
psql -h localhost -p 5432 -U myuser -d mydb

# Connect using a connection string (URI format)
psql "postgresql://myuser:mypassword@localhost:5432/mydb"
```

> ⚠️ **Important:** By default, PostgreSQL uses **peer authentication** for local connections, meaning the system user must match the database user. For application users (non-postgres), use `md5` or `scram-sha-256` authentication and connect via `localhost` (TCP) rather than the Unix socket.

### Authentication configuration (`pg_hba.conf`)

The file `/etc/postgresql/<version>/main/pg_hba.conf` controls who can connect and how.

```
# TYPE  DATABASE  USER    ADDRESS      METHOD
local   all       postgres             peer
host    all       all     127.0.0.1/32 scram-sha-256
host    all       all     ::1/128      scram-sha-256
```

After editing, reload without restarting:

```bash
sudo systemctl reload postgresql
# OR inside psql:
SELECT pg_reload_conf();
```

### Creating a project user

```sql
-- Run as postgres superuser
CREATE USER myuser WITH PASSWORD 'securepassword';
CREATE DATABASE mydb OWNER myuser;
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;

-- Grant schema-level access (PostgreSQL 15+)
\c mydb
GRANT ALL ON SCHEMA public TO myuser;
```

---

## 3. Database & Schema Basics

```sql
-- List all databases
\l

-- Switch to a database (inside psql)
\c mydb

-- List all schemas
\dn

-- List all tables in the current schema
\dt

-- Describe a table structure
\d tablename

-- List all tables with sizes
SELECT table_name,
       pg_size_pretty(pg_total_relation_size(quote_ident(table_name))) AS size
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY pg_total_relation_size(quote_ident(table_name)) DESC;
```

> ⚠️ **Important:** PostgreSQL is **case-sensitive** for identifiers when quoted. `UserID` and `userid` are different if one was created with quotes. Use **lowercase_with_underscores** for table and column names to avoid confusion.

---

## 4. Querying Data

### Basic SELECT

```sql
-- Select all columns
SELECT * FROM employees;

-- Select specific columns
SELECT id, name, department FROM employees;

-- Alias columns
SELECT id, name AS full_name, salary * 12 AS annual_salary
FROM employees;

-- Limit results
SELECT * FROM employees LIMIT 10;

-- Skip rows (pagination)
SELECT * FROM employees ORDER BY id LIMIT 10 OFFSET 20;
```

### Sorting

```sql
SELECT * FROM employees ORDER BY salary DESC, name ASC;
```

### Aggregation

```sql
SELECT department,
       COUNT(*)           AS headcount,
       AVG(salary)        AS avg_salary,
       MAX(salary)        AS top_salary
FROM employees
GROUP BY department
ORDER BY avg_salary DESC;
```

> ⚠️ **Important:** Every column in `SELECT` that is **not** an aggregate function must appear in `GROUP BY`. Forgetting this is the most common aggregation error.

### Joins

```sql
-- INNER JOIN: only matching rows
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;

-- LEFT JOIN: all employees, even those with no department
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id;
```

> ⚠️ **Important:** Prefer explicit `INNER JOIN` / `LEFT JOIN` over implicit comma joins (`FROM a, b WHERE a.id = b.id`). Implicit joins are harder to read and error-prone.

### Subqueries & CTEs

```sql
-- Subquery in WHERE
SELECT name FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- CTE (Common Table Expression) — preferred for readability
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY dept_id
)
SELECT e.name, e.salary, da.avg_sal
FROM employees e
JOIN dept_avg da ON e.dept_id = da.dept_id
WHERE e.salary > da.avg_sal;
```

---

## 5. Filtering Data

### WHERE clause operators

```sql
-- Equality and comparison
SELECT * FROM employees WHERE department = 'Engineering';
SELECT * FROM employees WHERE salary >= 50000;
SELECT * FROM employees WHERE salary BETWEEN 40000 AND 80000;

-- NULL checks — ALWAYS use IS NULL / IS NOT NULL
SELECT * FROM employees WHERE manager_id IS NULL;
SELECT * FROM employees WHERE manager_id IS NOT NULL;
```

> 🔴 **Critical:** **Never** use `= NULL` or `!= NULL`. These always return false in SQL.  
> The correct syntax is `IS NULL` and `IS NOT NULL`.

### Pattern matching

```sql
-- LIKE: case-sensitive pattern matching
SELECT * FROM employees WHERE name LIKE 'John%';    -- starts with John
SELECT * FROM employees WHERE name LIKE '%son';     -- ends with son
SELECT * FROM employees WHERE email LIKE '%@company.com';

-- ILIKE: case-insensitive version (PostgreSQL-specific)
SELECT * FROM employees WHERE name ILIKE 'john%';

-- Regex matching
SELECT * FROM employees WHERE name ~ '^J.*n$';      -- case-sensitive regex
SELECT * FROM employees WHERE name ~* '^j.*n$';     -- case-insensitive regex
```

> ⚠️ **Performance Warning:** `LIKE '%value%'` (leading wildcard) **cannot use a B-tree index** and causes a full table scan. For full-text or prefix searches on large tables, consider `pg_trgm` indexes or full-text search.

### IN and NOT IN

```sql
SELECT * FROM employees WHERE department IN ('Engineering', 'QA', 'DevOps');
SELECT * FROM employees WHERE id NOT IN (1, 2, 3);

-- IN with subquery
SELECT * FROM employees
WHERE dept_id IN (SELECT id FROM departments WHERE location = 'Hanoi');
```

> ⚠️ **Important:** `NOT IN` with a subquery returns **no rows** if the subquery contains any `NULL`. Use `NOT EXISTS` instead for safer semantics:
> ```sql
> -- Safe alternative to NOT IN with subquery
> SELECT * FROM employees e
> WHERE NOT EXISTS (
>     SELECT 1 FROM blacklist b WHERE b.employee_id = e.id
> );
> ```

### HAVING (filter after aggregation)

```sql
-- Filter groups after GROUP BY
SELECT department, COUNT(*) AS headcount
FROM employees
GROUP BY department
HAVING COUNT(*) > 5;
```

> **Rule:** Use `WHERE` to filter **rows** before aggregation; use `HAVING` to filter **groups** after aggregation.

---

## 6. Inserting, Updating & Deleting

### INSERT

```sql
-- Insert a single row
INSERT INTO employees (name, department, salary)
VALUES ('Alice Nguyen', 'Engineering', 75000);

-- Insert multiple rows
INSERT INTO employees (name, department, salary) VALUES
    ('Bob Tran', 'QA', 60000),
    ('Carol Le', 'DevOps', 68000);

-- Insert and return the generated ID
INSERT INTO employees (name, department, salary)
VALUES ('Dave Pham', 'Engineering', 72000)
RETURNING id, name;

-- Insert or update (upsert)
INSERT INTO employees (id, name, salary)
VALUES (42, 'Alice Nguyen', 80000)
ON CONFLICT (id) DO UPDATE SET salary = EXCLUDED.salary;
```

### UPDATE

```sql
-- Update specific rows
UPDATE employees SET salary = salary * 1.1 WHERE department = 'Engineering';

-- Update with RETURNING
UPDATE employees SET salary = 90000
WHERE id = 42
RETURNING id, name, salary;
```

> 🔴 **Critical:** **Always include a `WHERE` clause in `UPDATE` and `DELETE`**. Omitting it affects every row in the table. Run a `SELECT` with the same `WHERE` first to verify the target rows.

### DELETE

```sql
-- Delete specific rows
DELETE FROM employees WHERE id = 42;

-- Delete with RETURNING
DELETE FROM employees WHERE department = 'Temp' RETURNING *;

-- Truncate (fast delete of all rows — cannot be rolled back easily)
TRUNCATE TABLE logs;
```

---

## 7. Transactions

Wrap related operations in a transaction so they either all succeed or all fail.

```sql
BEGIN;

UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;

-- If everything is correct:
COMMIT;

-- If something went wrong:
ROLLBACK;
```

> ⚠️ **Important:** In PostgreSQL, **DDL statements (`CREATE`, `ALTER`, `DROP`) are transactional**. You can roll back a schema change — this is unlike MySQL/Oracle.

### Savepoints

```sql
BEGIN;
INSERT INTO orders (...) VALUES (...);

SAVEPOINT after_order;

INSERT INTO order_items (...) VALUES (...);
-- Oops, something wrong with items
ROLLBACK TO SAVEPOINT after_order;

-- Order is still intact; commit it
COMMIT;
```

### Isolation levels

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| READ COMMITTED *(default)* | No | Yes | Yes |
| REPEATABLE READ | No | No | No* |
| SERIALIZABLE | No | No | No |

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ... your queries
COMMIT;
```

---

## 8. Indexes

### Creating indexes

```sql
-- Basic index on a single column
CREATE INDEX idx_employees_department ON employees (department);

-- Multi-column index
CREATE INDEX idx_employees_dept_salary ON employees (department, salary);

-- Unique index
CREATE UNIQUE INDEX idx_employees_email ON employees (email);

-- Partial index (only index active employees — saves space)
CREATE INDEX idx_active_employees ON employees (department)
WHERE status = 'active';

-- Drop an index
DROP INDEX idx_employees_department;
```

> ⚠️ **Important:** Indexes **speed up reads** but **slow down writes** (`INSERT`/`UPDATE`/`DELETE`). Do not add indexes blindly. Add them for columns used frequently in `WHERE`, `JOIN ON`, and `ORDER BY` clauses.

### Check if a query uses an index

```sql
EXPLAIN ANALYZE
SELECT * FROM employees WHERE department = 'Engineering';
```

Key things to look for in `EXPLAIN ANALYZE` output:
- **Index Scan** / **Index Only Scan** → index is being used ✅
- **Seq Scan** on a large table → consider adding an index ⚠️
- **actual time** → real execution time in milliseconds

---

## 9. Useful `psql` Commands

| Command | Description |
|---|---|
| `\l` | List all databases |
| `\c dbname` | Connect to database |
| `\dt` | List tables in current schema |
| `\dt schema.*` | List tables in a specific schema |
| `\d tablename` | Describe table (columns, types, indexes) |
| `\di` | List indexes |
| `\dn` | List schemas |
| `\du` | List roles/users |
| `\timing` | Toggle query execution time display |
| `\x` | Toggle expanded output (useful for wide rows) |
| `\e` | Open query in `$EDITOR` |
| `\i file.sql` | Execute SQL from a file |
| `\o file.txt` | Send query output to a file |
| `\q` | Quit |
| `\?` | Help for psql commands |
| `\h SELECT` | SQL syntax help for a command |

### Run a query from the shell (non-interactive)

```bash
# Execute a single query
psql -U myuser -d mydb -c "SELECT COUNT(*) FROM employees;"

# Execute a SQL file
psql -U myuser -d mydb -f migration.sql

# Export query result to CSV
psql -U myuser -d mydb -c "\COPY (SELECT * FROM employees) TO '/tmp/employees.csv' CSV HEADER;"
```

---

## 10. Common Pitfalls

### ❌ Using `= NULL` instead of `IS NULL`
```sql
-- WRONG — always returns no rows
SELECT * FROM employees WHERE manager_id = NULL;

-- CORRECT
SELECT * FROM employees WHERE manager_id IS NULL;
```

### ❌ Forgetting `WHERE` in UPDATE/DELETE
```sql
-- DANGER — updates every row!
UPDATE employees SET salary = 0;

-- Safe — always verify with SELECT first
SELECT * FROM employees WHERE id = 42;
UPDATE employees SET salary = 0 WHERE id = 42;
```

### ❌ `NOT IN` with NULLs in subquery
```sql
-- DANGER — returns no rows if subquery has any NULL
SELECT * FROM a WHERE id NOT IN (SELECT foreign_id FROM b);

-- SAFE
SELECT * FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.foreign_id = a.id);
```

### ❌ String comparison with wrong quotes
```sql
-- WRONG — double quotes are for identifiers (column/table names)
SELECT * FROM employees WHERE name = "Alice";

-- CORRECT — single quotes for string literals
SELECT * FROM employees WHERE name = 'Alice';
```

### ❌ Relying on unordered results
```sql
-- WRONG — PostgreSQL does NOT guarantee row order without ORDER BY
SELECT * FROM employees LIMIT 5;

-- CORRECT — always specify ORDER BY when order matters
SELECT * FROM employees ORDER BY id LIMIT 5;
```

### ❌ `SELECT *` in production code
Avoid `SELECT *` in application queries. Fetch only the columns you need. `SELECT *` breaks if columns are added/reordered and sends unnecessary data over the network.

### ❌ Not using transactions for multi-step operations
Any sequence of writes (`INSERT`/`UPDATE`/`DELETE`) that must succeed or fail together **must** be wrapped in a `BEGIN ... COMMIT` block. Without it, a crash halfway through leaves the database in an inconsistent state.

---

## Quick Reference: Connection String Format

```
postgresql://[user[:password]@][host][:port][/dbname][?param=value]

# Examples
postgresql://myuser:mypass@localhost:5432/mydb
postgresql://myuser@/mydb                        # Unix socket, peer auth
postgresql://myuser:mypass@192.168.1.10:5432/mydb?sslmode=require
```

Environment variables (recognized by `psql` and `libpq`):

```bash
export PGHOST=localhost
export PGPORT=5432
export PGDATABASE=mydb
export PGUSER=myuser
export PGPASSWORD=mypassword   # Not recommended in scripts; use .pgpass instead
```

### `.pgpass` file (secure password storage)

```
# ~/.pgpass — format: hostname:port:database:username:password
localhost:5432:mydb:myuser:mypassword
```

```bash
chmod 600 ~/.pgpass   # Required — PostgreSQL ignores the file if permissions are too open
```

---

*Last updated: June 2026*
