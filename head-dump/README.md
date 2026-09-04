# Head Dump — Full CTF Write-Up

| | |
|---|---|
| **Platform** | picoCTF |
| **Category** | Web Exploitation |
| **Difficulty** | Easy |
| **Vulnerability** | Unauthenticated Node.js heap dump exposure |

---

## Challenge

The application exposed a `/heapdump` endpoint that allowed anyone to request a Node.js/V8 heap snapshot.

Because a heap snapshot contains objects and strings currently held in the server's memory, sensitive information such as the flag could be recovered from it.

---

## Initial Clue

The challenge description indicated that the application was a blog and mentioned:

- API Documentation
- An endpoint that generates files containing the server's memory
- A secret flag hidden somewhere in that memory

This suggested that we should:

1. Enumerate the web application.
2. Find the API documentation.
3. Inspect the documented API endpoints.
4. Find the endpoint responsible for generating a memory dump.
5. Download and analyze the dump.

---

## Enumeration

### 1. Initial Web Enumeration

The target was:

```text
http://verbal-sleep.picoctf.net:<PORT>/
```

I first used Gobuster to discover directories and files.

```bash
gobuster dir -u http://verbal-sleep.picoctf.net:<PORT>/ \
-w /usr/share/dirb/wordlists/common.txt
```

#### Results

```text
About       (Status: 200)
about       (Status: 200)
Services    (Status: 200)
services    (Status: 200)
img         (Status: 301)
```

The generic wordlist did not discover `/api-docs`.

This demonstrated an important enumeration lesson:

> A tool not finding an endpoint does not prove that the endpoint does not exist.

The endpoint may simply not be present in the wordlist.

---

### 2. API-Specific Enumeration

I then used an API-focused SecLists wordlist:

```bash
gobuster dir \
-u http://verbal-sleep.picoctf.net:<PORT>/ \
-w /usr/share/seclists/Discovery/Web-Content/common-api-endpoints-mazen160.txt
```

This only found:

```text
about (Status: 200)
```

Again, `/api-docs` was not discovered.

Instead of assuming it didn't exist, I inspected the application's HTML.

---

### 3. Inspecting HTML Source

I used:

```bash
curl -s http://verbal-sleep.picoctf.net:<PORT>/ | \
grep -Eoi 'href="[^"]+"|src="[^"]+"'
```

This extracted links and referenced resources from the HTML.

Among the results was:

```text
href="/api-docs"
```

This was our API documentation endpoint.

#### Why this worked

The homepage itself contained a link to the API documentation, even though our Gobuster wordlists hadn't discovered it.

This demonstrates why web enumeration should combine:

* Directory/file brute-forcing
* HTML/source inspection
* Browser inspection
* JavaScript analysis
* Network traffic analysis

---

## API Documentation

I opened:

```text
http://verbal-sleep.picoctf.net:<PORT>/api-docs
```

The application presented a **Swagger UI** interface.

Swagger documented the available API endpoints.

Among the endpoints was:

```text
GET /heapdump
```

The endpoint description indicated that it was related to diagnosing memory allocation.

This immediately matched the challenge's clue about an endpoint that generates files containing the server's memory.

---

## Exploitation

### 1. Accessing the Heap Dump

The `/heapdump` endpoint required no parameters.

I executed the request through Swagger UI.

The response was:

```text
HTTP 200
Content-Type: application/octet-stream
Content-Disposition: attachment; filename="heapdump-....heapsnapshot"
```

This confirmed that the server was providing a downloadable heap snapshot.

The application was also identified as an Express/Node.js application through the HTTP response header:

```text
X-Powered-By: Express
```

---

### 2. Downloading the Heap Snapshot

The heap dump could also be retrieved directly using curl:

```bash
curl -o heapdump.heapsnapshot \
http://verbal-sleep.picoctf.net:<PORT>/heapdump
```

The `-o` option tells curl to save the response to a file.

I then checked the downloaded file:

```bash
ls -lh heapdump.heapsnapshot
```

and:

```bash
file heapdump.heapsnapshot
```

---

## Analyzing the Heap Dump

The heap snapshot was very large and contained a huge amount of Node.js/V8 runtime data.

Dumping the entire file with `cat` or using a broad search produced enormous amounts of output.

Instead, I searched specifically for the picoCTF flag format.

```bash
grep -aoE 'picoCTF\{[^}]+\}' heapdump.heapsnapshot
```

This returned the flag.

---

## Understanding the Flag Search

The regex:

```text
picoCTF\{[^}]+\}
```

matches strings following the general format:

```text
picoCTF{something}
```

Breaking it down:

```text
picoCTF
```

Matches the literal text `picoCTF`.

```text
\{
```

Matches the opening `{`.

```text
[^}]
```

Means any character except `}`.

```text
+
```

Means one or more of the preceding characters.

```text
\}
```

Matches the closing `}`.

Therefore:

```text
picoCTF\{[^}]+\}
```

means:

> Find a string beginning with `picoCTF{`, containing one or more characters that are not `}`, and ending with `}`.

---

## Why `-o` Was Important

A broader command such as:

```bash
strings heapdump.heapsnapshot | grep -i 'picoCTF'
```

returned several matches.

However, searching for generic words such as:

```bash
strings heapdump.heapsnapshot | grep -i 'flag'
```

