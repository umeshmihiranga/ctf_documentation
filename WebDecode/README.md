# WebDecode — picoCTF 2024

## Challenge Information

| Field                         | Details                                    |
| ----------------------------- | ------------------------------------------ |
| **Challenge**                 | WebDecode                                  |
| **Category**                  | Web Exploitation                           |
| **Difficulty**                | Easy                                       |
| **Platform**                  | picoCTF 2024                               |
| **Target**                    | `http://titan.picoctf.net:58403/`          |
| **Vulnerability / Technique** | Information disclosure through HTML source |
| **Encoding**                  | Base64                                     |

---

## Initial Clue

The challenge tells us:

> **“Do you know how to use the web inspector?”**

This strongly suggests that the flag may be present somewhere in the webpage's HTML rather than requiring a complicated exploit.

---

## Enumeration

### 1. Open the target

Navigate to:

```text
http://titan.picoctf.net:58403/
```

### 2. Open Developer Tools

Use:

```text
F12
```

or:

```text
Ctrl + Shift + I
```

Then open the **Elements / Inspector** tab.

### 3. Inspect the HTML

Searching the page source for interesting terms such as:

```text
picoCTF
```

or:

```text
flag
```

led us to:

```html
<section class="about" notify_true="cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMjgzZTYyZmV9">
    <h1>
        Try inspecting the page!! You might find it there
    </h1>
</section>
```

The important discovery was the custom HTML attribute:

```text
notify_true="cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMjgzZTYyZmV9"
```

---

## Identifying the Encoding

The value:

```text
cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMjgzZTYyZmV9
```

has characteristics of **Base64** encoding.

For example, Base64 commonly contains:

```text
A-Z
a-z
0-9
+
/
=
```

The string also decodes into readable text.

---

## Exploit / Decode

We can decode the value using Kali Linux:

```bash
echo 'cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMjgzZTYyZmV9' | base64 -d
```

This reveals:

```text
picoCTF{web_successfully_dec0ded_283e62fe}
```

### Flag

```text
picoCTF{web_successfully_dec0ded_283e62fe}
```

---

## Tools Used

* **Firefox / Chromium Developer Tools**
* **HTML Inspector**
* **Kali Linux**
* `echo`
* `base64`

---

## What I Learned

### 1. Always inspect the HTML

The information displayed by a webpage isn't necessarily everything contained in the page.

HTML can contain:

* Comments
* Hidden elements
* Custom attributes
* Metadata
* Encoded values
* JavaScript variables
* Hidden links

For example:

```html
<div notify_true="...">
```

isn't something normally visible to the user, but it can still be read through Developer Tools.

### 2. Recognize common encodings

When you encounter a suspicious string, don't immediately assume it's encryption.

Common things to check include:

```text
Base64
URL encoding
Hex
Binary
ASCII
ROT13
```

A string containing letters/numbers and ending with `=` is often worth testing as Base64.

### 3. Developer Tools are important for web security

The browser inspector is useful not only for CTFs but also during real web security testing.

It allows us to examine:

```text
HTML
CSS
JavaScript
HTTP requests
Cookies
Storage
DOM
Network traffic
```

---

## Why the Vulnerability Existed

The flag was effectively exposed to the client.

The application placed the encoded flag directly inside an HTML attribute:

```html
notify_true="BASE64_VALUE"
```

Anything sent to the browser should be considered accessible to the user.

**Encoding is not encryption.**

Base64 does not provide confidentiality. Anyone who finds the value can decode it.

---

## How to Prevent It

Sensitive information such as flags, passwords, API keys, tokens, or secrets should **never be placed in client-side HTML**.

Bad:

```html
<div data-secret="c2Vuc2l0aXZlX2RhdGE=">
```

Better:

* Keep secrets server-side.
* Never send unnecessary sensitive data to the client.
* Do not rely on Base64 or other encoding to hide secrets.
* Review generated HTML and JavaScript for accidental information disclosure.
* Use proper authentication and authorization controls.

---

## Attack Flow

```text
Web Application
      │
      ▼
Open Developer Tools
      │
      ▼
Inspect HTML
      │
      ▼
Find hidden/custom attribute
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

> **If the browser can receive it, the user can inspect it.**
