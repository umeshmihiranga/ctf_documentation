# Crack the Gate 1 — CTF Writeup

| Field | Details |
| :--- | :--- |
| **Challenge** | Crack the Gate 1 |
| **Category** | Web Exploitation |
| **Difficulty** | Easy |
| **Vulnerability** | Authentication bypass through a developer/debug HTTP header |

---

## Initial Clue

The challenge gave us the victim's email:
```text
ctf-player@picoctf.org
```

It also stated that normal password guessing wasn't working and hinted that **the developer had left a secret way in**.

---

## Enumeration

1. Opened the login page.
2. Inspected the HTML source.
3. Found a developer comment containing an encoded message.
4. Recognized the message as **ROT13**.
5. Decoded it and discovered instructions to use:
   ```http
   X-Dev-Access: yes
   ```

This indicated that the application had a special developer access mechanism.

---

## Exploit

Added the custom HTTP header to the login request:
```http
X-Dev-Access: yes
```

Using `curl`, the request could be constructed with:
```bash
curl -v -H "X-Dev-Access: yes" ...
```

The server trusted this client-controlled header and allowed the developer bypass instead of requiring the normal password.

---

## Tools

* **Browser** & **Developer Tools** (View Source / Network)
* `curl` (HTTP request manipulation)
* HTTP request headers
* ROT13 decoding

---

## What I Learned

* How to inspect HTML source for leaked developer information.
* How ROT13 can hide simple clues.
* What HTTP request headers are and how they are structured.
* How `curl -H` adds a custom HTTP header.
* How `curl -v` shows full HTTP request and response flow.
* How authentication can be bypassed when the server blindly trusts a client-supplied header.
* Why understanding HTTP is extremely important for web exploitation.

---

## Why the Vulnerability Existed

The application contained a developer backdoor that trusted the value of a custom HTTP header supplied by the client.

The server effectively treated:
```http
X-Dev-Access: yes
```
as proof that the requester should receive privileged access.

The developer also accidentally exposed the bypass instructions in the page source comments.

---

## How to Prevent It

* **Never implement authentication bypasses based solely on client-controlled headers.**
* **Remove debugging/backdoor functionality** completely from production applications.
* **Never leave secrets or internal instructions** in HTML or JavaScript comments.
* **Perform proper server-side authentication and authorization.**
* **Treat every client-supplied header as untrusted input.**
* **Review production code and configuration** for lingering development or test access mechanisms.

---
---

# 🌐 Curl Notes — Web CTF Reference

## What is curl?

`curl` is a command-line tool for communicating with web servers and APIs.

Instead of using a browser, `curl` allows you to manually construct HTTP requests and inspect the server's exact responses.

**Basic syntax:**
```bash
curl [options] URL
```

For web CTFs, `curl` is useful because it lets you control and inspect:
* HTTP methods
* Headers
* Request bodies
* Cookies
* Authentication
* Redirects
* Server responses

---

## 1. `-v` — Verbose

```bash
curl -v http://target.com
```

`-v` means **verbose**. It shows complete details about the underlying HTTP communication.

The important symbols are:
* `>` = Request sent by client
* `<` = Response sent by the server

**Example output:**
```text
> GET /login HTTP/1.1
> Host: target.com
> User-Agent: curl/...
>
< HTTP/1.1 200 OK
< Content-Type: text/html
```

This allows you to see what `curl` actually sends and what the server returns.

**CTF use cases:**
* Inspect raw HTTP requests
* See response status codes
* Find redirects
* Inspect cookies and headers
* Debug custom requests

---

## 2. `-i` — Include Response Headers

```bash
curl -i http://target.com
```

Normally `curl` displays only the response body. With `-i`, it includes the response headers before the body.

**Example output:**
```http
HTTP/1.1 200 OK
Content-Type: text/html
Set-Cookie: session=abc123
```

**Useful for discovering:**
* Cookies (`Set-Cookie`)
* Status codes
* Server information (`Server`, `X-Powered-By`)
* Security headers
* Redirect information (`Location`)

---

## 3. `-I` — HEAD Request (Headers Only)

```bash
curl -I http://target.com
```

`-I` requests only the HTTP headers instead of downloading the page body. This is useful when quickly inspecting server headers without dumping large amounts of HTML.

**Difference:**
* `-i` = headers + body
* `-I` = headers only (`HEAD` request)

---

## 4. `-H` — Add an HTTP Header

```bash
curl -H "X-Test: hello" http://target.com
```

`-H` allows you to add a custom HTTP request header.

**Example:**
```bash
curl -H "X-Dev-Access: yes" http://target.com
```

The server receives:
```http
X-Dev-Access: yes
```

Headers can contain information such as:
* Authorization
* Cookies
* Content type
* User agent
* Custom application data

You can send multiple headers in a single command:
```bash
curl \
  -H "Content-Type: application/json" \
  -H "X-Dev-Access: yes" \
  http://target.com
```

> [!IMPORTANT]
> Custom headers are controlled by the client, so a secure application should never blindly trust them for authentication or authorization.

---

## 5. `-X` — Specify the HTTP Method

```bash
curl -X GET http://target.com
curl -X POST http://target.com
curl -X PUT http://target.com
curl -X DELETE http://target.com
```

Allows you to explicitly select the HTTP request method.

**Common methods:**
| Method | Description |
| :--- | :--- |
| `GET` | Retrieve a resource |
| `POST` | Submit or create data |
| `PUT` | Replace or update a resource |
| `PATCH` | Partially update a resource |
| `DELETE` | Remove a resource |

For example:
```bash
curl -X POST http://target.com/login
```

