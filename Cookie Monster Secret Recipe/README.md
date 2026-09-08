# 🍪 Cookie Monster Secret Recipe

## Challenge

**Challenge:** Cookie Monster Secret Recipe
**Category:** Web Exploitation
**Difficulty:** Easy
**Platform:** picoCTF 2025
**Vulnerability:** Sensitive information exposed through a client-side cookie
**Flag:** `picoCTF{c00k1e_m0nster_l0ves_c00kies_2C8040EF}`

---

## Initial Clue

After opening the challenge instance, the web page displayed:

> "No need password. We just need cookies!"

It also provided the hint:

> "Have you checked your cookies lately?"

These clues strongly suggested that the application was using a browser cookie to store something important.

---

## Enumeration

The first step was to inspect the website rather than immediately trying to exploit it.

### 1. Inspect the page

The application returned an **Access Denied** page, but its message indicated that authentication depended on cookies.

### 2. Open browser Developer Tools

We opened:

```text
Developer Tools
    ↓
Application
    ↓
Cookies
```

A cookie named:

```text
secret_recipe
```

was present.

Its value was:

```text
cGljb0NURntjMDBrMWVfbTBuc3Rlcl9sMHZlc19jMDBraWVzXzJDODA0MEVGfQ%3D%3D
```

The cookie itself became the main target of our investigation.

---

## Identifying the Encoding

The end of the cookie contained:

```text
%3D%3D
```

`%3D` is URL encoding for:

```text
=
```

After URL decoding, the value became:

```text
cGljb0NURntjMDBrMWVfbTBuc3Rlcl9sMHZlc19jMDBraWVzXzJDODA0MEVGfQ==
```

The resulting string had the characteristics of **Base64**:

```text
A-Z
a-z
0-9
+
/
=
```

This suggested that the cookie wasn't encrypted — it was simply encoded.

---

## Exploit

We decoded the Base64 value using Kali Linux:

```bash
echo 'cGljb0NURntjMDBrMWVfbTBuc3Rlcl9sMHZlc19jMDBraWVzXzJDODA0MEVGfQ==' | base64 -d
```

The result was:

```text
picoCTF{c00k1e_m0nster_l0ves_c00kies_2C8040EF}
```

### Flag

```text
picoCTF{c00k1e_m0nster_l0ves_c00kies_2C8040EF}
```

---

## Tools

### Browser Developer Tools

Used to inspect:

```text
Application
└── Cookies
    └── secret_recipe
```

### Base64

Used to decode the cookie value:

```bash
base64 -d
```

### URL Encoding

The `%3D%3D` portion was URL-decoded before Base64 decoding.

---

## What I Learned

### 1. Always inspect cookies during web enumeration

Cookies can contain:

* Session identifiers
* User information
* Roles
* Authentication data
* Application state
* Sometimes even sensitive information

When a challenge gives clues about cookies, checking them should be one of the first steps.

### 2. Encoding is NOT encryption

This was the most important lesson.

The application used Base64:

```text
Secret
  ↓
Base64 encoding
  ↓
Encoded value
```

Anyone can reverse this:

```text
Encoded value
  ↓
Base64 decoding
  ↓
Secret
```

Base64 provides **no confidentiality**.

### 3. Client-side data should not automatically be trusted

The browser had access to the cookie, meaning the user could inspect and potentially modify it.

Anything stored on the client side should be treated as potentially visible and potentially controllable by the user.

### 4. Follow the clues before using complicated techniques

We didn't need:

```text
SQL Injection
SSTI
XSS
Brute Force
Directory Fuzzing
Password Cracking
```

The challenge essentially told us where to look:

```text
"we just need cookies"
          ↓
       Cookies
          ↓
   secret_recipe
          ↓
      Decode it
          ↓
        Flag
```

This is an important CTF habit: **start with enumeration and clues before assuming the vulnerability is complicated.**

---

# Why the Vulnerability Existed

The application stored sensitive information directly inside a client-side cookie:

```text
secret_recipe=<encoded-secret>
```

Even though the value looked unreadable, it was only Base64 encoded.

Therefore, the application effectively exposed the secret to the client.

The problem wasn't that Base64 was "broken."

The problem was **storing the secret in the browser in the first place**.

---

# How to Prevent It

Sensitive information should not be stored directly in client-controlled cookies.

Instead:

### ❌ Bad

```text
secret_recipe=<base64 encoded secret>
```

### ✅ Better

Store sensitive information server-side and give the browser only a random session identifier:

```text
Browser:
session_id=random_value

Server:
session_id → user/session data
```

The application should also:

* Use HTTPS.
* Set `Secure` on sensitive cookies.
* Set `HttpOnly` when JavaScript doesn't need access.
* Use an appropriate `SameSite` policy.
* Never treat Base64 encoding as encryption.
* Never put passwords, secrets, or confidential application data into client-side cookies.
* Cryptographically sign/encrypt cookie data when client-side state is genuinely required.

---

# Attack Summary

```text
                    ┌─────────────────┐
                    │   Web Page      │
                    └────────┬────────┘
                             │
                             ▼
                    "We need cookies"
                             │
                             ▼
                    ┌─────────────────┐
                    │ Browser Cookies │
                    └────────┬────────┘
                             │
                             ▼
                    secret_recipe
                             │
                             ▼
                      URL decoding
                             │
                             ▼
                       Base64 value
                             │
                             ▼
                      Base64 decode
                             │
                             ▼
              picoCTF{...}
```

---

## Final Takeaway

> **Never assume an encoded value is secure.**

In web CTFs, when you encounter a suspicious cookie:

```text
1. Identify the cookie
2. Examine its value
3. Check for URL encoding
4. Check for Base64
5. Check for other recognizable formats
6. Determine whether the value is signed/encrypted
7. Test whether modifying it affects application behavior
```

For this challenge, the entire vulnerability boiled down to:

**Sensitive secret → client-side cookie → Base64 encoding → easily decoded.**
