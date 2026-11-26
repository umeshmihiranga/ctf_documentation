# CTF Write-up: Secrets

## Challenge Information

| Field | Details |
|---|---|
| Challenge | Secrets |
| Platform | picoCTF |
| Category | Web Exploitation |
| Difficulty | Medium |
| Status | Solved |
| Web Server | Nginx 1.31.5 |

---

## 1. Objective

The challenge description:

> We have several pages hidden. Can you find the one with the flag?

The objective was to investigate the target website, discover hidden directories and pages, and retrieve the flag.

Instead of immediately using automated directory enumeration tools, I started with manual web reconnaissance by inspecting HTTP responses and HTML source code.

## 2. Initial Reconnaissance

First, I inspected the website using `curl`.

```bash
curl -i http://TARGET:PORT/
```

Identified the web server:

```text
Server: nginx/1.31.5
```

### Checking robots.txt

I checked whether the website exposed a `robots.txt` file.

```bash
curl -i http://TARGET:PORT/robots.txt
```

Response:

```text
HTTP/1.1 404 Not Found
```

The file was not available, so I moved on to inspecting the website's HTML source.

## 3. HTML Source Inspection

Downloaded the homepage HTML:

```bash
curl -s http://TARGET:PORT/ -o index.html
```

Extracted links and resource references:

```bash
grep -Eoi '(href|src)="[^"]+"' index.html
```

Discovered references:

```text
href="vendor/bootstrap/css/bootstrap.min.css"
href="secret/assets/index.css"
href="#home"
href="about.html"
href="contact.html"
src="secret/assets/DX1KYM.jpg"
```

The following reference caught my attention:

```text
secret/assets/index.css
```

This suggested that the website contained a `/secret/` directory.

I also inspected HTML comments:

```bash
grep -inE '<!--|-->' index.html
```

No useful hidden information was discovered in the comments.

## 4. Discovering the First Hidden Directory

Accessed the `/secret/` directory:

```bash
curl -i http://TARGET:PORT/secret/
```

Response:

```text
HTTP/1.1 200 OK
```

The HTML response contained:

```html
<h1>Finally. You almost found me. you are doing well</h1>

<link rel="stylesheet" href="hidden/file.css" />
```

This revealed another interesting resource:

```text
/secret/hidden/file.css
```

The relative URL `hidden/file.css` was resolved against the current `/secret/` directory.

This suggested that another hidden directory existed:

```text
/secret/hidden/
```

## 5. Discovering the Hidden Login Page

Accessed the newly discovered directory:

```bash
curl -i http://TARGET:PORT/secret/hidden/
```

The server returned another HTML page containing a login form.

Two interesting references appeared in its source code.

### A. Hidden stylesheet reference

```html
<link href="superhidden/login.css" rel="stylesheet" />
```

### B. Hidden input field

```html
<input type="hidden" name="db" value="superhidden/xdfgwd.html" />
```

The hidden input suggested another possible resource:

```text
/secret/hidden/superhidden/xdfgwd.html
```

I attempted to access it:

```bash
curl -i http://TARGET:PORT/secret/hidden/superhidden/xdfgwd.html
```

Response:

```text
HTTP/1.1 404 Not Found
```

The referenced file was not available at that location.

### Investigating the JavaScript reference

The login page also contained:

```html
<form method="" action="/secret/assets/popup.js">
```

Retrieved the referenced JavaScript:

```bash
curl -i http://TARGET:PORT/secret/assets/popup.js
```

Response body:

```javascript
<script>alert(" Thank you for the attempt but oops! try harder")</script>;
```

This was a misleading clue rather than useful authentication logic.

### Investigating the login stylesheet

Retrieved the stylesheet:

```bash
curl -i http://TARGET:PORT/secret/hidden/superhidden/login.css
```

The response returned ordinary CSS styling for the login page.

No additional useful information was discovered in the stylesheet.

## 6. Investigating the Nested Directory

Instead of continuing to investigate the unavailable HTML file, I decided to inspect the directory itself.

