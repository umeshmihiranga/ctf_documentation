
# 🔐 Local Authority — picoCTF

## Challenge

**Name:** Local Authority  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Platform:** picoCTF  
**Event:** picoCTF 2022  

---

## 🎯 Objective

The challenge provides a login page and asks:

> Can you get the flag?

The goal was to investigate the web application and find a way to authenticate as an administrator.

---

## 🧠 Initial Observation

The website presented a login page:

```text
Login Page

Username:
Password:
[Login]
```

The page also indicated that only letters and numbers were allowed for the username and password.

Instead of immediately trying to guess credentials, I inspected how the login functionality worked.

---

## 🔎 Enumeration

I opened the browser's **Developer Tools** and inspected the loaded resources.

The website contained:

```text
login.php
secure.js
style.css
```

The interesting file was:

```text
secure.js
```

Looking at the JavaScript revealed the authentication function:

```javascript
function checkPassword(username, password)
{
    if (username == 'admin' && password == 'strongPassword098765')
    {
        return true;
    }
    else
    {
        return false;
    }
}
```

This immediately exposed the administrator credentials:

```text
Username: admin
Password: strongPassword098765
```

---

## 💥 Vulnerability

The application stored the authentication credentials directly inside a **client-side JavaScript file**.

### Vulnerability Type

**Client-Side Authentication / Hardcoded Credentials**

The problem is that JavaScript executed by the browser is visible to the user.

Anything placed in client-side JavaScript should therefore be considered public.

For example:

```javascript
if (username == 'admin' && password == 'strongPassword098765')
```

does NOT securely protect the credentials.

An attacker can simply:

1. Open Developer Tools
2. Inspect JavaScript files
3. Search for authentication logic
4. Recover the credentials

---

## 🚀 Exploitation

### Step 1 — Open the login page

The application initially displayed the login form.

### Step 2 — Inspect Developer Tools

I opened:

```text
Developer Tools → Sources
```

and inspected:

```text
secure.js
```

### Step 3 — Recover credentials

The JavaScript contained:

```text
admin
strongPassword098765
```

Therefore:

```text
Username: admin
Password: strongPassword098765
```

### Step 4 — Authenticate

I entered the discovered credentials into the login form.

### Step 5 — Access the admin page

After authentication, the application redirected to:

```text
/admin.php
```

The page displayed the flag.

---

## 🚩 Flag

```text
picoCTF{j5_15_7r4n5p4r3n7_05df90c8}
```

---

## 🛠️ Tools Used

* Web Browser
* Chrome/Brave Developer Tools
* Sources panel
* JavaScript source inspection

---

## 📌 Attack Path

```text
Login Page
     │
     ▼
Inspect Developer Tools
     │
     ▼
Find secure.js
     │
     ▼
Inspect JavaScript
     │
     ▼
Credentials exposed
     │
     ├── Username: admin
     └── Password: strongPassword098765
     │
     ▼
Login
     │
     ▼
/admin.php
     │
     ▼
Flag
```

---

## 🧩 Why the Vulnerability Existed

The developer attempted to implement authentication using JavaScript running in the user's browser.

The problem is that the browser is controlled by the user.

Therefore, the user can inspect:

* JavaScript
* HTML
* CSS
* API requests
* Client-side logic
* Hardcoded credentials

Client-side code should never contain secrets.

---

## 🔐 Why This Is Insecure

A secure authentication system should perform authentication on the **server**.

### ❌ Vulnerable approach

```javascript
function checkPassword(username, password)
{
    if (username == 'admin' &&
        password == 'strongPassword098765')
    {
        return true;
    }

    return false;
}
```

The password is literally delivered to the attacker.

### ✅ Better approach

The browser should send credentials to the server over HTTPS:

```text
Browser
   │
   │ username + password
   ▼
Server
   │
   │ verify password
   ▼
Authentication Result
```

The server should compare the supplied password against a securely stored password hash.

The actual password should never be embedded in frontend JavaScript.

---

## 🛡️ How to Prevent It

### 1. Perform authentication server-side

Never rely on JavaScript for security decisions.

### 2. Never hardcode credentials in frontend code

Avoid:

```javascript
const password = "secret123";
```

because users can inspect it.

### 3. Store passwords securely

Passwords should be stored using strong password hashing algorithms such as:

```text
Argon2
bcrypt
scrypt
```

### 4. Use HTTPS

Credentials should be transmitted over HTTPS to protect them in transit.

### 5. Implement server-side authorization

Even if someone manually visits:

```text
/admin.php
```

the server should verify that the user has the required privileges.

---

## 💡 What I Learned

This challenge taught me an important web security principle:

> **Anything sent to the client should be considered visible to the attacker.**

I learned to:

* Inspect JavaScript files during web enumeration
* Use Developer Tools to understand application logic
* Look for client-side authentication mechanisms
* Identify hardcoded credentials
* Understand why client-side authentication is insecure
* Distinguish authentication logic from proper server-side authorization

---

## 🧠 Key Takeaway

When testing a web application, don't only look at the visible webpage.

Always investigate:

```text
HTML
CSS
JavaScript
Network Requests
Cookies
Local Storage
API Endpoints
Source Files
```

In this challenge, the password wasn't hidden behind a complicated exploit.

It was simply **sent to the browser in JavaScript**.

```text
If the browser can read it,
the attacker can read it too.
```


## 🏁 Conclusion

The challenge was solved by performing basic web enumeration rather than brute forcing the login.

The vulnerable design exposed administrator credentials in client-side JavaScript, allowing authentication and access to the protected admin page.

**Core vulnerability:**

```text
Client-side authentication + hardcoded credentials
```

**Result:**

```text
picoCTF{j5_15_7r4n5p4r3n7_05df90c8}
```