> [!NOTE]
> Supplying `-d` causes `curl` to use `POST` automatically, so `-X POST` isn't always strictly necessary.

---

## 6. `-d` — Send Request Data

```bash
curl -d "username=admin&password=test" http://target.com/login
```

`-d` stands for **data**. It sends data in the HTTP request body with a default `application/x-www-form-urlencoded` header.

For JSON APIs:
```bash
curl \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test"}' \
  http://target.com/login
```

This is essential for testing login forms, API endpoints, and injection vectors.

---

## 7. `-b` — Send Cookies

```bash
curl -b "session=abc123" http://target.com/dashboard
```

This sends an HTTP `Cookie` header:
```http
Cookie: session=abc123
```

Cookies are commonly used for:
* Sessions
* Authentication
* User preferences
* Tracking

In CTFs, understanding cookies is critical because session manipulation and token forging are frequent attack areas.

---

## 8. `-c` — Save Cookies (Cookie Jar)

```bash
curl -c cookies.txt http://target.com/login
```

This saves cookies received from the server into `cookies.txt`.

You can then reuse them in subsequent requests:
```bash
curl -b cookies.txt http://target.com/dashboard
```

**Standard session workflow:**
```text
Login
  ↓
Receive session cookie (saved via -c)
  ↓
Reuse cookie (via -b)
  ↓
Access authenticated page
```

---

## 9. `-L` — Follow Redirects

Suppose the server responds:
```http
HTTP/1.1 302 Found
Location: /login
```
The server instructs the client to navigate to another location.

Normally, `curl http://target.com/admin` stops at the redirect response. With `-L`, `curl` automatically follows the `Location` header to the final page:
```bash
curl -L http://target.com/admin
```

> [!IMPORTANT]
> **CTF Habit:** First run `curl -v http://target.com/admin` and inspect the `Location` header before automatically using `-L`. The intermediate redirect response itself may leak sensitive information or flags.

---

## 10. `-u` — HTTP Basic Authentication

```bash
curl -u admin:password http://target.com
```

This is specifically for **HTTP Basic Authentication**.

`curl` automatically generates an HTTP `Authorization` header:
```http
Authorization: Basic YWRtaW46cGFzc3dvcmQ=
```

The credentials are `admin:password` encoded using Base64.

> [!NOTE]
> Base64 is **encoding, not encryption**. Anyone intercepting the header can decode the credentials.

You can observe the generated header via:
```bash
curl -v -u admin:password http://target.com
```

*(Note: `-u` is for HTTP Basic Auth challenges, not HTML login forms).*

---

## 11. `-o` — Save Response to a File

```bash
curl -o page.html http://target.com
```

Saves the response body directly to `page.html`. You can then inspect or search it:
```bash
cat page.html
grep -i "flag" page.html
```

---

## 12. `-s` — Silent Mode

```bash
curl -s http://target.com
```

Silences progress meters and error messages. This is especially useful when piping output into other command-line utilities:
```bash
curl -s http://target.com | grep -i flag
```

---

## Most Important Curl Flags for Web CTFs

| Flag | Meaning | Importance |
| :--- | :--- | :--- |
| `-v` | Show detailed HTTP communication | ⭐⭐⭐⭐⭐ |
| `-H` | Add or modify request header | ⭐⭐⭐⭐⭐ |
| `-d` | Send request body / data | ⭐⭐⭐⭐⭐ |
| `-i` | Show response headers + body | ⭐⭐⭐⭐⭐ |
| `-b` | Send cookies | ⭐⭐⭐⭐⭐ |
| `-L` | Follow redirects | ⭐⭐⭐⭐⭐ |
| `-X` | Specify HTTP method | ⭐⭐⭐⭐ |
| `-c` | Save cookies to file | ⭐⭐⭐⭐ |
| `-u` | HTTP Basic Auth | ⭐⭐⭐ |
| `-I` | Headers only (HEAD) | ⭐⭐⭐ |
| `-s` | Silent mode | ⭐⭐⭐ |
| `-o` | Save output to file | ⭐⭐⭐ |

---

## The Most Useful CTF Combinations

### 1. See exactly what is happening
```bash
curl -v http://target.com
```

### 2. Inspect response headers
```bash
curl -i http://target.com
```

### 3. Send a custom header
```bash
curl -v -H "X-Test: hello" http://target.com
```

### 4. Send JSON body
```bash
curl -v \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test"}' \
  http://target.com/login
```

### 5. Capture and replay cookies
```bash
curl -c cookies.txt http://target.com/login
curl -b cookies.txt http://target.com/dashboard
```

### 6. Follow redirects with verbose output
```bash
curl -v -L http://target.com
```

### 7. Basic authentication
```bash
curl -v -u admin:password http://target.com
```

---

## Crack the Gate 1 Example

The challenge leaked the following header requirement:
```text
X-Dev-Access: yes
```

We constructed the request using `curl`:
```bash
curl -v \
  -H "X-Dev-Access: yes" \
  ...
```

The underlying concept:
```text
curl
 ↓
-H
 ↓
HTTP request header
 ↓
X-Dev-Access: yes
 ↓
Server processes header
 ↓
Authentication bypass
```

---

## My Curl Learning Rule

When practicing web CTFs, always ask:
1. **What HTTP request is the browser making?**
2. **What method is it using?**
3. **What headers does it send?**
4. **What cookies does it send?**
5. **What data is in the request body?**
6. **What does the server return?**

Then reproduce that request with `curl`.

The goal isn't to memorize individual flags, but to become comfortable saying:

> **"I know what HTTP request I want, and I know how to construct it with curl."**

That skill transfers directly to web exploitation, API testing, Burp Suite, authentication testing, and CTFs.
