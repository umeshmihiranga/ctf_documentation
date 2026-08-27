## Challenge:

Crack the Gate 1

## Category:

Web Exploitation — Easy

## Vulnerability:

Authentication bypass through a developer/debug HTTP header

## Initial Clue:

The challenge gave us the victim's email:
`ctf-player@picoctf.org`

It also said that normal password guessing wasn't working and hinted that **the developer had left a secret way in**.

## Enumeration:

1. Opened the login page.
2. Inspected the HTML source.
3. Found a developer comment containing an encoded message.
4. Recognized the message as **ROT13**.
5. Decoded it and discovered instructions to use:
   ```http
   X-Dev-Access: yes
   ```

This indicated that the application had a special developer access mechanism.

## Exploit:

Added the custom HTTP header to the login request:
```http
X-Dev-Access: yes
```

Using `curl`, the request could be constructed with:
```bash
curl -v -H "X-Dev-Access: yes" ...
```

The server trusted this client-controlled header and allowed the developer bypass instead of requiring the normal password.

## Tools:

* Browser
* Developer Tools
* View Source
* `curl`
* HTTP headers
* ROT13 decoding

## What I Learned:

* How to inspect HTML source for leaked developer information.
* How ROT13 can hide simple clues.
* What HTTP request headers are.
* How `curl -H` adds a custom HTTP header.
* How `curl -v` shows the HTTP request and response.
* How authentication can be bypassed when the server blindly trusts a client-supplied header.
* Why understanding HTTP is extremely important for web exploitation.

## Why the Vulnerability Existed:

The application contained a developer backdoor that trusted the value of a custom HTTP header supplied by the client.

The server effectively treated:
```http
X-Dev-Access: yes
```
as proof that the requester should receive privileged access.

The developer also accidentally exposed the bypass instructions in the page source.

## How to Prevent It:

* Never implement authentication bypasses based solely on client-controlled headers.
* Remove debugging/backdoor functionality from production applications.
* Never leave secrets or internal instructions in HTML/JavaScript comments.
* Perform proper server-side authentication and authorization.
* Treat every client-supplied header as untrusted input.
* Review production code and configuration for development/test access mechanisms.

---

# Curl Notes — Web CTF Reference

## What is curl?

`curl` is a command-line tool for communicating with web servers and APIs.

Instead of using a browser, I can use `curl` to manually construct HTTP requests and inspect the server's responses.

**Basic syntax:**
```bash
curl [options] URL
```

For web CTFs, `curl` is useful because it lets me control and inspect:
* HTTP methods
* Headers
* Request bodies
* Cookies
* Authentication
* Redirects
* Server responses

## 1. `-v` — Verbose

```bash
curl -v http://target.com
```

`-v` means **verbose**. It shows details about the HTTP communication.

The important symbols are:
* `>` = request sent by me
* `<` = response sent by the server

**Example:**
```text
> GET /login HTTP/1.1
> Host: target.com
> User-Agent: curl/...
>
< HTTP/1.1 200 OK
< Content-Type: text/html
```

This allows me to see what `curl` actually sends and what the server returns.

**CTF use:**
* Inspect HTTP requests
* See response status codes
* Find redirects
* See cookies
* See headers
* Debug requests

## 2. `-i` — Include response headers

```bash
curl -i http://target.com
```

Normally `curl` mainly displays the response body. With `-i`, it also displays the response headers.

**Example:**
```text
HTTP/1.1 200 OK
Content-Type: text/html
Set-Cookie: session=abc123
```

**Useful for discovering:**
* Cookies
* Status codes
* Server information
* Security headers
* Redirect information

## 3. `-I` — HEAD request

```bash
curl -I http://target.com
```

`-I` requests only the HTTP headers instead of the normal page body. Useful when I quickly want to inspect how a server responds.

**Difference:**
* `-i` = headers + body
* `-I` = headers only

## 4. `-H` — Add an HTTP header

```bash
curl -H "X-Test: hello" http://target.com
```

`-H` allows me to add a custom HTTP request header.

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

I can send multiple headers:
```bash
curl \
  -H "Content-Type: application/json" \
  -H "X-Dev-Access: yes" \
  http://target.com
```

> [!IMPORTANT]
> Custom headers are controlled by the client, so a secure application should never blindly trust them for authentication or authorization.

## 5. `-X` — Specify the HTTP method

```bash
curl -X GET http://target.com
curl -X POST http://target.com
curl -X PUT http://target.com
curl -X DELETE http://target.com
```

It allows me to explicitly select the HTTP method.

**Common methods:**
```text
GET     → retrieve something
POST    → submit/create something
PUT     → replace/update something
PATCH   → partially update something
DELETE  → delete something
```

For example:
```bash
curl -X POST http://target.com/login
```

> [!NOTE]
> `-d` normally causes curl to use POST automatically, so `-X POST` isn't always necessary.

## 6. `-d` — Send request data

```bash
curl -d "username=admin&password=test" http://target.com/login
```

`-d` means **data**. It sends data in the request body.

