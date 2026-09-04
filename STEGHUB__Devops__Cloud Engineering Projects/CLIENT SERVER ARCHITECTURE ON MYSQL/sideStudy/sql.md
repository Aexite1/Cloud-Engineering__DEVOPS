# SQL Fundamentals: Common MySQL Commands

## Introduction

SQL provides the commands used to interact with relational databases. This guide covers five fundamental MySQL commands:

- `SHOW`
- `CREATE`
- `DROP`
- `SELECT`
- `INSERT`

The examples below demonstrate what each command does and how its basic syntax is structured.

---

## 1. SHOW

### What it does

`SHOW` is used to inspect information that exists inside a MySQL environment. Depending on the form used, it can display databases, tables, or the columns belonging to a table.

### General form

```sql
SHOW {DATABASES | TABLES | COLUMNS FROM table_name};
```

### Examples

Display all databases:

```sql
SHOW DATABASES;
```

Display tables in the currently selected database:

```sql
SHOW TABLES;
```

Display the columns defined for a table:

```sql
SHOW COLUMNS FROM table_name;
```

---

## 2. CREATE

### What it does

`CREATE` is used when a new database object needs to be created. Common examples include creating a database or defining a new table.

### Create a database

```sql
CREATE DATABASE database_name;
```

Example:

```sql
CREATE DATABASE my_database;
```

### Create a table

A table definition specifies the column names, their data types, and any required constraints.

```sql
CREATE TABLE table_name (
    column1_name data_type constraints,
    column2_name data_type constraints,
    ...
);
```

Example:

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    position VARCHAR(50),
    salary DECIMAL(10, 2)
);
```

---

## 3. DROP

### What it does

`DROP` removes a database object. Unlike merely removing rows from a table, dropping a database or table removes the object itself.

Because the operation is destructive, it should be used carefully.

### Remove a database

```sql
DROP DATABASE database_name;
```

Example:

```sql
DROP DATABASE my_database;
```

### Remove a table

```sql
DROP TABLE table_name;
```

Example:

```sql
DROP TABLE employees;
```

---

## 4. SELECT

### What it does

`SELECT` retrieves information stored in database tables. It can return every column, selected columns, or records that satisfy a condition.

### Basic structure

```sql
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

### Examples

Retrieve every column:

```sql
SELECT * FROM employees;
```

Retrieve only selected columns:

```sql
SELECT name, position
FROM employees;
```

Retrieve records matching a condition:

```sql
SELECT name, salary
FROM employees
WHERE position = 'Manager';
```

The `WHERE` clause limits the returned rows according to the specified condition.

---

## 5. INSERT

### What it does

`INSERT` adds new records to an existing table.

### Basic structure

```sql
INSERT INTO table_name (column1, column2, ...)
VALUES (value1, value2, ...);
```

### Insert one record

```sql
INSERT INTO employees (id, name, position, salary)
VALUES (1, 'John Doe', 'Manager', 75000.00);
```

### Insert several records

Multiple rows can be supplied in the same statement:

```sql
INSERT INTO employees (id, name, position, salary)
VALUES
    (2, 'Jane Smith', 'Developer', 65000.00),
    (3, 'Emily Johnson', 'Designer', 60000.00);
```

---

## Summary

These five commands provide a useful starting point for working with MySQL:

- `SHOW` helps inspect database structures and information.
- `CREATE` establishes databases and tables.
- `DROP` removes databases or tables.
- `SELECT` reads stored information.
- `INSERT` adds new records.

Understanding these commands gives a beginner a foundation for performing common database management and data manipulation tasks.
