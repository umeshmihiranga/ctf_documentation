# 🔍 Inspect HTML — picoCTF

## 📌 Challenge Information

- **Platform:** picoCTF
- **Category:** Web Exploitation
- **Challenge:** Inspect HTML
- **Difficulty:** Easy
- **Technique:** HTML Source Inspection

---

## 📝 Description

The challenge asks us to visit a website and discover the hidden flag.

The important clue is the challenge name:

> **Inspect HTML**

This suggests that the flag may be hidden inside the webpage's HTML source rather than being displayed normally.

---

## 🔎 Enumeration / Investigation

After opening the provided website, the page displayed an article about **Histiaeus**.

Instead of interacting with the visible content, I inspected the page using the browser's **Developer Tools**.

### Step 1 — Open Developer Tools

I opened Developer Tools using:

```text
F12
```

or:

```text
Right Click → Inspect
```

Then I selected the **Elements** tab.

---

### Step 2 — Inspect the HTML

Looking through the HTML source, I found a comment containing the flag.

The relevant section looked like:

```html
<!-- picoCTF{1n5p3ct0r_0f_h7ml_1fd8425b} -->
```

The flag was hidden inside an HTML comment, so it was not visible on the actual webpage.

---

## 🚩 Flag

```text
picoCTF{1n5p3ct0r_0f_h7ml_1fd8425b}
```

---

## 🧠 What I Learned

This challenge demonstrates that information can be hidden inside a webpage's **HTML source code**.

HTML comments are not rendered on the webpage:

```html
<!-- This is hidden from the user -->
```

However, anyone can inspect the page source or use Developer Tools to see them.

### Important lesson

When doing basic web reconnaissance, don't only look at what is visually displayed.

Check:

* HTML source
* HTML comments
* Hidden elements
* JavaScript
* Page metadata
* Form fields
* Developer Tools

---

## 🛠️ Tools Used

* Web Browser
* Browser Developer Tools
* Elements / HTML Inspector

---

## 🎯 Takeaway

The key idea was simply:

```text
Website
   ↓
Inspect HTML
   ↓
Find HTML comment
   ↓
Extract flag
```

This was a basic example of **information disclosure through client-side HTML**.