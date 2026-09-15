# GET aHEAD

**Category:** Web Exploitation  
**Difficulty:** Easy  
**Platform:** picoCTF 2021

## Challenge Description

> Find the flag being held on this server to get ahead of the competition.

**Target:**

```text
http://wily-courier.picoctf.net:56064/
```

---

## 1. Reconnaissance

Opening the webpage showed two forms:

* **Red** — submitted using `GET`
* **Blue** — submitted using `POST`

Inspecting the HTML revealed:

```html
<form action="index.php" method="GET">
    <input type="submit" value="Choose Red"/>
</form>
```

and:

```html
<form action="index.php" method="POST">
    <input type="submit" value="Choose Blue"/>
</form>
```

The challenge name **"GET aHEAD"** suggested that another HTTP method might be important.

---

## 2. Testing the HEAD Method

HTTP supports several request methods, including:

* `GET`
* `POST`
* `HEAD`
* `PUT`
* `DELETE`

A `HEAD` request is similar to `GET`, but the server normally returns only the response headers without the response body.

Using `curl`:

```bash
curl -I http://wily-courier.picoctf.net:56064/index.php
```

The `-I` option sends a `HEAD` request.

---

## 3. Server Response

The server responded with:

```text
HTTP/1.1 200 OK
Date: Mon, 14 Sep 2026 04:09:33 GMT
Server: Apache/2.4.38 (Debian)
X-Powered-By: PHP/7.2.34
flag: picoCTF{r3j3ct_th3_du4l1ty_8b13f07}
Content-Type: text/html; charset=UTF-8
```

The flag was exposed directly in the HTTP response headers.

---

## 4. Flag

```text
picoCTF{r3j3ct_th3_du4l1ty_8b13f07}
```

---

## Key Takeaway

The challenge demonstrates that **HTTP methods can trigger different server-side behavior**.

The webpage only visibly used `GET` and `POST`, but the challenge name hinted at the `HEAD` method.

The important command was:

```bash
curl -I http://wily-courier.picoctf.net:56064/index.php
```

Instead of looking only at the webpage body, inspecting the **HTTP response headers** revealed the flag.

### Lesson

> Always consider the full HTTP request/response, including HTTP methods and response headers. Information may be exposed outside the visible webpage.

```
