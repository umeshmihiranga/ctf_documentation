# Where Are the Robots?

## Challenge Information

- **Platform:** picoCTF
- **Year:** 2019
- **Category:** Web Exploitation
- **Difficulty:** Easy
- **Challenge:** Where Are the Robots?

## Objective

Find the hidden flag on the target web server.

---

## Enumeration

The challenge asks:

> Can you find the robots?

This suggests checking the website's `robots.txt` file.

### Check `robots.txt`

```bash
curl -s http://fickle-tempest.picoctf.net:60267/robots.txt
```

### Output

```text
User-agent: *
Disallow: /cc6b1.html
```

The `robots.txt` file reveals a disallowed path:

```text
/cc6b1.html
```

Although `robots.txt` tells search engine crawlers not to access this location, it does not prevent users from accessing it directly.

---

## Exploitation

Request the discovered page:

```bash
curl -s http://fickle-tempest.picoctf.net:60267/cc6b1.html
```

The page reveals the flag.

Alternatively, the following URL can be opened directly in a browser:

```text
http://fickle-tempest.picoctf.net:60267/cc6b1.html
```

---

## Flag

```text
picoCTF{ca1culat1ng_Mach1n3s_cc6b1}
```

---

## Key Takeaways

* `robots.txt` is used to provide crawling instructions to search engines.
* `Disallow` entries can reveal hidden or interesting web paths.
* `robots.txt` is **not an access-control mechanism**.
* During web enumeration, always check common files such as:

  * `/robots.txt`
  * `/sitemap.xml`
  * `/security.txt`

## Commands Used

```bash
curl -s http://fickle-tempest.picoctf.net:60267/robots.txt
curl -s http://fickle-tempest.picoctf.net:60267/cc6b1.html
```

## Attack Technique

**Robots.txt Enumeration → Hidden Path Discovery → Direct Resource Access**
