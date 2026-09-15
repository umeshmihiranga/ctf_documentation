# 🍪 Cookie Monster Secret Recipe — CTF Writeup

| Field | Details |
| :--- | :--- |
| **Challenge** | Cookie Monster Secret Recipe |
| **Category** | Web Exploitation |
| **Difficulty** | Easy |
| **Platform** | picoCTF 2025 |
| **Vulnerability** | Sensitive information exposed through a client-side cookie |
| **Flag** | `picoCTF{c00k1e_m0nster_l0ves_c00kies_2C8040EF}` |

---

## Challenge Overview

The application requires authentication via cookies rather than credentials. The objective was to inspect client-side cookie data and recover the hidden flag.

---

## Initial Clue

After opening the challenge instance, the web page displayed:

> "No need password. We just need cookies!"

It also provided the hint:

> "Have you checked your cookies lately?"

These clues strongly suggested that the application was storing sensitive data inside a browser cookie.

---

## Enumeration

The first step was to inspect the application rather than immediately trying complex exploits.

### 1. Inspect the Page
The application returned an **Access Denied** page, but its message indicated that authentication depended directly on cookies.

### 2. Open Browser Developer Tools
We navigated to:
```text
Developer Tools → Application → Cookies
```

A cookie named `secret_recipe` was present:
```text
secret_recipe
```

Its value was:
```text
cGljb0NURntjMDBrMWVfbTBuc3Rlcl9sMHZlc19jMDBraWVzXzJDODA0MEVGfQ%3D%3D
```

This cookie became the primary focus of our investigation.

---

## Identifying the Encoding

The end of the cookie contained:
```text
%3D%3D
```

`%3D` is the URL-encoded representation of `=`:
```text
%3D → =
```

After URL decoding, the value became:
```text
cGljb0NURntjMDBrMWVfbTBuc3Rlcl9sMHZlc19jMDBraWVzXzJDODA0MEVGfQ==
```

The resulting string displayed the classic characteristics of **Base64** encoding:
* Alphanumeric characters (`A-Z`, `a-z`, `0-9`)
* Base64 symbols (`+`, `/`)
* Trailing `=` padding characters

This indicated that the cookie wasn't encrypted — it was simply encoded.

---

## Exploit

We decoded the Base64 value using Kali Linux:

```bash
echo 'cGljb0NURntjMDBrMWVfbTBuc3Rlcl9sMHZlc19jMDBraWVzXzJDODA0MEVGfQ==' | base64 -d
```

The decoded output revealed the flag:
```text
picoCTF{c00k1e_m0nster_l0ves_c00kies_2C8040EF}
```

### Flag
```text
picoCTF{c00k1e_m0nster_l0ves_c00kies_2C8040EF}
```

---

## Tools Used

### Browser Developer Tools
Used to inspect cookies under:
```text
Application
└── Cookies
    └── secret_recipe
```

### Base64 Decoder
Used to decode the payload:
```bash
base64 -d
```

### URL Decoding
The `%3D%3D` URL encoding was decoded prior to Base64 decoding.

---

## What I Learned

### 1. Always inspect cookies during web enumeration
Cookies can contain:
* Session identifiers
* User roles and permissions
* Authentication tokens
* Application state
* Sensitive or confidential information

When a challenge hints at cookies, inspecting the browser storage should always be one of the first steps.

### 2. Encoding is NOT encryption
This was the most critical takeaway:

```text
Secret ──(Base64 Encode)──> Encoded Value ──(Base64 Decode)──> Secret
```

Base64 provides **zero confidentiality**. Anyone with access to the encoded string can decode it instantly.

### 3. Client-side data should never be trusted
The browser has direct access to cookies, meaning the user can inspect, tamper with, or forge them. Anything stored client-side must be treated as untrusted and visible.

### 4. Follow the clues before jumping to complex exploits
We did not need:
* SQL Injection
* Server-Side Template Injection (SSTI)
* Cross-Site Scripting (XSS)
* Brute Force or Fuzzing
* Password Cracking

The challenge explicitly pointed to cookies:
```text
"We just need cookies" → Cookies Tab → secret_recipe → Decode Base64 → Flag
```

> [!TIP]
> **CTF Habit:** Always start with basic enumeration and follow obvious clues before assuming a challenge requires complex exploitation.

---

## Why the Vulnerability Existed

The application stored sensitive data directly inside a client-side cookie:
```text
secret_recipe=<encoded-secret>
```

Even though the value appeared obscured, it was merely Base64-encoded. 

The security failure was not that Base64 is broken — the failure was **storing sensitive secrets in the browser in the first place**.

---

## How to Prevent It

Sensitive information must never be stored directly in client-controlled cookies.

### ❌ Insecure Practice
```text
secret_recipe=<base64 encoded secret>
```

### ✅ Secure Practice
Store sensitive data server-side and issue only an unpredictable, cryptographically random session identifier to the browser:
```text
Browser:  session_id=<random_opaque_token>
Server:   session_id ──maps to──> authenticated user data
```

**Additional Best Practices:**
* Enforce **HTTPS** across all endpoints.
* Set the `Secure` flag on all sensitive cookies so they are only transmitted over TLS.
* Set the `HttpOnly` flag to prevent client-side JavaScript from accessing cookies (mitigating XSS theft).
* Use appropriate `SameSite=Lax` or `SameSite=Strict` attributes to mitigate CSRF attacks.
* Never treat Base64 encoding as encryption.
* Never store secrets, passwords, or internal application state in cookies.
* If client-side state is required, use tamper-proof cryptographic signing (e.g., HMAC) or authenticated encryption.

---

## Attack Summary

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
                       URL decoding (%3D -> =)
                             │
                             ▼
                        Base64 string
                             │
                             ▼
                        base64 -d
                             │
                             ▼
               picoCTF{...}
```

---

## Final Takeaway

> [!IMPORTANT]
> **Never assume an encoded value is secure.**

In web security testing and CTFs, when encountering a suspicious cookie:
1. **Identify** the cookie name and scope.
2. **Examine** its value for standard structures.
3. **Check** for URL encoding (`%xx`).
4. **Check** for Base64 or Hex encoding.
5. **Check** for other recognized formats (JWT, serialized objects).
6. **Determine** whether the value is signed or encrypted.
7. **Test** whether modifying the value alters application behavior.
