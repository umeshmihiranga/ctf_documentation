

# 🔖 picoCTF — Bookmarklet

## Challenge

**Bookmarklet**

## Category

**Web Exploitation**

## Vulnerability / Concept

**JavaScript Bookmarklet / Client-Side Code Obfuscation**

## Initial Clue

The webpage says:

> "Here's a bookmarklet for you to try"

This was the main clue. Instead of looking for another endpoint or hidden file, we needed to understand and execute the provided JavaScript bookmarklet.

---

## Enumeration

Started by requesting the challenge webpage:

```bash
curl http://titan.picoctf.net:62715/
```

The response contained a `<textarea>` containing a JavaScript bookmarklet:

```javascript
javascript:(function() {
    var encryptedFlag = "àÒÆÞ¦È¬ëÙ£ÖÓÚåÛÑ¢ÕÓ¨ÍÕÄ¦í";
    var key = "picoctf";
    var decryptedFlag = "";

    for (var i = 0; i < encryptedFlag.length; i++) {
        decryptedFlag += String.fromCharCode(
            (encryptedFlag.charCodeAt(i)
            - key.charCodeAt(i % key.length)
            + 256) % 256
        );
    }

    alert(decryptedFlag);
})();
```

The important discoveries were:

```javascript
var encryptedFlag = "...";
var key = "picoctf";
```

and the decryption loop:

```javascript
(encryptedFlag.charCodeAt(i)
 - key.charCodeAt(i % key.length)
 + 256) % 256
```

---

## Understanding the Bookmarklet

A normal bookmark contains a URL:

```text
https://example.com
```

A bookmarklet instead contains JavaScript:

```text
javascript:alert("Hello")
```

When the bookmarklet is clicked, the browser executes the JavaScript.

In this challenge:

```text
Bookmarklet
     ↓
JavaScript executes
     ↓
Encrypted flag
     ↓
Decryption using "picoctf"
     ↓
alert(decryptedFlag)
     ↓
FLAG
```

---

## Exploitation

The bookmarklet was copied from the challenge page and saved as a browser bookmark.

Clicking the bookmark caused the browser to execute the supplied JavaScript.

The JavaScript:

1. Read the encrypted flag.
2. Used `picoctf` as the key.
3. Iterated through every encrypted character.
4. Subtracted the corresponding key character.
5. Converted the resulting character code back into a character.
6. Built the decrypted flag.
7. Displayed it using `alert()`.

The core operation was:

```javascript
String.fromCharCode(
    (encryptedFlag.charCodeAt(i)
    - key.charCodeAt(i % key.length)
    + 256) % 256
);
```

---

## Flag

```text
picoCTF{...}
```

**Flag obtained by executing the bookmarklet in the browser.**

> Keep your actual flag in your private/local write-up if this repository is public.

---

## Tools Used

```text
curl
Browser
Browser Developer Tools
JavaScript
```

---

## What I Learned

### 1. What is a bookmarklet?

A bookmarklet is a browser bookmark containing JavaScript instead of a normal URL.

```text
Normal bookmark:
https://example.com

Bookmarklet:
javascript:alert("Hello")
```

Clicking the bookmarklet executes the JavaScript in the browser.

### 2. Bookmarklets can interact with the current webpage

For example:

```javascript
javascript:alert(document.title)
```

can read the current page's title.

They can also manipulate the DOM.

### 3. Bookmarklets don't automatically bypass browser security

A bookmarklet is still subject to the browser's security model.

It doesn't automatically provide:

```text
Root access
Filesystem access
Access to every website
Access to HttpOnly cookies
Unlimited cross-origin access
```

### 4. Client-side code can contain secrets

The encrypted flag and decryption algorithm were both delivered to the browser.

Therefore, the encryption didn't provide meaningful secrecy — the information needed to decrypt it was already available to the client.

---

## Why the Vulnerability / Weakness Existed

The challenge intentionally placed the encrypted flag **and the decryption logic** in client-side JavaScript.

Anyone receiving the webpage could inspect:

```javascript
encryptedFlag
```

and:

```javascript
key
```

along with the algorithm used to decrypt them.

This demonstrates an important security principle:

> **Never assume information is secret if it is delivered to the client.**

If a browser receives both a secret and everything necessary to decrypt that secret, the user can ultimately inspect and recover it.

---

## How to Prevent It

For a real application:

### ❌ Don't put secrets in client-side JavaScript

Avoid:

```javascript
const secret = "MY_SECRET";
```

### ❌ Don't rely on client-side encryption to hide sensitive information

If the browser has:

```text
Encrypted data
+
Decryption key
+
Decryption algorithm
```

the user can inspect all three.

### ✅ Keep sensitive operations server-side

Instead:

```text
Browser
   ↓
Request
   ↓
Server
   ↓
Sensitive operation
   ↓
Server response
```

The server should control sensitive data and authorization.

---

# Key Takeaway

The biggest lesson from this challenge wasn't actually the decryption.

It was recognizing:

```text
javascript:
```

at the beginning of the bookmark.

That tells the browser:

> **"Execute this bookmark as JavaScript."**

So when you see **Bookmarklet** in a CTF, your first thought should be:

```text
Inspect the bookmarklet
        ↓
Understand the JavaScript
        ↓
Look for encoded/encrypted data
        ↓
Understand the transformation
        ↓
Execute/decode it
```

**Challenge solved by understanding how browser bookmarklets execute client-side JavaScript.**
