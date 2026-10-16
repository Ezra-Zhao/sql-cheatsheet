# SQL Cheatsheet — Queries That Actually Work

One page. Thirty statements. Each entry tells you **what it does**, **a statement you can copy-paste**, and **tags**. Standard SQL throughout, with MySQL / PostgreSQL dialect differences flagged inline. **Every statement was actually executed and verified against SQLite 3.45** — no made-up functions, no untested syntax.

> 🌍 Read this project in your language: [中文](README.md) · [Español](README.es.md) · [Português](README.pt.md)

## Test data (every example runs against this)

```sql
CREATE TABLE departments (id INTEGER PRIMARY KEY, name TEXT NOT NULL);
CREATE TABLE employees (id INTEGER PRIMARY KEY, name TEXT NOT NULL, dept_id INTEGER, salary INTEGER, manager_id INTEGER, hired_on TEXT);
CREATE TABLE orders (id INTEGER PRIMARY KEY, emp_id INTEGER, amount REAL, ordered_on TEXT);

INSERT INTO departments VALUES (1, 'Engineering'), (2, 'Sales'), (3, 'HR');
INSERT INTO employees VALUES
  (1, 'Alice', 1, 120000, NULL, '2019-03-01'),
  (2, 'Bob',   1,  95000, 1,    '2020-07-15'),
  (3, 'Carol', 1,  88000, 1,    '2021-01-10'),
  (4, 'David', 2,  72000, NULL, '2018-05-20'),
  (5, 'Eve',   2,  68000, 4,    '2022-09-01'),
  (6, 'Frank', 3,  54000, NULL, '2021-11-30'),
  (7, 'Grace', NULL, 60000, NULL, '2023-02-14');
INSERT INTO orders VALUES
  (101, 2, 250.00, '2024-01-05'),
  (102, 2, 120.50, '2024-01-20'),
  (103, 4, 980.00, '2024-02-02'),
  (104, 5, 430.75, '2024-02-14'),
  (105, 4, 150.00, '2024-03-01'),
  (106, 1, 2000.00, '2024-03-11'),
  (107, 5, 89.99,  '2024-03-15');
```

Note: Grace has no department (`dept_id` is NULL), and Carol / Frank / Grace have no orders — several examples exist precisely to make that visible.

---

## 🔍 Query Basics

### `SELECT` — pick the columns you need
```sql
SELECT name, salary FROM employees;
-- 7 rows. Avoid SELECT * in real code: naming columns is clearer and faster
```
🏷️ `#query` `#basics`
### `WHERE` — filter rows
```sql
SELECT name, salary FROM employees WHERE salary > 80000 ORDER BY salary DESC;
-- Alice 120000 / Bob 95000 / Carol 88000
```
🏷️ `#query` `#filtering`
### `ORDER BY ... LIMIT` — top N
```sql
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 3;
-- Top 3 salaries: Alice / Bob / Carol
-- LIMIT works in MySQL / PostgreSQL / SQLite; SQL Server uses TOP 3, the SQL:2008 spelling FETCH FIRST 3 ROWS ONLY works in Oracle and PostgreSQL 10+
```
🏷️ `#query` `#sorting` `#pagination`
### `DISTINCT` — deduplicate
```sql
SELECT DISTINCT dept_id FROM employees;
-- 1 / 2 / 3 / NULL (Grace has no department; NULL counts as a value here)
```
🏷️ `#query` `#dedupe`
### `LIKE` — pattern matching
```sql
SELECT name FROM employees WHERE name LIKE 'A%';
-- Alice. % matches any run of characters, _ matches exactly one
-- Dialect: LIKE is case-insensitive for ASCII in SQLite / MySQL, but case-sensitive in PostgreSQL.
--          PostgreSQL also has ILIKE for case-insensitive matching (PG only; MySQL / SQLite don't have it)
```
🏷️ `#query` `#filtering` `#strings`
### `IN` / `BETWEEN` — set membership and ranges
```sql
SELECT name, salary FROM employees
WHERE dept_id IN (1, 2) AND salary BETWEEN 70000 AND 100000
ORDER BY salary DESC;
-- Bob 95000 / Carol 88000 / David 72000 (BETWEEN includes both endpoints)
```
🏷️ `#query` `#filtering`

## 📊 Aggregation & Grouping

