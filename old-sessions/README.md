## Challenge:

Old Sessions

## Category:

Web Exploitation

## Vulnerability:

Session Management / Session Hijacking

## Initial clue:

The challenge description mentioned that misconfigured session expiration could leave a user's session active indefinitely.

The server set the session cookie to expire in **2027**, indicating an excessively long session lifetime.

## Enumeration:

1. Inspected the application using the browser and DevTools.
2. Checked **Application → Cookies** and identified the `session` cookie.
3. Checked the application's pages and comments.
4. Discovered the `/sessions` endpoint from a comment.
5. `/sessions` exposed active session IDs and their associated users.
6. Found a valid session belonging to the `admin` user.

## Exploit:

1. Logged in as a normal user.
2. Obtained the admin session ID from `/sessions`.
3. Replaced the normal user's `session` cookie with the leaked admin session ID using browser DevTools.
4. Reloaded the application.
5. The server accepted the session as belonging to `admin`, resulting in account takeover and access to the flag.

## Tools:

* Kali Linux
* Firefox/Chromium browser
* Browser Developer Tools

  * Application → Cookies
  * Network
* `curl` for HTTP enumeration

## What I learned:

* Authentication often depends on a session cookie rather than repeatedly checking the password.
* A valid session token can effectively act as an authentication credential.
* Session IDs must be treated as sensitive information.
* Session hijacking can allow account takeover without knowing the victim's password.
* Always inspect cookies, HTTP requests, responses, and session-related endpoints during web CTFs.
* A useful methodology is: **enumerate → understand what the application trusts → find a weakness → replay/manipulate the trusted value → verify the impact.**

## Why the vulnerability existed:

The application had poor session management. Sessions were configured with an excessively long expiration time, while the `/sessions` endpoint exposed valid session identifiers and their associated users.

This allowed an attacker to obtain an administrator's still-valid session token and reuse it.

## How to prevent it:

* Never expose session IDs to users.
* Remove or restrict debugging/session-management endpoints.
* Use reasonable session expiration and idle timeouts.
* Invalidate sessions properly during logout.
* Rotate session IDs after authentication when appropriate.
* Use `HttpOnly`, `Secure`, and appropriate `SameSite` cookie attributes.
* Protect administrative functionality with proper authorization checks.
* Treat session tokens as sensitive credentials.
