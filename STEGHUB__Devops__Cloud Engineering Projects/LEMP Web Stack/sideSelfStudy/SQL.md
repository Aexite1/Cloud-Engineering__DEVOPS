# Basic SQL Syntax & Most Commonly Used Commands (Beginner's Guide)

SQL (**Structured Query Language**) is the standard language used to communicate with relational databases such as MySQL, PostgreSQL, SQL Server, SQLite, and Oracle.

---

# 1. SQL Syntax Basics

### SQL keywords are usually written in uppercase.

```sql
SELECT * FROM students;
```

* `SELECT` → Retrieves data.
* `*` → Means "all columns."
* `FROM` → Specifies the table.

---

### SQL is NOT case-sensitive

These are the same:

```sql
SELECT * FROM students;
```

```sql
select * from students;
```

However, writing keywords in **uppercase** improves readability.

---

### Every SQL statement ends with a semicolon (`;`)

```sql
SELECT * FROM students;
```

---

## Sample Table

Lets use this table:

**Students**

| id | name  | age | department       | cgpa |
| -- | ----- | --- | ---------------- | ---- |
| 1  | John  | 20  | Computer Science | 4.3  |
| 2  | Sarah | 19  | Mathematics      | 4.8  |
| 3  | Mike  | 21  | Computer Science | 3.9  |
| 4  | Anna  | 20  | Physics          | 4.5  |


---

# 2. SELECT (Read Data)

Retrieve data from a table.

### Syntax

```sql
SELECT column_name
FROM table_name;
```

### Select everything

```sql
SELECT *
FROM students;
```

Output:

| id | name  | age | department       | cgpa |
| -- | ----- | --- | ---------------- | ---- |
| 1  | John  | 20  | Computer Science | 4.3  |
| 2  | Sarah | 19  | Mathematics      | 4.8  |
| 3  | Mike  | 21  | Computer Science | 3.9  |
| 4  | Anna  | 20  | Physics          | 4.5  |

---

### Select specific columns

```sql
SELECT name, department
FROM students;
```

Output

| name  | department       |
| ----- | ---------------- |
| John  | Computer Science |
| Sarah | Mathematics      |
| Mike  | Computer Science |
| Anna  | Physics          |

---

# 3. WHERE

Filters records.

### Syntax

```sql
SELECT *
FROM students
WHERE condition;
```

Example

```sql
SELECT *
FROM students
WHERE age = 20;
```

Output

| name | age |
| ---- | --- |
| John | 20  |
| Anna | 20  |

---

## Comparison Operators

| Operator | Meaning               |
| -------- | --------------------- |
| =        | Equal                 |
| >        | Greater than          |
| <        | Less than             |
| >=       | Greater than or equal |
| <=       | Less than or equal    |
| != or <> | Not equal             |

Example

```sql
SELECT *
FROM students
WHERE cgpa > 4.0;
```

---

# 4. AND / OR / NOT

### AND

Both conditions must be true.

```sql
SELECT *
FROM students
WHERE age = 20
AND department = 'Physics';
```

---

### OR

Either condition can be true.

```sql
SELECT *
FROM students
WHERE department = 'Physics'
OR department = 'Mathematics';
```

---

### NOT

Opposite of a condition.

```sql
SELECT *
FROM students
WHERE NOT department = 'Physics';
```

---

# 5. ORDER BY

Sorts results.

### Ascending (default)

```sql
SELECT *
FROM students
ORDER BY cgpa;
```

---

### Descending

```sql
SELECT *
FROM students
ORDER BY cgpa DESC;
```

---

# 6. LIMIT

Returns only a certain number of rows.

```sql
SELECT *
FROM students
LIMIT 2;
```

Output

Only the first two rows.

---

# 7. DISTINCT

Removes duplicates.

```sql
SELECT DISTINCT department
FROM students;
```

Output

Computer Science

Mathematics

Physics

---

# 8. INSERT INTO

Adds new data.

### Syntax

```sql
INSERT INTO students
(id, name, age, department, cgpa)
VALUES
(5, 'David', 22, 'Engineering', 4.1);
```

---

# 9. UPDATE

Changes existing data.

### Syntax

```sql
UPDATE students
SET cgpa = 4.6
WHERE id = 3;
```

