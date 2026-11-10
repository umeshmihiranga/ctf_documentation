# CTF Write-up: findme

## 1. Challenge Information

| Field | Details |
|---|---|
| Challenge | findme |
| Platform | CyLab Security Academy |
| Original Event | picoCTF 2023 |
| Category | Web Exploitation |
| Difficulty | Medium |
| Challenge Author | Geoffrey Njogu |
| Status | Solved |

**Flag:**
```text
academy{proxies_all_the_way_af51e23b}
```

---

## 2. Challenge Description

The challenge provided a web application with a login form.

The instructions explicitly provided these credentials:

```text
Username: test
Password: test
```

After successful authentication, the application redirected us through multiple pages before reaching the home page.

The objective was to investigate the application's behavior and discover the hidden flag.

---

## 3. Initial Reconnaissance

### Step 1: Access the application

Opened the challenge instance and logged in using:

```text
Username: test
Password: test
```

After authentication, the application redirected us to:

```text
/home
```

The home page displayed:

```text
Welcome fellow Human
```

It also contained a search form:

```text
Search for flags
```

### Step 2: Inspect the page source

Used browser Developer Tools:

```text
F12 → Sources → home
```

Found the following JavaScript function:

```javascript
function search() {
    const searchResult =
        document.getElementById("search-result");

    const searchTerm =
        document.getElementById("search-term");

    const keyword =
        document.getElementById("keyword");

    const comments =
        document.getElementById("comments");

    comments.style.visibility = "hidden";

    searchResult.style.display = "block";

    searchTerm.innerHTML = keyword.value;

    // fetching results related to the keyword value from the server

    return false;
}
```

### Observations

1. The function retrieves the search keyword.
2. It attempts to access an HTML element with the ID `comments`.
3. The HTML source did not contain an element with that ID.
4. Therefore, accessing `comments.style` causes a JavaScript error.
5. The search functionality does not execute normally.

Verified this using the browser console:

```javascript
document.getElementById("comments")
```

Output:

```javascript
null
```

**Learning:** JavaScript errors can interrupt the execution of subsequent statements. Also, `innerHTML` can introduce DOM-based XSS risks when untrusted input is inserted without sanitization, although that was not the vulnerability we ultimately used.

---

## 4. Investigating the Authentication Flow

Instead of focusing on the broken search feature, we investigated the application's HTTP requests.

### Step 1: Open the Network tab

Opened:

```text
F12 → Network
```

In Brave, the option was labelled:

```text
Keep log
```

Enabled it to preserve network requests when navigating between pages.

### Step 2: Capture the requests

Cleared the existing requests and logged in again.

Observed requests resembling:

```text
login
id=YWNh...
id=bF90...
home
```

The login request returned:

```text
302
```

The two intermediate ID requests showed:

```text
304
```

These responses required careful interpretation.

- HTTP 302: Redirect response.
- HTTP 304: Not Modified; allows the browser to reuse a cached response.

The important discovery was the two different `id` values in the intermediate URLs.

---

## 5. Base64 Investigation

The intermediate URL contained an ID that looked like Base64-encoded data.

Example URL structure:

```text
/next-page/id=ENCODED_VALUE
```

Instead of treating the ID as a random string, we investigated whether it contained readable information.

### Step 1: Extract the URL

Used the browser console:

```javascript
location.href
```

This returned the complete current URL.

### Step 2: Decode using Python

First entered the Python interpreter:

```bash
python3
```

Then executed:

```python
import base64

value = "PASTE_ID_HERE"

decoded = base64.b64decode(
    value + "=" * (-len(value) % 4)
)

print(decoded)
```

The padding calculation adds the required `=` characters when they are missing.

### Step 3: Decode using JavaScript

Alternatively, used the browser console:

```javascript
const id = location.pathname.split("id=")[1];

console.log(atob(id));
```

This extracts the ID from the URL path and decodes its Base64 representation.

### Step 4: Identify the flag fragments

The two decoded values were:

**First fragment:**

```text
academy{proxies_al
```

**Second fragment:**

```text
l_the_way_af51e23b}
```

The fragments needed to be combined in their original order.

---

## 6. Flag Reconstruction

Combined both decoded fragments:

```text
academy{proxies_al
+
l_the_way_af51e23b}
```

Final flag:

