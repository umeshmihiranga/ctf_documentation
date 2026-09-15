
# Unminify — picoCTF

## Challenge

**Unminify**

## Category

**Web Exploitation**

## Difficulty

**Easy**

## Platform

**picoCTF / picoGym**

---

## Vulnerability / Technique

**Information Disclosure through HTML Source Code**

The flag was hidden directly inside the webpage's HTML source. The HTML was **minified**, meaning unnecessary whitespace and line breaks had been removed to make the page smaller.

There was no complicated vulnerability or exploit required. The important technique was **inspecting the raw HTML source** instead of only looking at the rendered webpage.

---

## Initial Clue

The challenge description said:

> "I don't like scrolling down to read the code of my website, so I've squished it."

This was the key clue.

The word **"squished"** refers to **minification**.

### What is minification?

Minification removes unnecessary characters such as:

* Spaces
* New lines
* Indentation
* Sometimes comments

For example:

### Normal HTML

```html
<html>
    <body>
        <h1>Hello</h1>
        <p>Welcome!</p>
    </body>
</html>
```

### Minified HTML

```html
<html><body><h1>Hello</h1><p>Welcome!</p></body></html>
```

The browser can still understand both versions.

The minified version is simply smaller and can be transferred more efficiently.

---

# Enumeration

The first step was to inspect the webpage normally.

The page itself did not visibly display the flag.

Instead of assuming the flag required an exploit, we followed the clue and inspected the **page source**.

### Browser Method

In the browser:

```text
Right Click → View Page Source
```

or:

```text
Ctrl + U
```

The source was minified, making it difficult to read.

---

## Terminal Enumeration

We also investigated the page using `curl`.

Initially, we used:

```bash
curl -s http://titan.picoctf.net:56075/
```

which returned nothing because the challenge instance/port had changed.

We then used the currently assigned instance:

```bash
curl -v http://titan.picoctf.net:54535/
```

The `-v` option means **verbose mode**.

It showed the HTTP request and response:

```text
> GET / HTTP/1.1
> Host: titan.picoctf.net:54535
> User-Agent: curl/8.21.0
```

and:

```text
< HTTP/1.1 200 OK
< Content-Type: text/html
< Content-Length: 1352
```

Most importantly, `curl` displayed the **raw HTML returned by the server**.

---

# Finding the Flag

While examining the minified HTML, we found:

```html
<p class="picoCTF{pr3tty_c0d3_743d0f9b}"></p>
```

The flag was therefore stored directly inside an HTML attribute.

The flag was:

```text
picoCTF{pr3tty_c0d3_743d0f9b}
```

---

# Exploit / Solution

There wasn't a traditional exploit in this challenge.

The solution was:

```text
Open webpage
     ↓
Read challenge description
     ↓
"Squished" → Minified HTML
     ↓
View Page Source
     ↓
Inspect HTML
     ↓
Find hidden class attribute
     ↓
Flag
```

Using the terminal:

```bash
curl -v http://titan.picoctf.net:54535/
```

allowed us to see the raw HTML directly.

---

# Tools Used

| Tool             | Purpose                                |
| ---------------- | -------------------------------------- |
| Browser          | View the webpage                       |
| View Page Source | Inspect raw HTML                       |
| `curl`           | Retrieve the webpage from the terminal |
| `curl -v`        | Display HTTP request/response details  |
| HTML inspection  | Locate the hidden flag                 |

---

# What I Learned

### 1. Always inspect page source

A webpage can display one thing while the underlying HTML contains additional information.

Useful browser shortcut:

```text
Ctrl + U
```

---

### 2. Understand minification

Minification makes HTML, CSS, and JavaScript smaller by removing unnecessary characters.

When a challenge mentions words such as:

```text
squished
compressed
minified
small
optimized
```

I should consider **minified source code** as a possible clue.

---

### 3. `curl` is useful for web enumeration

Instead of relying only on a browser:

```bash
curl http://target/
```

can retrieve the raw response.

For more information:

```bash
curl -v http://target/
```

The verbose output helps identify:

* HTTP status code
* Server
* Content type
* Headers
* Response body

---

### 4. Look beyond visible content

The flag wasn't displayed as normal text.

It was hidden inside:

```html
class="picoCTF{...}"
```

So during web enumeration, I should inspect:

```text
HTML attributes
HTML comments
Hidden elements
JavaScript
CSS
Source code
Metadata
```

---

# Why the Information Disclosure Existed

The developer placed the flag directly inside the HTML source:

```html
class="picoCTF{...}"
```

HTML sent to the browser is **not secret**.

Anything included in the response can potentially be viewed by the user.

Minifying the HTML does **not** provide security.

It only makes the source harder for humans to read.

---

# How to Prevent It

If sensitive information must remain secret, **do not place it in client-side HTML**.

For example, this is insecure:

```html
<div class="secret">
    SECRET_PASSWORD
</div>
```

because the user can simply view the source.

Instead:

* Keep secrets on the server.
* Never put passwords/API keys in HTML.
* Don't rely on minification as a security mechanism.
* Don't assume hidden HTML elements are actually secret.
* Perform authorization and access control on the server.

A useful security principle is:

> **If the browser receives it, the user can potentially inspect it.**

---

# Key Takeaway

The main lesson from **Unminify** is:

```text
Minification ≠ Encryption
```

Minified code may look difficult to read, but it is still completely accessible to the user.

For future web CTFs, my basic enumeration checklist should be:

```text
1. View the webpage
2. View Page Source
3. Inspect HTML
4. Check comments
5. Check hidden elements/attributes
6. Inspect JavaScript
7. Check requests/responses
8. Use curl when appropriate
```

### Flag

```text
picoCTF{pr3tty_c0d3_743d0f9b}
```
