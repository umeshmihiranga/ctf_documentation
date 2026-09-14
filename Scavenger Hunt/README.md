# 🕵️ picoCTF — Scavenger Hunt

## 📌 Challenge Information

- **Platform:** picoCTF
- **Category:** Web Exploitation
- **Challenge:** Scavenger Hunt
- **Target:** `http://wily-courier.picoctf.net:54246/`
- **Technique:** Web Reconnaissance / Information Disclosure
- **Difficulty:** Easy

---

## 🎯 Objective

Find the flag hidden across different files and resources on the website.

---

## 🔎 Enumeration

The challenge description indicated that interesting information was hidden around the website.

Instead of immediately attempting an exploit, I inspected the website's source code and followed the clues provided by the application.

---

## 1️⃣ Inspect the HTML Source

First, I retrieved the website:

```bash
curl -i http://wily-courier.picoctf.net:54246/
````

The HTML contained a comment revealing the first part of the flag:

```html
<!-- Here's the first part of the flag picoCTF{t -->
```

### Flag Part 1

```text
picoCTF{t
```

The HTML also referenced external resources:

```html
<link rel="stylesheet" href="mycss.css">
<script type="application/javascript" src="my.js"></script>
```

This indicated that the CSS and JavaScript files should also be inspected.

---

## 2️⃣ Inspect the CSS File

I retrieved the referenced CSS file:

```bash
curl http://wily-courier.picoctf.net:54246/mycss.css
```

A comment in the CSS contained the second flag fragment:

```text
# Part 2: h4ts_4_l0
```

### Flag Part 2

```text
h4ts_4_l0
```

---

## 3️⃣ Inspect the JavaScript File

Next, I checked the JavaScript:

```bash
curl http://wily-courier.picoctf.net:54246/my.js
```

The JavaScript did not directly contain the next flag fragment.

Instead, it contained the clue:

```text
/* How can I keep Google from indexing my website? */
```

### Reasoning

Google uses web crawlers to discover and index websites.

A standard file used to provide instructions to web crawlers is:

```text
robots.txt
```

Therefore, I checked:

```bash
curl http://wily-courier.picoctf.net:54246/robots.txt
```

---

## 4️⃣ Inspect robots.txt

The response contained:

```text
User-agent: *
Disallow: /index.html

# Part 3: t_0f_pl4c
# I think this is an apache server... can you Access the next flag?
```

### Flag Part 3

```text
t_0f_pl4c
```

The new clue mentioned **Apache**.

Apache is a commonly used web server, and one of its configuration files is:

```text
.htaccess
```

Therefore, I checked:

```bash
curl http://wily-courier.picoctf.net:54246/.htaccess
```

---

## 5️⃣ Inspect .htaccess

The server returned:

```text
# Part 4: 3s_2_lO0k
# I love making websites on my Mac, I can Store a lot of information there.
```

### Flag Part 4

```text
3s_2_lO0k
```

The next clue contained the words:

```text
Mac
Store
```

This suggested the macOS metadata file:

```text
.DS_Store
```

---

## 6️⃣ Inspect .DS_Store

I checked:

```bash
curl http://wily-courier.picoctf.net:54246/.DS_Store
```

This revealed the final flag fragment:

```text
_9588550}
```

---

## 🏁 Final Flag

Combining all the discovered fragments:

```text
picoCTF{th4ts_4_l0t_0f_pl4c3s_2_lO0k_9588550}
```

---

## 🧠 What I Learned

This challenge demonstrated basic web reconnaissance and information disclosure.

### Important files and concepts

| Resource     | Purpose                     | Why it mattered                                              |
| ------------ | --------------------------- | ------------------------------------------------------------ |
| `HTML`       | Defines webpage structure   | Hidden comments can contain sensitive information            |
| `.css`       | Controls webpage appearance | CSS comments/resources can contain clues                     |
| `.js`        | Client-side JavaScript      | Can contain clues, secrets, or references to other resources |
| `robots.txt` | Gives crawler instructions  | Can reveal interesting paths and directories                 |
| `.htaccess`  | Apache configuration        | Can be accidentally exposed                                  |
| `.DS_Store`  | macOS Finder metadata       | Can be accidentally uploaded to web servers                  |

---

## 🔗 Attack/Discovery Chain

```text
Website
   │
   ├── HTML source
   │      └── Flag Part 1
   │
   ├── mycss.css
   │      └── Flag Part 2
   │
   └── my.js
          │
          └── "Google indexing?"
                  │
                  └── robots.txt
                          │
                          ├── Flag Part 3
                          │
                          └── "Apache?"
                                  │
                                  └── .htaccess
                                          │
                                          ├── Flag Part 4
                                          │
                                          └── "Mac / Store"
                                                  │
                                                  └── .DS_Store
                                                          │
                                                          └── Final Flag
```

---

## 🛡️ Security Takeaways

The challenge demonstrates how accidentally exposed files can leak information.

### Common files worth checking during web reconnaissance

```text
/robots.txt
/sitemap.xml
/.htaccess
/.DS_Store
/.git/
/.env
```

These files are **not automatically vulnerabilities**, but accidental exposure can reveal useful information.

For example:

* `robots.txt` may reveal interesting directories.
* `.git/` may expose source code and repository history.
* `.env` may expose application configuration or secrets.
* `.DS_Store` may reveal filesystem metadata.
* `.htaccess` may reveal Apache configuration.

### Important distinction

`robots.txt` is **not an access-control mechanism**.

```text
Disallow: /admin/
```

means:

> "Crawlers should not crawl `/admin/`."

It does **not** mean:

> "Users cannot access `/admin/`."

Actual protection must be implemented by the web server or application through authentication and authorization.

---

## 💡 Key Lesson

The main skill tested by this challenge was not memorizing filenames.

It was **following clues and connecting them to technologies**:

```text
Google
   ↓
Web crawler
   ↓
robots.txt

Apache
   ↓
.htaccess

Mac
   ↓
.DS_Store
```

This is a fundamental web reconnaissance technique:

> **Identify a technology or concept → determine the associated resource → inspect it → follow the next clue.**

````
