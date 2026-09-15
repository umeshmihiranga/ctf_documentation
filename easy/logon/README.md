# picoCTF 2019 - logon

## Challenge

**Category:** Web Exploitation  
**Difficulty:** Easy  
**Challenge:** logon

### Description

> The factory is hiding things from all of its users.  
> Can you login as Joe and find what they've been looking at?

---

## Enumeration

After opening the challenge, I logged in and inspected the site's cookies using:

**Browser → Developer Tools → Application → Cookies**

The following cookies were present:

```text
admin=False
password=js
username=admin
```

The important cookie was:

```text
admin=False
```

This suggested that the application was trusting a client-side cookie to determine whether the user had administrator privileges.

---

## Exploitation

I modified the `admin` cookie from:

```text
False
```

to:

```text
True
```

No other cookies needed to be changed.

After refreshing the `/flag` page, the application accepted the modified cookie and displayed the flag.

---

## Flag

```text
picoCTF{th3_c0nsp1r4cy_l1v3s_4d184b0d}
```

---

## Vulnerability

The application suffers from **client-side authorization / insecure cookie-based access control**.

The server trusted the value of the `admin` cookie:

```text
admin=False
```

instead of securely determining the user's privileges on the server.

By changing:

```text
admin=False
```

to:

```text
admin=True
```

I was able to bypass the authorization check and access administrator-only content.

---

## Key Takeaway

Never trust authorization information supplied directly by the client.

Values such as:

```text
admin=True
role=admin
is_admin=1
```

should not be trusted simply because they are stored in browser cookies. Authorization decisions should be enforced server-side using properly protected session information.

---

## Tools Used

* Web Browser
* Chrome/Brave Developer Tools
* Application → Cookies
* HTTP Cookies

## Skills Demonstrated

* Web application enumeration
* Cookie inspection
* Client-side authorization bypass
* Access-control testing
