# InsP3ct0r — picoCTF 2019

## 📝 Challenge Description

The challenge provides a simple website and hints that the code may need **inspection**. The goal is to inspect the website's source files and find the flag split across the **HTML, CSS, and JavaScript**.

---

## 🔍 Enumeration

Opened the provided challenge instance:

```text
http://fickle-tempest.picoctf.net:49483/
```

Instead of only looking at the rendered webpage, inspected the page source and its referenced files.

---

## 1️⃣ HTML — Part 1/3

Using:

```bash
curl -s http://fickle-tempest.picoctf.net:49483/ | grep -i pico
```

Output:

```html
<!-- Html is neat. Anyways have 1/3 of the flag: picoCTF{tru3_d3 -->
```

First part:

```text
picoCTF{tru3_d3
```

---

## 2️⃣ CSS — Part 2/3

The HTML references `mycss.css`, so inspected it:

```bash
curl -s http://fickle-tempest.picoctf.net:49483/mycss.css | tail -5
```

Output:

```text
/* You need CSS to make pretty pages. Here's part 2/3 of the flag: t3ct1ve_0r_ju5t */
```

Second part:

```text
t3ct1ve_0r_ju5t
```

---

## 3️⃣ JavaScript — Part 3/3

The HTML also references `myjs.js`:

```bash
curl -s http://fickle-tempest.picoctf.net:49483/myjs.js | tail -5
```

Output:

```text
/* Javascript sure is neat. Anyways part 3/3 of the flag: _lucky?302945a7} */
```

Third part:

```text
_lucky?302945a7}
```

---

## 🚩 Flag

Combining all three parts:

```text
picoCTF{tru3_d3t3ct1ve_0r_ju5t_lucky?302945a7}
```

---

## 🧠 What I Learned

* Important information can be hidden in **HTML comments**.
* Web challenges may hide secrets inside referenced **CSS and JavaScript files**.
* `curl` can quickly retrieve source files without relying on browser DevTools.
* `grep` is useful for locating comments and keywords.
* Always inspect **all resources referenced by a webpage**, not just the HTML.

### Key commands

```bash
curl -s http://TARGET/
curl -s http://TARGET/mycss.css
curl -s http://TARGET/myjs.js
```

The challenge demonstrates a basic but important web-security technique: **inspect the application's client-side source code for exposed information.**
