# picoCTF - Includes

## Challenge Information

- **Category:** Web Exploitation
- **Difficulty:** Easy
- **Platform:** picoCTF
- **Challenge:** Includes
- **Event:** picoCTF 2022

---

## Objective

Find the flag hidden within the files included by a webpage.

---

## Enumeration

After opening the challenge website, I inspected the page using **Browser Developer Tools**.

I went to:

```text
DevTools → Sources
```

The page loaded several resources:

```text
(index)
script.js
style.css
```

Since the challenge was named **Includes**, I suspected that important information might be hidden inside one of the included CSS or JavaScript files.

---

## Step 1 — Inspect `style.css`

Inside `style.css`, I found the following comment:

```css
/* picoCTF{1nclu51v17y_1of2_ */
```

This appeared to be the **first part of the flag**.

---

## Step 2 — Inspect `script.js`

Next, I opened `script.js`.

At the bottom of the file, I found:

```javascript
// f7w_2of2_b8f4b022}
```

This was the **second part of the flag**.

---

## Step 3 — Combine the Two Parts

Combining both pieces:

```text
picoCTF{1nclu51v17y_1of2_f7w_2of2_b8f4b022}
```

### Flag

```text
picoCTF{1nclu51v17y_1of2_f7w_2of2_b8f4b022}
```

---

## Key Concept

Webpages commonly load external resources such as:

```html
<link rel="stylesheet" href="style.css">
<script src="script.js"></script>
```

These files are sent to the browser and therefore can be inspected by the user.

Sensitive information accidentally stored inside:

* HTML
* CSS
* JavaScript
* comments
* other client-side resources

should be considered **publicly accessible**.

---

## What I Learned

* How to use **DevTools → Sources** to inspect webpage resources.
* External CSS and JavaScript files are accessible to the client.
* HTML source inspection alone may not reveal everything.
* Challenge names can provide useful hints about where to look.
* Client-side code should **never contain secrets** such as passwords, API keys, or sensitive data.

---

## Tools Used

* Web Browser
* Browser Developer Tools
* Sources panel

---

## Conclusion

The flag was split between two externally included files. Inspecting `style.css` revealed the first half, while `script.js` contained the second half.

**Lesson:** Always inspect the complete set of resources loaded by a webpage, not just the main HTML document.