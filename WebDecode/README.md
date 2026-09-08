# WebDecode — CTF Writeup

| Field | Details |
| :--- | :--- |
| **Challenge** | WebDecode |
| **Category** | Web Exploitation |
| **Difficulty** | Easy |
| **Platform** | picoCTF 2024 |
| **Target** | `http://titan.picoctf.net:58403/` |
| **Vulnerability / Technique** | Information disclosure through HTML source |
| **Encoding** | Base64 |

---

## Initial Clue

The challenge states:

> **“Do you know how to use the web inspector?”**

This strongly suggests that the flag or a key clue is embedded somewhere within the webpage's HTML structure rather than requiring complex exploitation.

---

## Enumeration

### 1. Open the Target
Navigate to the challenge URL:
```text
http://titan.picoctf.net:58403/
```

### 2. Open Developer Tools
Press:
```text
F12
```
or:
```text
Ctrl + Shift + I
```
Then select the **Elements / Inspector** tab.

### 3. Inspect the HTML Source
Searching the page source for terms such as `picoCTF` or `flag` led directly to:

```html
<section class="about" notify_true="cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMjgzZTYyZmV9">
    <h1>
        Try inspecting the page!! You might find it there
    </h1>
</section>
```

The critical discovery was the custom HTML attribute on the `<section>` element:
```text
notify_true="cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMjgzZTYyZmV9"
```

---

## Identifying the Encoding

The extracted string:
```text
cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMjgzZTYyZmV9
```

exhibits the standard characteristics of **Base64** encoding:
* Alphanumeric characters (`A-Z`, `a-z`, `0-9`)
* Base64 character set
* Decodes into human-readable ASCII text

---

## Exploit / Decode

We decoded the value using Kali Linux:

```bash
echo 'cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMjgzZTYyZmV9' | base64 -d
```

The output revealed the flag:
```text
picoCTF{web_successfully_dec0ded_283e62fe}
```

### Flag
```text
picoCTF{web_successfully_dec0ded_283e62fe}
```

---

## Tools Used

* **Firefox / Chromium Developer Tools** (Elements Inspector)
* **Kali Linux**
* `echo`
* `base64`

---

## What I Learned

### 1. Always inspect the HTML
The visual presentation of a webpage does not represent everything delivered to the client. HTML source can contain:
* Developer comments (`<!-- ... -->`)
* Hidden DOM elements (`display: none`, `visibility: hidden`)
* Custom HTML attributes (`data-*`, `notify_true`, etc.)
* Metadata tags
* Encoded values and secrets
* Inline JavaScript variables and configuration objects
* Hidden endpoints and routing links

For example:
```html
<div notify_true="...">
```
is completely invisible in rendered browser output, but easily viewed through DevTools.

### 2. Recognize common encodings
When encountering an obscure string, do not assume encryption immediately. Common encodings to test include:
* **Base64** (often ends with `=` or `==`, alphanumeric with `+` and `/`)
* **URL encoding** (contains `%xx`)
* **Hexadecimal** (0-9, a-f)
* **Binary** (0s and 1s)
* **ASCII decimal / octal**
* **ROT13 / Caesar cipher**

### 3. Developer Tools are essential for web security
The browser inspector is a primary tool for both CTFs and real-world penetration testing:
* **Elements:** Inspect and modify DOM structure & HTML attributes
* **Console:** Run JavaScript and inspect client-side variables
* **Sources / Debugger:** Trace client-side scripts
* **Network:** Inspect headers, parameters, and status codes
* **Application / Storage:** Review cookies, LocalStorage, and SessionStorage

---

## Why the Vulnerability Existed

The application exposed the flag directly to the client inside an HTML attribute:
```html
notify_true="BASE64_VALUE"
```

Any data delivered in an HTTP response body is accessible to the user. Base64 encoding provides **no confidentiality** or protection against extraction.

---

## How to Prevent It

Sensitive data (flags, credentials, API secrets, private tokens) must **never be embedded in client-side HTML or templates**.

### ❌ Insecure Practice
```html
<div data-secret="c2Vuc2l0aXZlX2RhdGE=">
```

### ✅ Secure Practice
* Store sensitive assets and authorization logic on the server side.
* Only send data to the client that the current user is authenticated and authorized to view.
* Never rely on encoding (Base64, Hex, ROT13) to protect secrets.
* Audit generated templates and client scripts for accidental information disclosure before deployment.

---

## Attack Flow

```text
Web Application
      │
      ▼
Open Developer Tools
      │
      ▼
Inspect HTML Elements
      │
      ▼
Find custom attribute (notify_true)
      │
      ▼
Extract Base64 string
      │
      ▼
Base64 Decode
      │
      ▼
picoCTF{web_successfully_dec0ded_283e62fe}
```

---

## Key Takeaway

> [!IMPORTANT]
> **If the browser can receive it, the user can inspect it.**
