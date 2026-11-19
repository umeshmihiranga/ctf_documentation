# SQL Direct

**Category:** Database Security / PostgreSQL  
**Platform:** CyLab Security Academy  
**Difficulty:** Beginner  
**Target:** `xebec.cylabacademy.net:35768`

## 1. Challenge Description

The challenge provides direct access to a PostgreSQL database server. The objective is to connect to the database, explore its structure, and retrieve the hidden flag.

### Connection Details

| Parameter | Value |
|---|---|
| Host | `xebec.cylabacademy.net` |
| Port | `35768` |
| Database | `pico` |
| Username | `postgres` |
| Password | `postgres` |
| Database type | PostgreSQL |

---

## 2. Initial Connection

Connect using `psql`, the PostgreSQL command-line client.

```bash
psql -h xebec.cylabacademy.net -p 35768 -U postgres pico
```

Enter the password when prompted:

```text
postgres
```

Successful connection:

```text
psql (18.6 (Debian 18.6-3))
Type "help" for help.

pico=#
```

---

## 3. Database Enumeration

### List available databases

```sql
\l
```

The server contained the following databases:

- `pico`
- `postgres`
- `template0`
- `template1`

The challenge database was `pico`.

### List tables

```sql
\dt
```

Output:

```text
 Schema | Name  | Type  | Owner
--------+-------+-------+----------
 public | flags | table | postgres
```

We discovered a table named `flags` in the `public` schema.

---

## 4. Table Structure Enumeration

Inspect the structure of the table:

```sql
\d flags
```

The table contained the following columns:

| Column | Data Type | Description |
|---|---|---|
| `id` | `integer` | Record identifier |
| `firstname` | `varchar(255)` | First name |
| `lastname` | `varchar(255)` | Last name |
| `address` | `varchar(255)` | Address |

The `id` column was the primary key.

---

## 5. Data Extraction

Retrieve the records from the table:

```sql
SELECT * FROM flags;
```

We initially limited the output to ten rows:

```sql
SELECT * FROM flags LIMIT 10;
```

The database returned three records:

| ID | Firstname | Lastname | Address |
|---|---|---|---|
| 1 | Luke | Skywalker | `academy{L3arN_S0m3_5qL_t0d4Y_412c85d9}` |
| 2 | Leia | Organa | Alderaan |
| 3 | Han | Solo | Corellia |

The flag was stored inside the `address` column of the first record.

---

## 6. Flag

```text
academy{L3arN_S0m3_5qL_t0d4Y_412c85d9}
```

---

## 7. Attack Methodology

```text
Identify exposed PostgreSQL service
          |
          v
Connect using provided credentials
          |
          v
Enumerate available databases
          |
          v
Identify challenge database: pico
          |
          v
Enumerate tables
          |
          v
Discover table: flags
          |
          v
Inspect table columns
          |
          v
Retrieve records
          |
          v
Extract flag from address column
```

---

## 8. Commands Learned

| Command | Purpose |
|---|---|
| `psql -h HOST -p PORT -U USER DATABASE` | Connect to PostgreSQL |
| `\l` | List databases |
| `\dt` | List tables |
| `\d table_name` | Describe table structure |
| `SELECT * FROM table_name;` | Retrieve all records |
| `SELECT * FROM table_name LIMIT 10;` | Limit query results |

---

## 9. Key Concepts Learned

### PostgreSQL

PostgreSQL is an open-source relational database management system (RDBMS). It stores data in databases, schemas, tables, rows, and columns.

### Database Enumeration

Database enumeration is the process of discovering the available databases, tables, columns, and stored information.

### SQL Queries

SQL (Structured Query Language) is used to interact with relational databases.

### Primary Key

A primary key uniquely identifies each record in a table. In this challenge, the `id` column was the primary key.

### Database Exposure

A database service exposed to a network, especially with weak or default credentials, can allow unauthorized users to access sensitive information.

---

## 10. Security Lessons

- Avoid exposing database services directly to the public internet unless absolutely necessary.
- Never use default or weak passwords.
- Restrict database access using firewalls and network segmentation.
- Follow the principle of least privilege.
- Avoid storing sensitive information in unnecessary database fields.
- Monitor database authentication attempts and access logs.

---

## 11. Conclusion

This challenge demonstrated how exposed database services and weak credentials can allow direct access to sensitive information.

By connecting to PostgreSQL, enumerating the database structure, inspecting the `flags` table, and retrieving its records, we successfully located the flag without exploiting a SQL injection vulnerability.