produced a huge amount of unrelated Node.js/V8 runtime data.

The successful command was:

```bash
grep -aoE 'picoCTF\{[^}]+\}' heapdump.heapsnapshot
```

The important option was:

```text
-o
```

which means:

> Output only the matching portion.

Without `-o`, grep can print the entire line containing the match. Since a heap snapshot contains very large structured lines, that can result in enormous output.

---

## Tools Used

### Gobuster

Used for directory and endpoint enumeration.

```bash
gobuster dir -u http://TARGET/ \
-w /usr/share/dirb/wordlists/common.txt
```

### SecLists

Used an API-specific wordlist:

```text
/usr/share/seclists/Discovery/Web-Content/common-api-endpoints-mazen160.txt
```

### curl

Used to manually interact with the web application and download the heap dump.

```bash
curl -s URL
```

```bash
curl -i URL
```

```bash
curl -o filename URL
```

### grep

Used to search and extract interesting information from the application's HTML and heap snapshot.

### Swagger UI

Used to inspect the documented API endpoints and identify:

```text
GET /heapdump
```

---

## What I Learned

### 1. Enumeration is more than directory brute-forcing

Gobuster is useful, but it depends heavily on the wordlist.

Not finding:

```text
/api-docs
```

doesn't mean the endpoint doesn't exist.

The endpoint was actually exposed through the application's HTML source.

---

### 2. Always inspect the application's source

A website may contain links to:

* API documentation
* hidden functionality
* JavaScript files
* administrative pages
* API endpoints
* configuration files

A simple command such as:

```bash
curl -s URL | grep -iE 'api|swagger|docs'
```

can reveal useful information.

---

### 3. Swagger/OpenAPI documentation is valuable during enumeration

Swagger UI provides a map of an application's API.

Instead of guessing:

```text
/api
/api/v1
/api/debug
/api/dump
...
```

we can inspect the documented endpoints directly.

---

### 4. Learn what commands do instead of memorizing huge commands

For example:

```bash
curl -s URL
```

means:

```text
curl → make HTTP request
-s   → silent output
```

And:

```bash
grep -aoE 'PATTERN' file
```

means:

```text
grep → search
-a   → treat as text
-o   → output only matches
-E   → extended regex
```

The important skill is understanding the building blocks so commands can be constructed when needed.

---

### 5. Zsh has an important `PATH` behavior

While doing enumeration, I initially used:

```bash
for path in ...
```

In Zsh, `path` is a special variable associated with `PATH`.

Using `path` as a loop variable changed the shell's PATH and caused commands such as `curl` and `nano` to stop being found.

The safer approach is to use variables such as:

```bash
for endpoint in ...
```

or:

```bash
for route in ...
```

This was an important Linux/Zsh lesson.

---

## Why the Vulnerability Existed

The application exposed a diagnostic heap-dump endpoint:

```text
GET /heapdump
```

without requiring authentication or authorization.

Heap dumps are normally debugging/diagnostic artifacts intended for developers or administrators.

A heap snapshot can contain data that exists in the application's memory, including:

* Strings
* Objects
* Variables
* Application state
* Potential credentials
* Tokens
* Secrets
* Other sensitive information

Therefore, exposing the endpoint publicly created an information-disclosure vulnerability.

In this challenge, the flag was intentionally placed somewhere in the application's memory so that retrieving the heap dump would expose it.

---

## How to Prevent It

### 1. Do not expose diagnostic endpoints publicly

Endpoints such as:

```text
/heapdump
/debug
/metrics
/profiler
```

should not normally be accessible to unauthenticated internet users.

---

### 2. Require authentication and authorization

If a heap-dump endpoint is necessary, restrict it to authorized administrators or developers.

For example:

```text
User
 ↓
Authentication
 ↓
Authorization
 ↓
/heapdump
```

rather than:

```text
Anyone
 ↓
/heapdump
 ↓
Memory dump
```

---

### 3. Disable debugging functionality in production

Debugging and diagnostic features should be disabled when they are not required.

Development functionality should not accidentally be deployed to a production environment.

---

### 4. Protect sensitive data in application memory

Applications should avoid keeping sensitive information in memory longer than necessary.

Secrets should also be handled carefully because memory snapshots can expose application state.

---

### 5. Perform security testing before deployment

Endpoints should be reviewed to determine whether they expose:

* Internal information
* Debug functionality
* Stack traces
* Configuration
* Memory
* Credentials
* Tokens
* Internal APIs

---

## Attack Chain

The complete attack chain was:

```text
Public Web Application
        ↓
HTML Source Enumeration
        ↓
/api-docs
        ↓
Swagger UI
        ↓
GET /heapdump
        ↓
Unauthenticated Heap Snapshot
        ↓
Download Server Memory
        ↓
Search Heap Snapshot
        ↓
picoCTF{...}
```

---

## Key Takeaway

The main lesson from this challenge was not simply:

> "Use `/heapdump`."

The more important lesson was understanding the **enumeration process**:

```text
Enumerate
   ↓
Inspect
   ↓
Identify technologies
   ↓
Find documentation
   ↓
Understand endpoints
   ↓
Identify dangerous functionality
   ↓
Retrieve data
   ↓
Analyze it intelligently
```

The vulnerability was an **information disclosure caused by exposing a Node.js heap snapshot endpoint without proper access control**.