Without a `WHERE` clause, **every row will be updated**.
---

# 10. DELETE

Removes rows.

```sql
DELETE FROM students
WHERE id = 5;
```

Without a `WHERE` clause, **all rows will be deleted**.

---

# 11. CREATE TABLE

Creates a new table.

```sql
CREATE TABLE students (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    age INT,
    department VARCHAR(100),
    cgpa DECIMAL(3,2)
);
```

---

# 12. ALTER TABLE

Changes the table structure.

```sql
ALTER TABLE students
ADD email VARCHAR(100);
```

---

# 13. DROP TABLE

Deletes the entire table.

```sql
DROP TABLE students;
```

This removes both the table and all its data.

---

# 14. TRUNCATE TABLE

Deletes all rows but keeps the table structure (supported in many databases).

```sql
TRUNCATE TABLE students;
```

---

# 15. COUNT()

Counts rows.

```sql
SELECT COUNT(*)
FROM students;
```

Output

```
4
```

---

# 16. MAX()

Largest value.

```sql
SELECT MAX(cgpa)
FROM students;
```

Output

```
4.8
```

---

# 17. MIN()

Smallest value.

```sql
SELECT MIN(cgpa)
FROM students;
```

---

# 18. AVG()

Average value.

```sql
SELECT AVG(cgpa)
FROM students;
```

---

# 19. SUM()

Adds numbers.

```sql
SELECT SUM(age)
FROM students;
```

---

# 20. GROUP BY

Groups rows before applying aggregate functions.

```sql
SELECT department, COUNT(*)
FROM students
GROUP BY department;
```

This returns one row per department along with the number of students in each.


---

# 21. HAVING

Filters grouped results.

```sql
SELECT department, COUNT(*)
FROM students
GROUP BY department
HAVING COUNT(*) > 1;
```

---

# 22. LIKE

Searches for patterns.

Starts with J

```sql
SELECT *
FROM students
WHERE name LIKE 'J%';
```

Ends with a

```sql
SELECT *
FROM students
WHERE name LIKE '%a';
```

Contains oh

```sql
SELECT *
FROM students
WHERE name LIKE '%oh%';
```

---

# 23. IN

Checks if a value matches any item in a list.

```sql
SELECT *
FROM students
WHERE department IN ('Physics', 'Mathematics');
```

---

# 24. BETWEEN

Checks if a value falls within a range.

```sql
SELECT *
FROM students
WHERE age BETWEEN 19 AND 21;
```

---

# 25. IS NULL

Finds missing values.

```sql
SELECT *
FROM students
WHERE email IS NULL;
```

---

# 26. JOIN

Combines data from multiple tables.

Suppose we have another table:

**Courses**

| student_id | course            |
| ---------- | ----------------- |
| 1          | Database Systems  |
| 2          | Calculus          |
| 3          | Operating Systems |

```sql
SELECT students.name, courses.course
FROM students
INNER JOIN courses
ON students.id = courses.student_id;
```

This matches students with their courses.

---

# 27. PRIMARY KEY

A column that uniquely identifies each row.

```sql
CREATE TABLE students (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);
```

Rules:

* Cannot contain duplicate values.
* Cannot be `NULL`.

---

# 28. FOREIGN KEY

Links one table to another.

```sql
CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    student_id INT,
    FOREIGN KEY (student_id)
        REFERENCES students(id)
);
```

genui{"computing_relational_databases_learning_block_staging":{"type_id":"SQL_PRIMARY_FOREIGN_KEY_CONSTRAINTS"}}

---

# 29. SQL Command Categories

SQL commands are commonly grouped into two main categories:

* **DDL (Data Definition Language):** Defines or changes the database structure.

  * `CREATE`
  * `ALTER`
  * `DROP`
* **DML (Data Manipulation Language):** Works with the data stored in tables.

  * `SELECT`
  * `INSERT`
  * `UPDATE`
  * `DELETE`

genui{"computing_relational_databases_learning_block_staging":{"type_id":"SQL_DDL_VS_DML"}}

---

# 30. Transactions

Transactions ensure that related database operations succeed or fail together.

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

If something goes wrong before `COMMIT`, you can cancel all pending changes:

```sql
ROLLBACK;
```