```text
academy{proxies_all_the_way_af51e23b}
```

The flag was successfully submitted and accepted.

---

# 7. Extra Learning – Technical Concepts

## A. HTTP Redirects

A redirect tells the browser that a requested resource is available at another location.

Example:

```http
HTTP/1.1 302 Found
Location: /home
```

The browser follows the provided destination.

Important HTTP status codes:

| Code | Meaning |
|---|---|
| 200 | OK |
| 301 | Moved Permanently |
| 302 | Found (temporary redirect) |
| 303 | See Other |
| 304 | Not Modified |
| 307 | Temporary Redirect |
| 308 | Permanent Redirect |

### Important distinction

A 302 and a 304 are not the same.

- 302 is a redirect.
- 304 is related to HTTP caching.

## B. Browser Network Inspection

The Network tab helps us understand communication between the browser and the server.

Important sections:

| Section | Purpose |
|---|---|
| Headers | Request and response headers |
| Payload | Data sent to the server |
| Preview | Formatted response |
| Response | Raw response content |
| Initiator | What triggered the request |
| Timing | Request execution timing |

This is particularly useful when investigating authentication, redirects, APIs and hidden parameters.

## C. Base64 Encoding

Base64 represents binary data using printable ASCII characters.

Example:

Original:

```text
Hello
```

Encoded:

```text
SGVsbG8=
```

Decode using Kali Linux:

```bash
echo -n "SGVsbG8=" | base64 -d
```

Output:

```text
Hello
```

Encode:

```bash
echo -n "Hello" | base64
```

### Important security concept

Base64 is NOT encryption.

It does not use a secret key and does not provide confidentiality.

Anyone who obtains a Base64 value can decode it.

## D. URL Parameters

URLs can carry information through parameters.

Example:

```text
https://example.com/profile?id=123
```

Here:

- `id` is the parameter name.
- `123` is its value.

Two common URL structures:

Query parameter:

```text
/profile?id=123
```

Path parameter:

```text
/profile/123
```

Understanding URL structures helps when investigating authentication flows and manipulating application inputs.

## E. Multi-stage Redirects

An application can involve multiple intermediate requests before reaching the final page.

For example:

```text
Browser
   |
   v
/login
   |
   v
/next-page/id=VALUE1
   |
   v
/next-page/id=VALUE2
   |
   v
/home
```

The important lesson is to investigate intermediate requests instead of examining only the final destination.

---

# 8. Useful Commands and Tools

### Browser Developer Tools

```text
F12 → Network
F12 → Sources
F12 → Console
```

### Inspect current URL

```javascript
location.href
```

### Decode Base64 using Kali

```bash
echo -n "BASE64_VALUE" | base64 -d
```

### Encode Base64

```bash
echo -n "TEXT" | base64
```

### Python Base64 decoding

```python
import base64

encoded = "SGVsbG8="

decoded = base64.b64decode(encoded)

print(decoded.decode())
```

### Inspect HTTP headers

```bash
curl -I https://example.com
```

### Verbose HTTP inspection

```bash
curl -v https://example.com
```

---

# 9. Lessons Learned

Through this challenge, I learned:

- How HTTP redirect responses work.
- How to inspect browser requests and responses.
- How to use the Network tab's Keep log feature.
- How to distinguish HTTP 302 from HTTP 304.
- How to identify Base64-encoded values.
- How to decode Base64 using Python and Linux commands.
- How to extract information from URL paths.
- How to investigate multi-stage application navigation.
- Why understanding application behavior is more useful than blindly trying payloads.
- How JavaScript errors can interrupt client-side functionality.

---

# 10. Personal Takeaway

The biggest lesson from this challenge was that interesting information is not always visible on the final webpage.

Initially, I focused on the search form and attempted HTML and JavaScript payloads. However, the actual breakthrough came from inspecting the browser's Network tab.

By examining the intermediate requests, identifying the encoded URL parameters and decoding them, I recovered the complete flag.

**My key takeaway:**

Don't just inspect what the browser displays. Investigate what happens behind the scenes.

---

**Challenge completed:** `academy{proxies_all_the_way_af51e23b}`

**Category:** Web Exploitation  
**Main concepts:** HTTP Redirects, Network Inspection, Base64, URL Parameters and Multi-stage Navigation.