### `COUNT` / `SUM` / `AVG` — basic aggregates
```sql
SELECT COUNT(*) AS headcount, SUM(salary) AS total_payroll, AVG(salary) AS avg_salary FROM employees;
-- 7 people / total 557000 / average 79571.43
```
🏷️ `#aggregation`
### `GROUP BY` — per-group stats
```sql
SELECT dept_id, COUNT(*) AS n, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id;
-- dept 1: 3 people, avg 101000; dept 2: 2, 70000; dept 3: 1, 54000; NULL group: 1, 60000
```
🏷️ `#aggregation` `#grouping`
### `HAVING` — filter groups, not rows
```sql
SELECT dept_id, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id HAVING AVG(salary) > 60000;
-- Only dept 1 (101000) and dept 2 (70000) survive
-- WHERE filters rows before grouping; HAVING filters groups after. Don't mix them up
```
🏷️ `#aggregation` `#grouping` `#filtering`
### `COUNT(DISTINCT ...)` — count unique values
```sql
SELECT COUNT(DISTINCT dept_id) AS dept_count FROM employees;
-- 3. COUNT skips NULLs, so Grace's missing department isn't counted
```
🏷️ `#aggregation` `#dedupe`
### `CASE WHEN` — conditional aggregation
```sql
SELECT COUNT(*) AS total,
       SUM(CASE WHEN salary >= 90000 THEN 1 ELSE 0 END) AS high_earners
FROM employees;
-- total 7 / high earners 2 (Alice, Bob). Several metrics with different rules in one pass
```
🏷️ `#aggregation` `#conditional`

## 🔗 JOINs

### `INNER JOIN` — matching rows from both tables
```sql
SELECT e.name, d.name AS dept FROM employees e JOIN departments d ON e.dept_id = d.id;
-- 6 rows: Grace has no department and drops out
```
🏷️ `#join`
### `LEFT JOIN` — keep every row of the left table
```sql
SELECT e.name, d.name AS dept FROM employees e LEFT JOIN departments d ON e.dept_id = d.id;
-- 7 rows: Grace is there with dept = NULL. Full roster → LEFT JOIN; only matched → INNER JOIN
```
🏷️ `#join`
### `LEFT JOIN ... IS NULL` — rows with no match
```sql
SELECT e.name FROM employees e LEFT JOIN orders o ON o.emp_id = e.id WHERE o.id IS NULL;
-- Carol / Frank / Grace: employees who never ordered. The classic orphan-finder
```
🏷️ `#join` `#debugging`
### Self join — a table joined to itself
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e LEFT JOIN employees m ON e.manager_id = m.id;
-- Bob→Alice, Carol→Alice, Eve→David; people without a manager get NULL
-- Org charts, threaded comments, any hierarchy stored in one table works this way
```
🏷️ `#join` `#hierarchy`
### Three-table JOIN + aggregation — orders per department
```sql
SELECT d.name AS dept, COUNT(o.id) AS order_count, SUM(o.amount) AS revenue
FROM departments d
JOIN employees e ON e.dept_id = d.id
JOIN orders o ON o.emp_id = e.id
GROUP BY d.id, d.name;
-- Engineering: 3 orders, 2370.50; Sales: 4 orders, 1650.74; HR has no orders and doesn't appear (INNER JOIN)
```
🏷️ `#join` `#aggregation` `#real-world`

## 🧩 Subqueries & CTEs

### Scalar subquery — compare against an overall average
```sql
SELECT name, salary FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);
-- Alice / Bob / Carol: everyone paid above the company average (79571.43)
```
🏷️ `#subquery`
### `IN` subquery — filter by another table's result
```sql
SELECT name FROM employees WHERE id IN (SELECT emp_id FROM orders WHERE amount > 500);
-- Alice / David: employees who placed an order over 500
```
🏷️ `#subquery`
### `EXISTS` — existence check
```sql
SELECT d.name FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.id AND e.salary > 100000);
-- Engineering: departments with at least one employee earning over 100k.
-- EXISTS only cares whether a row exists — a common pick for large tables
```
🏷️ `#subquery`
### `WITH` (CTE) — name an intermediate result
```sql
WITH dept_avg AS (
  SELECT dept_id, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id
)
SELECT e.name, e.salary FROM employees e
JOIN dept_avg a ON e.dept_id = a.dept_id
WHERE e.salary > a.avg_salary;
-- Alice (above dept avg 101000) / David (above 70000): paid more than their own department's average
-- CTEs make complex queries read top-to-bottom instead of inside-out; MySQL 8.0+ / PostgreSQL / SQLite 3.8.8+ support them
```
🏷️ `#cte` `#subquery`
### Window function `RANK()` — ranking inside groups
```sql
SELECT name, dept_id, salary,
       RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
FROM employees;
-- In dept 1: Alice 1, Bob 2, Carol 3; in dept 2: David 1, Eve 2
-- GROUP BY collapses rows; window functions keep every row and attach a rank — that's the key difference
-- Dialect: MySQL 8.0+ / PostgreSQL / SQLite 3.25+. Older MySQL can't do this
```
🏷️ `#window-functions` `#ranking`

## ✏️ Writes & Transactions