For JSON APIs:
```bash
curl \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test"}' \
  http://target.com/login
```

This is extremely useful for testing login forms and APIs.

## 7. `-b` — Send cookies

```bash
curl -b "session=abc123" http://target.com/dashboard
```

This sends:
```http
Cookie: session=abc123
```

Cookies are commonly used for:
* Sessions
* Authentication
* User preferences
* Tracking

In CTFs, understanding cookies is very important because session manipulation is a common attack area.

## 8. `-c` — Save cookies

```bash
curl -c cookies.txt http://target.com/login
```

This saves cookies received from the server into `cookies.txt`.

I can then reuse them:
```bash
curl -b cookies.txt http://target.com/dashboard
```

**Useful workflow:**
```text
Login
  ↓
Receive session cookie
  ↓
Save cookie
  ↓
Reuse cookie
  ↓
Access authenticated page
```

## 9. `-L` — Follow redirects

Suppose the server responds:
```http
HTTP/1.1 302 Found
Location: /login
```
The server is telling the client to go somewhere else.

Normally, `curl http://target.com/admin` may stop at the redirect.

With `curl -L http://target.com/admin`, `curl` follows the redirect.

> [!IMPORTANT]
> **CTF Habit:** First try `curl -v http://target.com/admin` and look at the `Location` header before automatically using `-L`. The redirect itself may reveal useful information.

## 10. `-u` — HTTP Basic Authentication

```bash
curl -u admin:password http://target.com
```

This is specifically for **HTTP Basic Authentication**.

`curl` generates an HTTP header similar to:
```http
Authorization: Basic YWRtaW46cGFzc3dvcmQ=
```

The credentials are essentially `admin:password` encoded using Base64.

> [!NOTE]
> Base64 is **encoding, not encryption**.

I can see the authentication header using:
```bash
curl -v -u admin:password http://target.com
```

`-u` should not be confused with a normal website login form.

## 11. `-o` — Save response to a file

```bash
curl -o page.html http://target.com
```

This saves the response as `page.html`. I can then inspect it:
```bash
cat page.html
```
or search it:
```bash
grep -i "flag" page.html
```

## 12. `-s` — Silent mode

```bash
curl -s http://target.com
```

Removes unnecessary progress/output information. This is useful when piping `curl` into other Linux commands:
```bash
curl -s http://target.com | grep -i flag
```

## Most Important Curl Flags for Web CTFs:

| Flag | Meaning | Importance |
| ---- | -------------------------------- | ---------- |
| `-v` | Show detailed HTTP communication | ⭐⭐⭐⭐⭐ |
| `-H` | Add/modify request header | ⭐⭐⭐⭐⭐ |
| `-d` | Send request body/data | ⭐⭐⭐⭐⭐ |
| `-i` | Show response headers + body | ⭐⭐⭐⭐⭐ |
| `-b` | Send cookies | ⭐⭐⭐⭐⭐ |
| `-L` | Follow redirects | ⭐⭐⭐⭐⭐ |
| `-X` | Choose HTTP method | ⭐⭐⭐⭐ |
| `-c` | Save cookies | ⭐⭐⭐⭐ |
| `-u` | HTTP Basic Auth | ⭐⭐⭐ |
| `-I` | Headers only | ⭐⭐⭐ |
| `-s` | Silent mode | ⭐⭐⭐ |
| `-o` | Save output | ⭐⭐⭐ |

## The Most Useful CTF Combinations:

### See exactly what is happening
```bash
curl -v http://target.com
```

### Inspect headers
```bash
curl -i http://target.com
```

### Send a custom header
```bash
curl -v -H "X-Test: hello" http://target.com
```

### Send JSON
```bash
curl -v \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test"}' \
  http://target.com/login
```

### Use cookies
```bash
curl -c cookies.txt http://target.com/login
curl -b cookies.txt http://target.com/dashboard
```

### Follow redirects
```bash
curl -v -L http://target.com
```

### Basic authentication
```bash
curl -v -u admin:password http://target.com
```

## Crack the Gate 1 Example:

The challenge leaked:
```text
X-Dev-Access: yes
```

I can construct the HTTP request using `curl`:
```bash
curl -v \
  -H "X-Dev-Access: yes" \
  ...
```

The important concept isn't memorizing this particular command. It's understanding:

```text
curl
 ↓
-H
 ↓
HTTP request header
 ↓
X-Dev-Access: yes
 ↓
Server processes the header
 ↓
Authentication bypass
```

## My Curl Learning Rule:

When practicing web CTFs, I should ask:
1. **What HTTP request is the browser making?**
2. **What method is it using?**
3. **What headers does it send?**
4. **What cookies does it send?**
5. **What data is in the request body?**
6. **What does the server return?**

Then I reproduce that request with `curl`.

The goal isn't to memorize `curl` commands. The goal is to become comfortable saying:

> **"I know what HTTP request I want, and I know how to construct it with curl."**

That skill transfers directly to web exploitation, API testing, Burp Suite, authentication testing, and CTFs.
