# SQL (Structured Query Language)

## SQL vs NoSQL

| | SQL | NoSQL |
|---|---|---|
| **Structure** | Tables | Documents, key-value, graphs |
| **Schema** | Fixed | Flexible |
| **Language** | SQL | Varies |
| **ACID** | Yes | Often no |
| **Scaling** | Vertical | Horizontal |
| **Examples** | MySQL, PostgreSQL | MongoDB, Redis, Neo4j |

---

## SQL Sublanguages

| Sublanguage | Full Name | Purpose |
|---|---|---|
| **DDL** | Data Definition Language | Structure — tables, indexes |
| **DML** | Data Manipulation Language | Data — insert, read, update |
| **DQL** | Data Query Language | Querying data |
| **DCL** | Data Control Language | Access control |
| **TCL** | Transaction Control Language | Transactions |

---

## DDL — Data Definition Language

| Command | What it does |
|---|---|
| `CREATE` | Create database/table/index |
| `ALTER` | Modify existing structure |
| `DROP` | Delete structure permanently |
| `TRUNCATE` | Clear all data, keep structure |

### CREATE

```sql
CREATE DATABASE university;

CREATE TABLE students (
  student_id INT PRIMARY KEY AUTO_INCREMENT,
  name       VARCHAR(100),
  age        INT,
  gpa        DECIMAL(3,2)
);
```

### ALTER

```sql
ALTER TABLE students ADD phone VARCHAR(20);          -- add column
ALTER TABLE students MODIFY phone VARCHAR(50);       -- change column type
ALTER TABLE students DROP COLUMN phone;              -- remove column
ALTER TABLE students RENAME COLUMN name TO fullname; -- rename column
ALTER TABLE students ADD CONSTRAINT chk_gpa CHECK (gpa >= 0 AND gpa <= 4);
```

### DROP

```sql
DROP TABLE students;
DROP TABLE IF EXISTS students;   -- safe version, no error if not exists
DROP DATABASE university;
DROP INDEX idx_name ON students;
```

### TRUNCATE

```sql
TRUNCATE TABLE students;  -- deletes all rows, keeps table structure
```

**TRUNCATE vs DELETE:**

| | TRUNCATE | DELETE |
|---|---|---|
| **Filters** | No — removes all rows | Yes — supports `WHERE` |
| **Speed** | Fast | Slower |
| **Logging** | Minimal | Full |
| **Rollback** | Cannot | Can |
| **AUTO_INCREMENT** | Resets | Does not reset |

---

## DML — Data Manipulation Language

| Command | What it does |
|---|---|
| `SELECT` | Read data |
| `INSERT` | Insert data |
| `UPDATE` | Update data |
| `DELETE` | Delete rows |

### SELECT

```sql
SELECT * FROM students WHERE age = 20;
SELECT name, gpa FROM students WHERE age > 18 ORDER BY gpa DESC;
```

### INSERT

```sql
INSERT INTO students (name, age) VALUES ('John', 20);
INSERT INTO students (name, age) VALUES ('Alice', 22), ('Bob', 21);  -- multiple rows
```

### UPDATE

```sql
UPDATE students SET age = 21 WHERE student_id = 1;
UPDATE students SET gpa = 4.0 WHERE name = 'Alice';
```

### DELETE

```sql
DELETE FROM students WHERE age < 18;
DELETE FROM students;  -- deletes all rows (unlike TRUNCATE, supports rollback)
```

---

## DQL — Data Query Language

DQL is technically part of DML but often treated separately. It's just `SELECT` and its clauses.

| Clause | What it does |
|---|---|
| `FROM` | Which table to read from |
| `WHERE` | Filter rows **before** grouping/aggregation |
| `GROUP BY` | Group rows with the same value in a column |
| `HAVING` | Filter results **after** GROUP BY (like WHERE but for groups) |
| `ORDER BY` | Sort results |
| `DISTINCT` | Remove duplicate rows |
| `LIMIT` | Return only N rows |

**Execution order** (how SQL actually processes a query):
```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

```sql
SELECT age, COUNT(*) AS total
FROM students
WHERE gpa > 2.0
GROUP BY age
HAVING COUNT(*) > 5
ORDER BY age DESC
LIMIT 10;
```

---

## DCL — Data Control Language

| Command | What it does |
|---|---|
| `GRANT` | Give permissions |
| `REVOKE` | Remove permissions |

### GRANT

```sql
GRANT SELECT ON students TO user1;                         -- single permission
GRANT SELECT, INSERT, UPDATE ON students TO user1;         -- multiple permissions
GRANT ALL PRIVILEGES ON students TO user1;                 -- all permissions
GRANT CREATE USER TO admin;                                -- admin permission
GRANT SELECT ON students TO user1 WITH GRANT OPTION;       -- user can also grant to others
```

### REVOKE

```sql
REVOKE INSERT ON students FROM user1;
REVOKE ALL PRIVILEGES ON students FROM user1;
```

---

## TCL — Transaction Control Language

| Command | What it does |
|---|---|
| `BEGIN` / `START TRANSACTION` | Start a transaction |
| `COMMIT` | Save all changes |
| `ROLLBACK` | Undo all changes |
| `SAVEPOINT` | Create a checkpoint inside a transaction |

### Basic transaction

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE acc_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE acc_id = 2;

COMMIT;    -- if everything is fine
-- or
ROLLBACK;  -- if something went wrong
```

### SAVEPOINT

Useful when you want to roll back only part of a transaction, not all of it.

```sql
START TRANSACTION;

INSERT INTO orders (...) VALUES (...);
SAVEPOINT order_created;             -- checkpoint after order is created

INSERT INTO order_items (...) VALUES (...);
INSERT INTO order_items (...) VALUES (...);
-- something went wrong with items

ROLLBACK TO order_created;           -- undo only the items, keep the order
COMMIT;
```