### `INSERT INTO` — add a row
```sql
INSERT INTO employees (id, name, dept_id, salary, manager_id, hired_on)
VALUES (8, 'Henry', 2, 71000, 4, '2024-06-01');
-- Always name the columns; multi-row inserts chain more (...) groups after VALUES
```
🏷️ `#writes`
### `UPDATE ... WHERE` — change rows
```sql
UPDATE employees SET salary = salary + 5000 WHERE name = 'Eve';
-- Eve: 68000 → 73000, exactly 1 row affected
-- ⚠️ Forget the WHERE and you update the whole table. Run a SELECT with the same condition first to count the rows
```
🏷️ `#writes` `#danger`
### `DELETE ... WHERE` — remove rows
```sql
DELETE FROM orders WHERE amount < 100;
-- Removes 1 row (the 89.99 order). Without WHERE it empties the table
-- Business data is usually soft-deleted instead (a deleted_at flag), not hard-DELETEd
```
🏷️ `#writes` `#danger`
### Transactions — all or nothing
```sql
BEGIN;
UPDATE employees SET dept_id = 2 WHERE name = 'Frank';
COMMIT;
-- Frank moves from HR to Sales. If any step fails, ROLLBACK undoes everything as if it never happened
-- Money transfers and stock deductions must run inside a transaction. In MySQL, START TRANSACTION is the same as BEGIN
```
🏷️ `#transactions` `#writes`
### `UPSERT` — update if present, insert if not
```sql
INSERT INTO departments (id, name) VALUES (3, 'People')
ON CONFLICT(id) DO UPDATE SET name = excluded.name;
-- id=3 already exists → the department is renamed HR → People; if it didn't exist, it would be inserted
-- This is the SQLite / PostgreSQL spelling. MySQL: ON DUPLICATE KEY UPDATE name = VALUES(name) — but VALUES() is deprecated since MySQL 8.0.20; modern spelling uses a row alias: ... VALUES (3, 'People') AS new ON DUPLICATE KEY UPDATE name = new.name
```
🏷️ `#writes` `#dialects`

## 🗂️ Indexes & Schema

### `CREATE TABLE` — define a table
```sql
CREATE TABLE projects (
  id INTEGER PRIMARY KEY,
  title TEXT NOT NULL,
  lead_id INTEGER REFERENCES employees(id)
);
-- Primary keys, NOT NULL and foreign keys belong in the schema, not in application code
-- SQLite note: foreign keys are parsed but NOT enforced unless you run PRAGMA foreign_keys = ON
-- Dialect: INTEGER PRIMARY KEY is SQLite's auto-incrementing row id;
--          MySQL writes INT AUTO_INCREMENT PRIMARY KEY;
--          PostgreSQL writes INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY (or the older SERIAL)
```
🏷️ `#ddl` `#schema`
### `ALTER TABLE` — add a column
```sql
ALTER TABLE employees ADD COLUMN email TEXT;
-- New columns start as NULL. Backfill first, add NOT NULL later — small steps avoid long table locks
```
🏷️ `#ddl`
### `CREATE INDEX` — speed up frequent lookups
```sql
CREATE INDEX idx_employees_dept ON employees(dept_id);
-- After this, WHERE dept_id = ? uses the index instead of scanning the whole table
-- Verify: EXPLAIN QUERY PLAN in SQLite, EXPLAIN in MySQL / PostgreSQL
-- Every index slows writes down, so index the columns you actually filter on — not everything
```
🏷️ `#ddl` `#index` `#performance`
### `DROP` — remove an index or a table
```sql
DROP INDEX idx_employees_dept;
-- MySQL spelling: DROP INDEX idx_employees_dept ON employees;
DROP TABLE projects;
-- ⚠️ Irreversible. Check for foreign keys pointing at the table first, and back up before you run it in production
```
🏷️ `#ddl` `#danger`

---

Found an error, a wrong dialect note, or a missing everyday query? PRs are welcome — keep the format (one-line description / sql block / tags) and **run the statement against a real database before submitting**. Unverified statements don't get in.

## More cheatsheets

- [ffmpeg-cheatsheet](https://github.com/Ezra-Zhao/ffmpeg-cheatsheet) — 28 FFmpeg commands
- [git-cheatsheet](https://github.com/Ezra-Zhao/git-cheatsheet) — 30 daily + rescue Git commands
- [docker-cheatsheet](https://github.com/Ezra-Zhao/docker-cheatsheet) — 28 Docker commands
- [curl-cheatsheet](https://github.com/Ezra-Zhao/curl-cheatsheet) — 29 curl commands for API debugging
- [regex-cheatsheet](https://github.com/Ezra-Zhao/regex-cheatsheet) — 30 real-world regex patterns
- [jq-cheatsheet](https://github.com/Ezra-Zhao/jq-cheatsheet) — 30 jq filters for JSON
- [terminal-superpowers-cheatsheet](https://github.com/Ezra-Zhao/terminal-superpowers-cheatsheet) — 14 modern terminal tools
- [adobe-free-alternatives-cheatsheet](https://github.com/Ezra-Zhao/adobe-free-alternatives-cheatsheet) — 16 free Adobe alternatives
- [desubscribe-cheatsheet](https://github.com/Ezra-Zhao/desubscribe-cheatsheet) — 10 subscription swaps, ≈ $1,260/year saved
- [kubectl-cheatsheet](https://github.com/Ezra-Zhao/kubectl-cheatsheet) — everyday Kubernetes commands
- [ssh-cheatsheet](https://github.com/Ezra-Zhao/ssh-cheatsheet) — logins, tunnels and file transfer

## License

MIT © 2026 Guangyi Zhao — see [LICENSE](LICENSE).