Executed:

```bash
curl -i http://TARGET:PORT/secret/hidden/superhidden/
```

The server returned:

```text
HTTP/1.1 200 OK
Content-Type: text/html
```

This was the breakthrough.

The response contained:

```html
<!DOCTYPE html>
<html>
  <head>
    <title></title>
    <link rel="stylesheet" href="mycss.css" />
  </head>

  <body>
    <h1>Finally. You found me. But can you see me</h1>
    <h3 class="flag">academy{succ3ss_@h3n1c@10n_e77e3359}</h3>
  </body>
</html>
```

The flag was directly exposed inside the HTML source code.

The message:

```text
Finally. You found me. But can you see me
```

was another hint that encouraged inspecting the actual HTML response instead of relying only on the browser's visual presentation.

## 7. Final Flag

**Successfully captured flag:**

```text
academy{succ3ss_@h3n1c@10n_e77e3359}
```

The flag was found inside the following HTML element:

```html
<h3 class="flag">
    academy{succ3ss_@h3n1c@10n_e77e3359}
</h3>
```

Location:

```text
/secret/hidden/superhidden/
```

**Challenge Status: SOLVED**

---

## 8. Attack Path Summary

The complete discovery process:

```text
Homepage
   |
   | HTML source inspection
   v
/secret/
   |
   | Found hidden/file.css
   v
/secret/hidden/
   |
   | Inspected login page source
   | Discovered superhidden/
   v
/secret/hidden/superhidden/
   |
   | Retrieved HTML source
   v
FLAG DISCOVERED
```

## 9. Tools Used

| Tool | Purpose |
|---|---|
| curl | Sending HTTP requests and inspecting responses |
| grep | Extracting HTML references and comments |
| Terminal | Manual web reconnaissance |
| Browser | Viewing the target website |

## 10. Key Concepts Learned

### 1. HTML Source Inspection

A website's HTML source can expose resources and directory paths that are not visible through its normal navigation.

### 2. Relative URLs

Relative URLs are resolved against the current document's URL.

For example:

```text
Current directory: /secret/

Relative path: hidden/file.css

Resolved path: /secret/hidden/file.css
```

Understanding this helped follow the hidden resource references.

### 3. HTTP Status Codes

Learned to interpret common HTTP responses:

- `200 OK` – The server successfully returned a response.
- `404 Not Found` – The requested resource was not found.

A successful HTTP response does not necessarily mean that useful content exists. Response bodies and content lengths must also be inspected.

### 4. Hidden Directory Discovery

Websites can expose directories that are not linked from their homepage.

Following references in HTML source can reveal these locations.

### 5. Client-Side Information Disclosure

Hidden HTML input fields are still visible to anyone who can inspect the page source.

They can expose internal resource names, paths, and other application information.

### 6. JavaScript and CSS Inspection

Supporting resources can reveal application behavior and additional information.

However, not every referenced resource contains useful information.

### 7. Manual Web Reconnaissance

Following evidence from the application helped discover the hidden pages without relying on automated directory brute-forcing.

---

## 11. Lessons Learned

- Always inspect HTML source during web reconnaissance.
- Follow interesting links and resource references.
- Understand how relative URLs are resolved.
- Investigate nested directories instead of assuming they are inaccessible.
- Inspect HTTP response bodies, not just status codes.
- Hidden input fields are not secure storage.
- Do not assume every interesting-looking path is a valid resource.
- Avoid blindly relying on automated tools before understanding the application structure.
- Small clues in HTML can lead to significant discoveries.

## 12. Conclusion

This challenge demonstrated how manual web reconnaissance and HTML source inspection can uncover hidden resources on a web server.

By following references discovered in the homepage and subsequent hidden pages, I gradually navigated through multiple directory levels and eventually retrieved the flag directly from the final HTML response.

The challenge reinforced the importance of understanding HTTP responses, HTML structure, relative URLs, and client-side information disclosure when investigating web applications.

**Challenge successfully completed!**