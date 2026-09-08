# Old Sessions — CTF Writeup

| Field | Details |
| :--- | :--- |
| **Challenge** | Old Sessions |
| **Category** | Web Exploitation |
| **Vulnerability** | Session Management / Session Hijacking |

---

## Challenge Overview

The application suffered from severe session management flaws: excessively long session lifetimes combined with an endpoint leaking active user session tokens, allowing an unauthenticated attacker to hijack an administrative session.

---

## Initial Clue

The challenge description mentioned that misconfigured session expiration could leave a user's session active indefinitely.

Inspecting the session cookie revealed that the server set the expiration date to **2027**, indicating an excessively long, persistent session lifetime.

---

## Enumeration

1. **Inspect Application in Browser:**
   Explored application functionality and inspected client-side storage via Developer Tools.
2. **Identify Session Cookie:**
   Navigated to `Application → Cookies` and identified the active `session` cookie.
3. **Inspect Page Source & Comments:**
   Reviewed HTML comments and application pages for hidden routes or developer notes.
4. **Discover Leaked Endpoint:**
   Discovered the `/sessions` endpoint referenced inside a developer comment.
5. **Inspect Leaked Sessions:**
   Navigated to `/sessions`. The endpoint publicly exposed active session IDs along with their associated usernames.
6. **Identify Administrative Token:**
   Identified an active session token belonging to the `admin` user.

---

## Exploitation

1. **Log in as Normal User:**
   Established a baseline session as an unprivileged user.
2. **Retrieve Admin Session ID:**
   Extracted the leaked admin session identifier from the `/sessions` endpoint.
3. **Session Replay / Cookie Replacement:**
   Using browser DevTools (`Application → Cookies`), replaced the unprivileged `session` cookie value with the leaked `admin` session ID.
4. **Reload Application:**
   Refreshed the browser.
5. **Account Takeover & Flag Retrieval:**
   The server blindly trusted the administrative session token, granting immediate administrative privileges and revealing the flag.

### Attack Flow
```text
Inspect HTML / Comments
         │
         ▼
Discover /sessions endpoint
         │
         ▼
Extract Admin Session ID
         │
         ▼
Replace session Cookie in DevTools
         │
         ▼
Refresh Page ──> Account Takeover ──> Admin Access & Flag
```

---

## Tools Used

* **Kali Linux**
* **Firefox / Chromium Browser**
* **Browser Developer Tools**
  * `Application → Cookies` (Cookie inspection and modification)
  * `Network` tab (Request & response analysis)
* **`curl`** (Command-line HTTP enumeration)

---

## What I Learned

* **Session tokens are bearer credentials:** Authentication frequently relies entirely on session cookies rather than repeatedly checking user credentials.
* **Possession equals identity:** Whoever holds a valid session token is treated by the server as that user, enabling account takeover without needing the user's password.
* **Session IDs are highly sensitive:** Exposing session identifiers in logs, APIs, or debugging endpoints completely compromises authentication security.
* **Session longevity amplifies risk:** Sessions that do not expire or remain valid indefinitely leave persistent windows for hijacking.
* **Core Web CTF Methodology:**
  ```text
  Enumerate ──> Identify Trusted State ──> Locate Weakness ──> Replay/Manipulate ──> Verify Impact
  ```

---

## Why the Vulnerability Existed

The application suffered from two critical session management flaws:
1. **Excessive Session Lifetimes:** Sessions were configured with multi-year expiration dates rather than short, inactivity-based timeouts.
2. **Exposed Session Tokens:** The `/sessions` debugging endpoint publicly disclosed valid session identifiers mapped to user accounts.

Because the server accepted valid session tokens without secondary validation (such as IP binding, user agent checks, or re-authentication), any user who obtained the admin token could impersonate the administrator.

---

## How to Prevent It

* **Never Expose Session IDs:** Session identifiers must remain secret and never be exposed in API responses, URLs, client-side scripts, or debug endpoints.
* **Remove Debug Endpoints:** Remove or strictly restrict access to administrative, profiling, and session-listing endpoints in production environments.
* **Implement Strict Timeouts:** Enforce reasonable absolute session expiration (e.g. 15–30 minutes) and idle inactivity timeouts.
* **Invalidate on Logout:** Ensure sessions are properly destroyed server-side upon user logout.
* **Rotate Session Identifiers:** Re-issue new session tokens upon authentication, privilege changes, and critical actions.
* **Use Secure Cookie Flags:**
  * `HttpOnly`: Prevents JavaScript access to cookies (mitigates XSS-based session theft).
  * `Secure`: Ensures cookies are only sent over encrypted HTTPS connections.
  * `SameSite=Lax` / `SameSite=Strict`: Protects against Cross-Site Request Forgery (CSRF).
* **Enforce Proper Authorization:** Protect sensitive administrative functions with server-side role and permission checks, not just the presence of a cookie.
